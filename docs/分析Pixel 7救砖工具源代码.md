# 分析Pixel 7救砖工具源代码

> **Workspace:** /home/meizu/Documents/personal/tensor-usbdl

---

**User**

我希望深入研究该工具的源代码，以理解其如何实现对Google Pixel 7设备的救砖功能。当前我的Pixel 7设备处于异常状态：无法使用fastboot和adb工具进行通信，设备显示为18d1:4f00 Pixel ROM Recovery模式。此前已通过该工具成功将设备从18d1:4f00状态恢复到正常的fastboot模式（18d1:4ee0）。请分析源代码中实现这一状态转换的具体技术原理、关键算法、通信协议以及与硬件交互的核心模块。

**Assistant**

我来深入分析这个工具的源代码。首先让我全面了解项目结构。

现在我已经完整阅读了所有源代码。下面是对该工具实现 Pixel 7 救砖功能的深度技术分析。

---

## Tensor-USBDL 救砖原理深度分析

### 一、整体架构

项目分为两层：

| 层级  | 包             | 文件                                                                                                                                                                                                                                                                                                         | 职责                    |
| --- | ------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------- |
| 库层  | `tensorutils` | [dnw.go](file:///home/meizu/Documents/personal/tensor-usbdl/dnw.go), [command.go](file:///home/meizu/Documents/personal/tensor-usbdl/command.go), [message.go](file:///home/meizu/Documents/personal/tensor-usbdl/message.go), [devices.go](file:///home/meizu/Documents/personal/tensor-usbdl/devices.go) | USB串口通信、DNW协议编解码、消息解析 |
| 应用层 | `main`        | [main.go](file:///home/meizu/Documents/personal/tensor-usbdl/cmd/tensor-usbdl/main.go), [identify.go](file:///home/meizu/Documents/personal/tensor-usbdl/cmd/tensor-usbdl/identify.go), [utils.go](file:///home/meizu/Documents/personal/tensor-usbdl/cmd/tensor-usbdl/utils.go)                           | 引导加载流程控制、设备识别、镜像写入    |

---

### 二、关键状态转换：18d1:4f00 → 18d1:4ee0

**18d1:4f00** 是 Google Pixel 设备进入 **Exynos USB Boot (EUB)** 模式时的 USB VID:PID。当设备 BootROM 无法正常引导时（如引导加载器损坏），SoC 的 iROM（片内 ROM）会回退到此模式，暴露一个串口设备等待主机发送引导加载器。

**18d1:4ee0** 是正常的 **fastboot** 模式。状态转换的核心是：**通过 EUB 协议逐级向设备注入完整的引导加载器链，使设备从裸机状态逐步恢复到可以进入 fastboot 的状态。**

---

### 三、通信协议：Samsung DNW (Downloader) over USB Serial

#### 3.1 设备发现与连接

[dnw.go:14-18](file:///home/meizu/Documents/personal/tensor-usbdl/dnw.go#L14-L18) 定义了目标设备匹配：

```go
devicePairsDNW = [][]string{
    {"18D1", "4F00"}, //Google Pixel 6/6a/6Pro
}
```

连接参数（[dnw.go:38](file:///home/meizu/Documents/personal/tensor-usbdl/dnw.go#L38)）：

- **波特率**: 115200
- **校验位**: 无
- **数据位**: 8
- **停止位**: 1

这是标准的 UART 串口参数，通过 USB 虚拟串口（CDC-ACM）传输。

#### 3.2 DNW 命令格式

[command.go:6-8](file:///home/meizu/Documents/personal/tensor-usbdl/command.go#L6-L8) 定义了 DNW 协议的核心常量：

```go
OpDNW   = []byte("\x1BDNW")   // ESC + "DNW" = 4字节命令头
CmdDNW  = NewCommand(OpDNW, nil, nil, nil)
CmdStop = NewCommand(OpDNW, make([]byte, 4), nil, []byte("\x01\x00"))
```

DNW 命令的二进制结构（[command.go:24-41](file:///home/meizu/Documents/personal/tensor-usbdl/command.go#L24-L41)）：

```
+--------+--------+--------+--------+
| CMD    | ARG    | DATA   | CRC    |
| 4字节  | 4字节  | N字节  | 2字节  |
| \x1BDNW| 长度   | 载荷   | 校验和 |
+--------+--------+--------+--------+
```

- **CMD**: `\x1BDNW`（ESC + "DNW"），是 Samsung Exynos SoC 的标准下载协议标识
- **ARG**: 当未显式指定时，自动计算为 `4 + CmdLen + CRCLen + DataLen`（即整个命令的字节长度）
- **DATA**: 实际要写入的引导加载器镜像数据
- **CRC**: 2字节校验和，算法为所有 DATA 字节的 `uint16` 累加和（[utils.go:36-52](file:///home/meizu/Documents/personal/tensor-usbdl/cmd/tensor-usbdl/utils.go#L36-L52)）

#### 3.3 写入地址

[main.go:87](file:///home/meizu/Documents/personal/tensor-usbdl/cmd/tensor-usbdl/main.go#L87) 默认写入地址为 `OpDNW = \x1BDNW`，即 `0x1B444E57`。这是 Exynos SoC 的 SRAM 下载地址，BootROM 会从此地址执行下载的代码。

---

### 四、EUB (Exynos USB Boot) 消息协议

#### 4.1 消息格式

设备通过串口发送文本消息，以 `\n` 或 `\r` 分隔。[message.go:22-30](file:///home/meizu/Documents/personal/tensor-usbdl/message.go#L22-L30) 解析格式：

```
cmd:sub:dev:arg
```

例如：`eub:req:09845001cddf16d00bd4:DPM`

| 字段  | 含义   | 示例                                                       |
| --- | ---- | -------------------------------------------------------- |
| cmd | 命令类型 | `eub`, `exynos_usb_booting`, `C`, `irom_booting_failure` |
| sub | 子命令  | `req`, `ack`, `nak`                                      |
| dev | 设备标识 | `09845001cddf16d00bd4`（SoC ID + 序列号）                     |
| arg | 参数   | `DPM`, `BL1`, `ABL` 等                                    |

#### 4.2 交互流程

[main.go:338-508](file:///home/meizu/Documents/personal/tensor-usbdl/cmd/tensor-usbdl/main.go#L338-L508) 实现了完整的 EUB 交互状态机：

```
主机                                    设备
 |                                       |
 |-- '\n' (触发设备发送首条消息) -------->|
 |                                       |
 |<-- "exynos_usb_booting:...:SoCID" ----|  设备声明进入EUB模式
 |                                       |
 |<-- "eub:req:SoCID:BL1" --------------|  请求BL1引导加载器
 |                                       |
 |-- "C" (设备发送C表示准备好接收) ------>|  (设备内部流控)
 |                                       |
 |-- [DNW命令: BL1镜像数据] ------------>|  主机发送BL1
 |                                       |
 |<-- "eub:ack:SoCID:BL1" --------------|  设备确认接收BL1
 |                                       |
 |<-- "eub:req:SoCID:DPM" --------------|  请求DPM
 |                                       |
 |-- [DNW命令: DPM镜像数据] ------------>|
 |                                       |
 |<-- "eub:req:SoCID:EPBL" -------------|  请求PBL(作为EPBL)
 |                                       |
 |-- [DNW命令: PBL镜像数据] ------------>|
 |                                       |
 |          ... 逐级请求 ...             |
 |                                       |
 |<-- "eub:req:SoCID:ABL" --------------|  请求Android引导加载器
 |                                       |
 |-- [DNW命令: ABL镜像数据] ------------>|
 |                                       |
 |         (ABL启动后设备重启)           |
 |         进入fastboot模式              |
 |         VID:PID变为18d1:4ee0          |
```

关键机制：

1. **设备主动请求**：每个引导阶段由设备通过 `eub:req` 主动请求，主机被动响应
2. **流控**：设备发送 `C` 字符表示准备好接收数据，主机必须在收到 `C` 后才发送
3. **ACK/NAK**：设备通过 `eub:ack` 确认接收成功，`eub:nak` 表示拒绝

---

### 五、引导加载器链（Boot Chain）

这是实现救砖的核心——逐级恢复 Exynos/Tensor SoC 的完整引导链：

```
iROM (BootROM, 不可修改)
  └─> BL1 (Primary Bootloader)     ← 第1级，由iROM加载
       └─> DPM (Dead Pixel Manager?) ← 12KB数据区
       └─> PBL/EPBL (Primary Bootloader Extended)
            └─> BL2 (Secondary Bootloader)  ← 分header+body发送
                 └─> GSA (Google Secure Agent)
                 └─> ABL (Android Bootloader) ← 分header+body发送
                 │    └─> TZSW (TrustZone Software) ← 分header+body发送
                 │    └─> LDFW (Loadable Firmware) ← 分header+body发送
                 │    └─> BL31 (ARM Trusted Firmware) ← 分header+body发送
                 └─> GCF (Google Calibration Firmware?) ← 可选
                 └─> GSAF (Google Secure Agent Full?) ← 可选
```

#### 5.1 镜像分割策略

[main.go:16-29](file:///home/meizu/Documents/personal/tensor-usbdl/cmd/tensor-usbdl/main.go#L16-L29) 定义了引导加载器镜像的头部结构：

```
偏移    大小    含义
0x000   512     未知字段1
0x200   512     未知字段2（设备型号相关，同系列设备一致）
0x400   4       uint32 magic
0x404   8       未知字段
0x40C   4       uint32: 引导加载器body长度
0x410   4       uint32: "USB Bootable"位标志
0x414   12      未知字段
0x420   32      签名1
0x440   32      签名2
0x460   2976    保留（始终为空）
总计: 4096字节 (4KB) header + body
```

镜像以 4KB 头部为界分为 header 和 body 两部分。设备会分别请求：

| 请求名           | 发送内容                    | op值      |
| ------------- | ----------------------- | -------- |
| `BL1`         | 完整镜像                    | 0 (full) |
| `DPM`         | 完整镜像（或零填充12KB）          | 0        |
| `EPBL`        | PBL完整镜像（Pixel 8仅header） | 0或1      |
| `EPBB`        | PBL body部分              | 2        |
| `BL2`         | BL2 header部分            | 1        |
| `BL2B`        | BL2 body部分              | 2        |
| `ABL`/`ABLB`  | ABL header/body         | 1/2      |
| `TZSW`/`TZSB` | TZSW header/body        | 1/2      |
| `LDFW`/`LDFB` | LDFW header/body        | 1/2      |
| `BL31`/`BL3B` | BL31 header/body        | 1/2      |
| `GCF`/`GCFB`  | GCF header/body         | 1/2      |
| `GSA1`/`GSAF` | 完整镜像                    | 0        |

`--usb` 标志会将 header 偏移 0x410（第1040字节）从 `0x00` 改为 `0x01`，即设置 "USB Bootable" 位，使引导加载器知道这是通过 USB 下载的。

---

### 六、设备识别

[identify.go:35-60](file:///home/meizu/Documents/personal/tensor-usbdl/cmd/tensor-usbdl/identify.go#L35-L60) 通过两种方式识别设备：

1. **USB VID:PID**：`18D1:4F00` → Pixel ROM Recovery (Tensor系列通用)
2. **SoC ID**（来自EUB消息的dev字段）：
   - `9845` → Pixel 6 系列 (Exynos 2100/GS101)
   - `9855` → **Pixel 7 系列** (Tensor G2/GS201) ← 你的设备
   - `9865` → Pixel 8 系列 (Tensor G3/GS301)

---

### 七、错误处理

[main.go:473-499](file:///home/meizu/Documents/personal/tensor-usbdl/cmd/tensor-usbdl/main.go#L473-L499) 处理 `irom_booting_failure` 消息。BootROM 引导失败时会发送一个包含16个条目的调用栈跟踪，工具解析并格式化输出，帮助诊断具体是哪个引导阶段失败。

---

### 八、数据流架构

```
                    ┌─────────────────────┐
                    │   串口硬件 (USB CDC) │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │   serial.Port       │
                    │   (go.bug.st/serial)│
                    └──────────┬──────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
    ┌─────────▼──────┐  ┌─────▼──────┐  ┌──────▼─────────┐
    │  readThread()  │  │  Write()   │  │  WriteCmd()    │
    │  (goroutine)   │  │            │  │  (DNW封装)     │
    │  持续读取→buffer│  │  原始写入  │  │  CMD+ARG+DATA  │
    └─────────┬──────┘  └────────────┘  │  +CRC           │
              │                          └────────────────┘
    ┌─────────▼──────┐
    │ crunchio.Buffer│ ← 异步消息队列
    │ (EUB Writer)   │
    └─────────┬──────┘
              │
    ┌─────────▼──────┐
    │ crunchio.Buffer│ ← 读取克隆
    │ (EUB Reader)   │
    └─────────┬──────┘
              │
    ┌─────────▼──────┐
    │  ReadMsg()     │ ← 逐字节读取直到\n
    │  → NewMessage()│ ← 解析 "cmd:sub:dev:arg"
    └────────────────┘
```

关键设计：

- **读写分离**：`readThread()` 作为独立 goroutine 持续从串口读取数据，写入 `crunchio.Buffer` 异步队列
- **读写互斥**：`sync.Mutex` 保护串口的并发读写
- **流式缓冲**：`crunchio.Buffer` 支持流式写入和随机读取，`SetStream(true)` 避免读取时遇到 EOF
- **分块写入**：每次最多写入 10240 字节（[dnw.go:221](file:///home/meizu/Documents/personal/tensor-usbdl/dnw.go#L221)），避免缓冲区溢出

---

### 九、Pixel 7 救砖的完整流程总结

1. **设备进入EUB模式**：Pixel 7 的 Tensor G2 (Exynos 9855) SoC 的 iROM 检测到引导失败，回退到 USB Boot 模式，枚举为 `18d1:4f00`
2. **工具扫描设备**：通过 USB VID:PID 匹配找到设备，以 115200/8N1 打开串口
3. **触发通信**：发送 `\n` 激活设备发送首条 `exynos_usb_booting` 消息
4. **逐级注入引导链**：设备按 `BL1 → DPM → EPBL → BL2 → GSA → ABL → TZSW → LDFW → BL31` 的顺序请求，主机响应每个 `eub:req` 发送对应的 DNW 命令封装的镜像数据
5. **ABL接管**：ABL（Android Bootloader）加载完成后，初始化 fastboot 服务，设备重启进入 fastboot 模式
6. **状态转换完成**：设备从 `18d1:4f00` (EUB) 变为 `18d1:4ee0` (fastboot)，此时可通过 `fastboot` 命令刷入完整固件

本质上，这个工具利用了 Samsung Exynos SoC 的 **DNW 下载协议** 和 **EUB 引导协议**，在 BootROM 层面与设备通信，逐级重建从 BL1 到 ABL 的完整引导链，最终使设备恢复到可被 fastboot 识别的状态。

---
