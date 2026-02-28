# 6.4 SoC 集成

## 6.4.1 SoC 集成概述

### 什么是 SoC 集成

SoC（System on Chip，片上系统）集成是指将多个功能模块（如处理器核心、存储器、外设控制器等）集成到单个芯片上的过程。在这个过程中，CoralNPU 作为一个 IP 核（Intellectual Property Core，知识产权核心）被集成到更大的系统中。

**基本概念解释：**

- **IP 核**：可以理解为一个预先设计好的、可重用的硬件模块，就像软件中的库（library）一样。CoralNPU 就是这样一个 IP 核，可以被其他芯片设计者集成到他们的 SoC 中。

- **总线接口**：不同的硬件模块之间需要通过总线（bus）进行通信。总线就像是硬件模块之间的"高速公路"，数据通过总线在不同模块之间传输。

- **主从关系**：
  - **主设备（Master）**：发起数据传输请求的设备，比如 CPU 或 DMA 控制器
  - **从设备（Slave）**：响应主设备请求的设备，比如存储器或外设

### CoralNPU 的集成方式

CoralNPU 支持两种主流的总线接口标准：

1. **AXI4 接口**：ARM 公司定义的高性能总线标准，广泛应用于各种 SoC 设计中
2. **TileLink 接口**：开源的总线标准，常用于 RISC-V 生态系统

CoralNPU 在 SoC 中既可以作为主设备（通过主接口访问外部存储器和外设），也可以作为从设备（允许外部主设备访问其内部的 TCM 和 CSR）。

```
典型的 SoC 集成架构：

    +------------------+
    |   主处理器 CPU   |
    +--------+---------+
             |
    +--------v---------+
    |   系统总线       |
    |   (AXI/TileLink) |
    +--+----+----+-----+
       |    |    |
   +---v+ +-v--+ +v-------+
   |DDR | |外设| |CoralNPU|
   |存储| |控制| | IP 核  |
   +----+ +----+ +--------+
```

## 6.4.2 顶层设计

CoralNPU 提供了两个顶层模块，分别对应不同的总线接口：

### CoreAxi - AXI4 接口顶层模块

`CoreAxi` 是使用 AXI4 总线接口的顶层模块，定义在 `hdl/chisel/src/coralnpu/CoreAxi.scala` 文件中。

**模块结构：**

```
CoreAxi 模块内部结构：

  +--------------------------------------------------+
  |                    CoreAxi                       |
  |                                                  |
  |  +------------+      +--------+                  |
  |  | 复位同步器  |      | CoreCSR|                  |
  |  | (RstSync)  |      | (控制)  |                  |
  |  +-----+------+      +----+---+                  |
  |        |                  |                      |
  |  +-----v------+      +----v---+                  |
  |  | 时钟门控    |      |  Core  |                  |
  |  |(ClockGate) |----->| (核心) |                  |
  |  +------------+      +----+---+                  |
  |                          |                       |
  |  +--------+  +--------+  |  +--------+           |
  |  | ITCM   |  | DTCM   |  |  | DBus2  |           |
  |  | (指令) |  | (数据) |  |  | Axi    |           |
  |  +---+----+  +---+----+  |  +---+----+           |
  |      |           |       |      |                |
  |  +---v-----------v-------v------v---+            |
  |  |      Fabric Arbiter/Mux          |            |
  |  +---+---------------------------+--+            |
  |      |                           |               |
  +------+---------------------------+---------------+
         |                           |
    axi_slave                   axi_master
    (从接口)                    (主接口)
```

**端口定义：**

| 端口名称 | 方向 | 位宽 | 说明 |
|---------|------|------|------|
| `aclk` | Input | 1 | AXI 总线时钟 |
| `aresetn` | Input | 1 | AXI 总线复位（低电平有效） |
| `axi_slave` | Slave | - | AXI4 从接口，用于访问 ITCM、DTCM 和 CSR |
| `axi_master` | Master | - | AXI4 主接口，用于访问外部存储器和外设 |
| `halted` | Output | 1 | 核心停止信号（高电平表示核心已停止） |
| `fault` | Output | 1 | 核心故障信号（高电平表示发生故障） |
| `wfi` | Output | 1 | 等待中断信号（Wait For Interrupt） |
| `irq` | Input | 1 | 中断请求输入 |
| `debug` | Output | - | 调试接口（仅用于仿真） |
| `slog` | Output | - | 字符串日志接口（仅用于仿真） |
| `te` | Input | 1 | 测试使能信号 |

**信号说明：**

- **时钟和复位**：
  - `aclk`：所有 AXI 接口的同步时钟
  - `aresetn`：异步复位信号，低电平有效（0 表示复位，1 表示正常工作）
  - `te`：测试使能，用于 DFT（Design For Test，可测试性设计）

- **AXI 从接口（axi_slave）**：
  - 允许外部主设备（如主 CPU）访问 CoralNPU 的内部资源
  - 地址映射：
    - `0x0000 - 0x1FFF`：ITCM（指令紧耦合存储器，8KB）
    - `0x10000 - 0x17FFF`：DTCM（数据紧耦合存储器，32KB）
    - `0x30000 - ...`：CSR（控制状态寄存器）

- **AXI 主接口（axi_master）**：
  - CoralNPU 通过此接口访问外部存储器和外设
  - 支持突发传输（burst transfer）
  - 数据宽度：32 位或 128 位（取决于配置）

- **状态和控制信号**：
  - `halted`：当 CoralNPU 执行 `mpause` 指令或进入调试模式时置高
  - `fault`：当发生异常（如非法指令、访问错误）时置高
  - `wfi`：当核心执行 WFI 指令等待中断时置高，此时核心会被时钟门控以节省功耗
  - `irq`：外部中断输入，可以唤醒处于 WFI 状态的核心

### CoreTlul - TileLink 接口顶层模块

`CoreTlul` 是使用 TileLink-UL（Uncached Lightweight）总线接口的顶层模块，定义在 `hdl/chisel/src/coralnpu/CoreTlul.scala` 文件中。

**模块结构：**

```
CoreTlul 模块内部结构：

  +--------------------------------------------------+
  |                   CoreTlul                       |
  |                                                  |
  |  +------------+                                  |
  |  |  CoreAxi   |  (内部实例化 AXI 版本)            |
  |  +-----+------+                                  |
  |        |                                         |
  |  +-----v------+      +------------+              |
  |  | Axi2TLUL   |      | TLUL2Axi   |              |
  |  | (主桥接)   |      | (从桥接)   |              |
  |  +-----+------+      +-----+------+              |
  |        |                   |                     |
  |  +-----v------+      +-----v------+              |
  |  | Request    |      | Response   |              |
  |  | IntegrityGen|     | IntegrityGen|             |
  |  +-----+------+      +-----+------+              |
  |        |                   |                     |
  +--------+-------------------+---------------------+
           |                   |
      tl_host              tl_device
      (主接口)             (从接口)
```

**端口定义：**

| 端口名称 | 方向 | 位宽 | 说明 |
|---------|------|------|------|
| `clk` | Input | 1 | TileLink 总线时钟 |
| `rst_ni` | Input | 1 | TileLink 总线复位（低电平有效） |
| `tl_host` | Host | - | TileLink 主接口（CoralNPU 作为主设备） |
| `tl_device` | Device | - | TileLink 从接口（CoralNPU 作为从设备） |
| `halted` | Output | 1 | 核心停止信号 |
| `fault` | Output | 1 | 核心故障信号 |
| `wfi` | Output | 1 | 等待中断信号 |
| `irq` | Input | 1 | 中断请求输入 |
| `te` | Input | 1 | 测试使能信号 |
| `dm` | Optional | - | 调试模块接口（可选） |

**TileLink 接口说明：**

TileLink 是一种基于信用（credit-based）的总线协议，使用两个通道进行通信：

- **A 通道（请求通道）**：
  - 主设备发送读写请求
  - 包含地址、操作类型、数据（写操作）等信息

- **D 通道（响应通道）**：
  - 从设备返回响应
  - 包含读数据、响应状态等信息

**完整性检查（Integrity Check）：**

`CoreTlul` 模块包含了完整性生成器（IntegrityGen），用于：
- 为发送的请求生成完整性校验码
- 验证接收的响应的完整性
- 这是 OpenTitan 项目的安全特性，用于检测总线传输错误

### AXI4 信号详解

对于不熟悉 AXI 协议的读者，这里详细解释 AXI4 接口的信号：

**AXI4 主接口信号（axi_master）：**

*写地址通道（AW - Address Write）：*

| 信号名 | 方向 | 位宽 | 说明 |
|--------|------|------|------|
| `awaddr` | Output | 32 | 写地址 |
| `awvalid` | Output | 1 | 写地址有效信号 |
| `awready` | Input | 1 | 写地址准备好信号 |
| `awid` | Output | 可配置 | 写事务 ID |
| `awlen` | Output | 8 | 突发长度（传输次数 - 1） |
| `awsize` | Output | 3 | 每次传输的字节数（2^awsize） |
| `awburst` | Output | 2 | 突发类型（0=FIXED, 1=INCR, 2=WRAP） |

**解释：**
- `awaddr`：CoralNPU 想要写入的地址
- `awvalid`：当 CoralNPU 准备好发送写地址时置高
- `awready`：当从设备准备好接收写地址时置高
- `awlen`：表示这次突发传输包含多少次数据传输，例如 awlen=3 表示传输 4 次数据
- `awsize`：每次传输的数据大小，例如 awsize=2 表示每次传输 4 字节（2^2=4）

*写数据通道（W - Write Data）：*

| 信号名 | 方向 | 位宽 | 说明 |
|--------|------|------|------|
| `wdata` | Output | 32/128 | 写数据 |
| `wvalid` | Output | 1 | 写数据有效信号 |
| `wready` | Input | 1 | 写数据准备好信号 |
| `wstrb` | Output | 4/16 | 字节选通信号（每位对应一个字节） |
| `wlast` | Output | 1 | 最后一次传输标志 |

**解释：**
- `wdata`：要写入的数据
- `wstrb`：字节选通，每一位对应 wdata 中的一个字节，1 表示该字节有效
  - 例如：wstrb=0b0011 表示只写入最低 2 个字节
- `wlast`：在突发传输的最后一次数据传输时置高

*写响应通道（B - Write Response）：*

| 信号名 | 方向 | 位宽 | 说明 |
|--------|------|------|------|
| `bresp` | Input | 2 | 写响应（0=OKAY, 2=SLVERR） |
| `bvalid` | Input | 1 | 写响应有效信号 |
| `bready` | Output | 1 | 写响应准备好信号 |
| `bid` | Input | 可配置 | 写响应 ID |

*读地址通道（AR - Address Read）：*

| 信号名 | 方向 | 位宽 | 说明 |
|--------|------|------|------|
| `araddr` | Output | 32 | 读地址 |
| `arvalid` | Output | 1 | 读地址有效信号 |
| `arready` | Input | 1 | 读地址准备好信号 |
| `arid` | Output | 可配置 | 读事务 ID |
| `arlen` | Output | 8 | 突发长度 |
| `arsize` | Output | 3 | 每次传输的字节数 |
| `arburst` | Output | 2 | 突发类型 |

*读数据通道（R - Read Data）：*

| 信号名 | 方向 | 位宽 | 说明 |
|--------|------|------|------|
| `rdata` | Input | 32/128 | 读数据 |
| `rvalid` | Input | 1 | 读数据有效信号 |
| `rready` | Output | 1 | 读数据准备好信号 |
| `rresp` | Input | 2 | 读响应 |
| `rlast` | Input | 1 | 最后一次传输标志 |
| `rid` | Input | 可配置 | 读响应 ID |

**AXI 握手机制：**

AXI 协议使用 valid/ready 握手机制：
- 主设备通过 `valid` 信号表示数据有效
- 从设备通过 `ready` 信号表示准备接收
- 只有当 `valid` 和 `ready` 同时为高时，数据传输才会发生

```
时序示例：

Clock:    __|‾‾|__|‾‾|__|‾‾|__|‾‾|__
valid:    ______|‾‾‾‾‾‾‾‾‾‾‾‾|______
ready:    __________|‾‾‾‾‾‾‾‾|______
传输:              ↑ 这里发生传输
```

## 6.4.3 时钟和复位

### 时钟域

CoralNPU 支持单时钟域和多时钟域设计：

**单时钟域设计（推荐用于简单系统）：**

```
所有模块使用同一个时钟：

    系统时钟 ──┬─→ CoralNPU
               ├─→ 存储器
               ├─→ 外设
               └─→ 总线
```

**多时钟域设计（用于复杂系统）：**

```
不同模块使用不同时钟：

    核心时钟 ────→ CoralNPU 核心
    总线时钟 ────→ AXI/TileLink 总线
    DDR 时钟 ────→ DDR 控制器
    外设时钟 ────→ 外设控制器
```

在 `CoralNPUChiselSubsystem` 中，支持异步时钟域：

```scala
// 主时钟域
val clk_i = Input(Clock())
val rst_ni = Input(AsyncReset())

// 异步设备时钟域（如 DDR）
val async_ports_devices = new Bundle {
  val clocks = Input(Vec(asyncDeviceDomains.length, Clock()))
  val resets = Input(Vec(asyncDeviceDomains.length, AsyncReset()))
}
```

**时钟域交叉（Clock Domain Crossing, CDC）：**

当信号需要从一个时钟域传递到另一个时钟域时，需要使用特殊的同步电路：

```
时钟域 A                时钟域 B
  数据 ──→ [异步 FIFO] ──→ 数据
  clk_a                  clk_b
```

CoralNPU 使用 `TlulFifoAsync` 模块处理 TileLink 总线的时钟域交叉：

```scala
val fifo = Module(new TlulFifoAsync(params))
fifo.io.clk_h_i := clock_domain_a  // 主时钟域
fifo.io.clk_d_i := clock_domain_b  // 从时钟域
```

### 复位策略

CoralNPU 使用异步复位、同步释放（Asynchronous Assert, Synchronous Deassert）策略：

**基本概念：**
- **异步复位**：复位信号可以在时钟的任何时刻生效
- **同步释放**：复位信号的释放与时钟边沿同步

**为什么使用这种策略？**
1. 异步复位可以立即将系统置于已知状态
2. 同步释放避免了亚稳态（metastability）问题

**复位同步器（Reset Synchronizer）：**

```
异步复位输入 ──→ [同步器] ──→ 同步复位输出
                   ↑
                 时钟
```

在 `CoreAxi` 中的实现：

```scala
val rst_sync = Module(new RstSync())
rst_sync.io.clk_i := io.aclk
rst_sync.io.rstn_i := io.aresetn  // 异步复位输入
// rst_sync.io.rstn_o 是同步后的复位输出
```

**复位时序：**

```
时序图：

Clock:     __|‾‾|__|‾‾|__|‾‾|__|‾‾|__|‾‾|__
aresetn:   __|‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾
                ↑ 异步复位释放
rstn_o:    ______|‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾
                    ↑ 同步复位释放（与时钟边沿对齐）
```

**复位顺序：**

1. 上电时，所有模块处于复位状态
2. 系统时钟稳定后，释放复位
3. 等待至少 2 个时钟周期，确保复位同步器工作
4. 系统开始正常工作

**代码示例（C 语言）：**

```c
// 1. 保持复位状态
*reset_reg = 0;  // 低电平有效复位

// 2. 等待时钟稳定
delay_us(10);

// 3. 释放复位
*reset_reg = 1;

// 4. 等待复位同步
delay_us(1);

// 5. 系统可以开始工作
```

### 时钟门控（Clock Gating）

时钟门控是一种节省功耗的技术，当模块不工作时关闭其时钟。

**基本原理：**

```
使能信号 ──┐
           AND ──→ 门控时钟
时钟输入 ──┘
```

**注意：** 简单的 AND 门会产生毛刺（glitch），实际电路使用锁存器式时钟门控：

```
正确的时钟门控电路：

enable ──→ [Latch] ──┐
            ↑        AND ──→ gated_clock
          clock ─────┘
```

在 CoralNPU 中的应用：

```scala
val cg = Module(new ClockGate)
cg.io.clk_i := clock
cg.io.enable := !core.io.wfi  // 当核心不在 WFI 状态时使能时钟
val gated_clock = cg.io.clk_o
```

**时钟门控的好处：**
- 降低动态功耗（时钟树的翻转功耗）
- 当 CoralNPU 执行 WFI（Wait For Interrupt）指令时，自动关闭时钟
- 可以节省 20-30% 的功耗

**使用时钟门控的注意事项：**
1. 门控时钟的使能信号必须稳定
2. 在释放门控前，确保所有输入信号稳定
3. 某些关键路径（如复位逻辑）不应该被门控

## 6.4.4 电源管理

### 电源域

现代 SoC 通常包含多个电源域，以实现更精细的功耗控制：

```
SoC 电源域划分示例：

+------------------+
| Always-On Domain |  ← 始终供电（RTC、唤醒逻辑）
|   (0.9V)         |
+------------------+
        |
+------------------+
|  Core Domain     |  ← CoralNPU 核心（可关闭）
|   (0.9V)         |
+------------------+
        |
+------------------+
| Peripheral Domain|  ← 外设（可独立控制）
|   (1.8V/3.3V)    |
+------------------+
```

**电源域隔离（Power Domain Isolation）：**

当一个电源域被关闭时，需要隔离其输出信号，防止漏电流：

```
电源域 A (ON)          电源域 B (OFF)
  信号 ──→ [隔离单元] ──→ 固定值（0 或 1）
```

### 低功耗模式

CoralNPU 支持多种低功耗模式：

**1. WFI 模式（Wait For Interrupt）：**
- 核心执行 WFI 指令后进入等待状态
- 时钟被门控，但寄存器状态保持
- 中断可以唤醒核心
- 功耗降低约 70%

**2. 时钟门控模式：**
- 通过 CSR 寄存器手动关闭时钟
- 适用于长时间空闲的场景
- 功耗降低约 80%

**3. 电源关闭模式（需要外部支持）：**
- 完全关闭 CoralNPU 的电源
- 需要重新初始化
- 功耗降低约 99%

**低功耗模式转换：**

```
状态转换图：

    [正常运行]
        ↓ WFI 指令
    [WFI 模式]
        ↓ 中断
    [正常运行]
        ↓ 写 CSR
    [时钟门控]
        ↓ 写 CSR
    [正常运行]
        ↓ 电源控制
    [电源关闭]
        ↓ 上电复位
    [正常运行]
```

### 电源开关

电源开关用于控制电源域的开关：

```
电源开关电路：

VDD ──┬─→ [开关] ──→ VDDCORE
      │      ↑
      │   控制信号
      └─→ [隔离单元]
```

**电源开关时序：**

```
1. 关闭电源：
   a. 保存状态（如果需要）
   b. 使能隔离单元
   c. 关闭时钟
   d. 关闭电源开关

2. 打开电源：
   a. 打开电源开关
   b. 等待电源稳定
   c. 释放复位
   d. 禁用隔离单元
   e. 使能时钟
   f. 恢复状态（如果需要）
```

**注意事项：**
- 电源开关的控制通常由外部电源管理单元（PMU）完成
- CoralNPU 本身不包含电源开关逻辑
- 集成时需要根据具体的 PMU 设计连接控制信号

## 6.4.5 调试接口

### JTAG 接口

JTAG（Joint Test Action Group）是一种标准的调试接口，用于：
- 调试程序执行
- 读写寄存器和存储器
- 下载程序到 TCM
- 单步执行

**JTAG 信号：**

| 信号名 | 方向 | 说明 |
|--------|------|------|
| `TCK` | Input | 测试时钟 |
| `TMS` | Input | 测试模式选择 |
| `TDI` | Input | 测试数据输入 |
| `TDO` | Output | 测试数据输出 |
| `TRST` | Input | 测试复位（可选） |

**JTAG 连接示意图：**

```
调试器                    CoralNPU
(OpenOCD/GDB)
    |
    | USB/以太网
    |
[JTAG 适配器]
    |
    | TCK, TMS, TDI, TDO
    |
[JTAG TAP] ──→ [Debug Module] ──→ [Core]
```

### Debug Module

CoralNPU 可选地包含一个调试模块（Debug Module），符合 RISC-V Debug Specification：

**调试模块功能：**
1. 暂停和恢复核心执行
2. 单步执行指令
3. 读写通用寄存器
4. 读写 CSR 寄存器
5. 读写存储器
6. 设置硬件断点

**调试模块接口：**

在 `CoreTlul` 中：

```scala
val dm = Option.when(p.useDebugModule)(new DebugModuleIO(p))
```

**调试模块信号：**

| 信号名 | 方向 | 说明 |
|--------|------|------|
| `haltreq` | Input | 暂停请求 |
| `resumereq` | Input | 恢复请求 |
| `halted` | Output | 核心已暂停 |
| `running` | Output | 核心正在运行 |

### 如何连接调试器

**硬件连接：**

1. 将 JTAG 适配器连接到 CoralNPU 的 JTAG 引脚
2. 确保 JTAG 信号的电平匹配（通常是 3.3V 或 1.8V）
3. 连接地线（GND）

**软件配置（使用 OpenOCD）：**

创建 OpenOCD 配置文件 `coralnpu.cfg`：

```tcl
# JTAG 适配器配置
adapter driver ftdi
ftdi_vid_pid 0x0403 0x6014

# JTAG 速度
adapter speed 1000

# 目标配置
jtag newtap coralnpu cpu -irlen 5 -expected-id 0x10002001

target create coralnpu.cpu riscv -chain-position coralnpu.cpu

# 初始化
init
halt
```

**使用 GDB 调试：**

```bash
# 启动 OpenOCD
openocd -f coralnpu.cfg

# 在另一个终端启动 GDB
riscv32-unknown-elf-gdb program.elf

# 在 GDB 中连接到 OpenOCD
(gdb) target remote localhost:3333

# 加载程序
(gdb) load

# 设置断点
(gdb) break main

# 运行
(gdb) continue
```

**常用 GDB 命令：**

```
info registers     # 查看寄存器
x/10x 0x10000      # 查看存储器内容
step               # 单步执行
continue           # 继续执行
backtrace          # 查看调用栈
```

## 6.4.6 集成检查清单

在将 CoralNPU 集成到 SoC 中时，请检查以下项目：

### 接口连接检查

- [ ] 时钟信号已正确连接
- [ ] 复位信号已正确连接（注意极性：低电平有效）
- [ ] AXI/TileLink 主接口已连接到系统总线
- [ ] AXI/TileLink 从接口已连接到系统总线
- [ ] 中断信号已连接到中断控制器
- [ ] 调试接口已连接（如果使用）
- [ ] 所有信号的位宽匹配

### 时钟复位检查

- [ ] 时钟频率满足时序要求（检查综合报告）
- [ ] 复位信号有足够的保持时间（至少 2 个时钟周期）
- [ ] 时钟域交叉使用了适当的同步电路
- [ ] 时钟门控逻辑正确实现
- [ ] 复位顺序正确（先时钟稳定，再释放复位）

### 地址映射检查

- [ ] CoralNPU 的地址范围不与其他模块冲突
- [ ] ITCM 地址范围：0x0000 - 0x1FFF（8KB）
- [ ] DTCM 地址范围：0x10000 - 0x17FFF（32KB）
- [ ] CSR 地址范围：0x30000 - ...
- [ ] 外部存储器地址范围已正确配置
- [ ] 地址译码逻辑正确

**地址映射示例：**

```
系统地址空间：

0x00000000 - 0x0FFFFFFF: DDR 存储器 (256MB)
0x10000000 - 0x1FFFFFFF: 外设
0x70000000 - 0x7003FFFF: CoralNPU (256KB)
  0x70000000 - 0x70001FFF:   ITCM (8KB)
  0x70010000 - 0x70017FFF:   DTCM (32KB)
  0x70030000 - 0x700300FF:   CSR
```

### 验证要点

**1. 复位验证：**
- [ ] 上电复位后，CoralNPU 处于已知状态
- [ ] 软件复位功能正常
- [ ] 复位期间没有毛刺或亚稳态

**2. 时钟验证：**
- [ ] 时钟信号质量良好（无抖动、无毛刺）
- [ ] 时钟门控功能正常
- [ ] 时钟域交叉无数据丢失

**3. 总线验证：**
- [ ] AXI/TileLink 握手正确
- [ ] 突发传输正常
- [ ] 错误响应正确处理
- [ ] 总线仲裁公平

**4. 功能验证：**
- [ ] 可以通过从接口访问 ITCM
- [ ] 可以通过从接口访问 DTCM
- [ ] 可以通过从接口访问 CSR
- [ ] CoralNPU 可以通过主接口访问外部存储器
- [ ] 中断功能正常
- [ ] WFI 指令正常工作

**5. 性能验证：**
- [ ] 总线带宽满足需求
- [ ] 延迟在可接受范围内
- [ ] 没有死锁或活锁

**6. 功耗验证：**
- [ ] 时钟门控有效降低功耗
- [ ] WFI 模式功耗符合预期
- [ ] 电源域隔离正常

**验证方法：**

1. **仿真验证：**
   - 使用 Verilator 或 VCS 进行 RTL 仿真
   - 运行测试程序，检查功能正确性
   - 使用波形查看器检查信号时序

2. **形式验证：**
   - 使用形式验证工具检查协议一致性
   - 验证死锁自由性

3. **FPGA 验证：**
   - 在 FPGA 上实现完整系统
   - 运行实际应用程序
   - 测量性能和功耗

4. **硅后验证：**
   - 在实际芯片上测试
   - 验证时序裕量
   - 测试极限工作条件

**常见问题和解决方法：**

| 问题 | 可能原因 | 解决方法 |
|------|----------|----------|
| 复位后无响应 | 复位时序不正确 | 检查复位同步器，延长复位保持时间 |
| 总线传输失败 | 地址译码错误 | 检查地址映射，确认地址范围 |
| 时钟域交叉错误 | CDC 电路缺失 | 添加异步 FIFO 或同步器 |
| 功耗过高 | 时钟门控未生效 | 检查时钟门控使能信号 |
| 调试器无法连接 | JTAG 信号错误 | 检查 JTAG 引脚连接和电平 |

---

## 总结

本章介绍了 CoralNPU 的 SoC 集成方法，包括：

1. **顶层设计**：CoreAxi 和 CoreTlul 两种接口选择
2. **时钟和复位**：时钟域、复位策略、时钟门控
3. **电源管理**：电源域、低功耗模式、电源开关
4. **调试接口**：JTAG、Debug Module、调试器连接
5. **集成检查清单**：接口、时钟、地址、验证要点

通过遵循这些指南，可以将 CoralNPU 成功集成到各种 SoC 设计中。

**下一步：**
- 阅读第七章"软件开发"，了解如何为 CoralNPU 编写程序
- 参考 `doc/integration_guide.md` 获取更多集成细节
- 查看 `hdl/chisel/src/soc/` 目录中的示例设计
