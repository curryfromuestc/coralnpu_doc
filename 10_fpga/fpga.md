# 第十章：FPGA 实现

## 10.1 FPGA 概述

### 10.1.1 什么是 FPGA

FPGA（Field-Programmable Gate Array，现场可编程门阵列）是一种可以通过编程来配置硬件逻辑的集成电路芯片。与传统的固定功能芯片不同，FPGA 内部包含大量可编程的逻辑单元、存储单元和互连资源，用户可以通过硬件描述语言（如 Verilog、VHDL）来定义这些资源的功能和连接方式。

**FPGA 的主要组成部分：**

1. **可编程逻辑块（Logic Blocks）**：实现基本的逻辑运算，如与、或、非等
2. **可编程互连（Interconnects）**：连接各个逻辑块
3. **I/O 块（I/O Blocks）**：与外部引脚连接，实现输入输出
4. **存储资源（Memory）**：片上 RAM，用于存储数据
5. **DSP 块**：专用的数字信号处理单元
6. **时钟管理**：PLL（锁相环）等时钟生成和分配电路

**基本概念解释：**

- **综合（Synthesis）**：将 Verilog/VHDL 代码转换为门级网表的过程，类似于软件编译
- **布局布线（Place and Route）**：将逻辑门映射到 FPGA 的物理位置，并连接它们
- **比特流（Bitstream）**：FPGA 的配置文件，类似于软件的可执行文件
- **时序收敛（Timing Closure）**：确保所有信号能在时钟周期内完成传输

### 10.1.2 为什么要在 FPGA 上实现

CoralNPU 项目在 FPGA 上实现有以下几个重要原因：

1. **硬件验证**：在流片（制造真正的芯片）之前，可以在 FPGA 上验证设计的正确性
2. **快速原型**：FPGA 可以快速实现和修改设计，加速开发周期
3. **真实环境测试**：可以在真实硬件上运行软件，测试性能和功能
4. **成本效益**：相比流片，FPGA 开发成本低得多
5. **灵活性**：可以随时修改设计并重新配置 FPGA

## 10.2 支持的开发板

### 10.2.1 Nexus 开发板

CoralNPU 项目目前支持基于 Xilinx UltraScale+ 系列 FPGA 的 Nexus 开发板。

**硬件规格：**

- **FPGA 型号**：Xilinx VU13P (xcvu13p-fhga2104-2-e)
- **封装**：FHGA2104
- **速度等级**：-2
- **系统时钟**：80 MHz（主时钟）
- **DDR4 内存**：支持 DDR4 SDRAM 接口

**主要外设接口：**

1. **UART**：2 个 UART 接口（UART0 和 UART1）
   - UART1 用于调试输出
   - 波特率：115200 bps

2. **SPI**：
   - SPI Slave 接口：用于加载程序
   - SPI Master 接口：用于连接外部 SPI 设备

3. **GPIO**：8 个通用 I/O 引脚

4. **JTAG**：用于调试和编程

5. **LED 指示灯**：
   - `io_halted`：处理器停止指示
   - `io_fault`：故障指示
   - `ddr_cal_complete_o`：DDR 校准完成指示
   - 其他 DDR 状态指示灯

### 10.2.2 硬件要求

**开发主机要求：**

- 操作系统：Linux（推荐 Ubuntu 20.04 或更高版本）
- 内存：至少 16 GB RAM（推荐 32 GB）
- 硬盘空间：至少 100 GB 可用空间
- Xilinx Vivado Design Suite（2021.1 或更高版本）

**连接要求：**

- USB 转 UART 线缆（用于串口通信）
- JTAG 调试器（如 Xilinx Platform Cable）
- 电源适配器

## 10.3 综合和布局布线

### 10.3.1 构建系统

CoralNPU 使用 **FuseSoC** 作为构建系统，它是一个用于管理 HDL 项目的工具。FuseSoC 可以自动调用 Vivado 进行综合和实现。

**基本概念：**

- **Core 文件**：`.core` 文件定义了一个 IP 核或模块，包含源文件列表、依赖关系、参数等
- **Target**：构建目标，如 `sim`（仿真）或 `synth`（综合）
- **Filesets**：文件集合，如 RTL 文件、约束文件、TCL 脚本等

### 10.3.2 使用 Vivado 综合

**步骤 1：准备环境**

首先确保已安装 Vivado 并设置好环境变量：

```bash
source /path/to/Vivado/2021.1/settings64.sh
```

**步骤 2：构建比特流**

使用 Bazel 构建系统生成 FPGA 比特流：

```bash
# 构建默认配置（标准内存）
bazel build //fpga:build_chip_nexus_bitstream

# 构建高内存配置
bazel build //fpga:build_chip_nexus_bitstream_highmem
```

**构建过程说明：**

1. **读取设计文件**：FuseSoC 读取 `.core` 文件，收集所有 RTL 源文件
2. **综合（Synthesis）**：
   - Vivado 将 SystemVerilog 代码转换为门级网表
   - 优化逻辑，去除冗余电路
   - 映射到 FPGA 的基本单元（LUT、FF、BRAM 等）
3. **布局（Placement）**：
   - 将逻辑单元分配到 FPGA 的物理位置
   - 考虑时序和布线资源
4. **布线（Routing）**：
   - 连接各个逻辑单元
   - 使用 FPGA 的可编程互连资源
5. **时序分析**：
   - 检查所有路径是否满足时序要求
   - 报告时序违例（如果有）

### 10.3.3 时序收敛

**什么是时序收敛？**

时序收敛是指设计中所有信号路径都能在规定的时钟周期内完成传输。如果信号传输时间超过时钟周期，就会出现时序违例，导致电路工作不正常。

**时序约束文件：**

CoralNPU 的时序约束定义在 `pins_nexus.xdc` 文件中：

```tcl
# 主时钟：100 MHz 输入，周期 10ns
create_clock -period 10.00 -name sys_clk_pin [get_ports clk_p_i]

# 生成时钟：80 MHz 主时钟
create_generated_clock -name clk_main [get_pin i_clkgen/i_clkgen/pll/CLKOUT0]

# 异步时钟组（这些时钟之间不需要时序约束）
set_clock_groups -asynchronous \
  -group [get_clocks sys_clk_pin] \
  -group [get_clocks c0_sys_clk_p] \
  -group [get_clocks spi_clk_i] \
  -group [get_clocks jtag_tck_i]
```

**基本概念解释：**

- **时钟周期（Clock Period）**：时钟的一个完整周期，单位是纳秒（ns）
  - 例如：80 MHz 时钟的周期 = 1000 / 80 = 12.5 ns
- **建立时间（Setup Time）**：数据在时钟沿到来之前必须稳定的时间
- **保持时间（Hold Time）**：数据在时钟沿到来之后必须保持稳定的时间
- **异步时钟域**：不同频率或不相关的时钟，它们之间需要特殊处理（CDC - Clock Domain Crossing）

**时序优化技巧：**

1. **降低时钟频率**：如果时序不满足，可以降低目标频率
2. **添加流水线**：在长路径上插入寄存器，分割组合逻辑
3. **优化约束**：正确设置时钟约束和 I/O 约束
4. **使用更快的速度等级**：选择更快的 FPGA 器件

### 10.3.4 生成比特流

综合和布局布线完成后，Vivado 会生成比特流文件：

```
bazel-bin/fpga/com.google.coralnpu_fpga_chip_nexus_0.1/synth-vivado/chip_nexus.bit
```

**比特流文件说明：**

- `.bit` 文件：标准比特流，用于配置 FPGA
- `.mmi` 文件：内存映射信息，用于更新 BRAM 内容
- `.ltx` 文件：逻辑分析仪探针文件（如果使用 ILA）

**加载比特流到 FPGA：**

可以使用 Vivado Hardware Manager 或命令行工具加载：

```bash
# 使用 Vivado 命令行
vivado -mode batch -source load_bitstream.tcl
```

或者使用 JTAG 编程器直接加载。


## 10.4 IP 核

### 10.4.1 什么是 IP 核

IP 核（Intellectual Property Core）是预先设计好的、可重用的硬件模块。就像软件开发中的库函数一样，IP 核可以直接集成到设计中，避免重复开发。

**IP 核的类型：**

1. **软核（Soft IP）**：以 RTL 代码形式提供，可以综合到任何 FPGA
2. **硬核（Hard IP）**：已经映射到特定 FPGA 的物理资源，性能更好但不可移植

### 10.4.2 内存控制器

CoralNPU 使用两种类型的内存：

#### SRAM IP 核

**位置**：`fpga/ip/sram/`

SRAM 用于片上快速存储，通常用于：
- 指令存储器（IMEM）
- 数据存储器（DMEM）
- 缓存

**特点：**
- 单周期访问
- 容量较小（受 FPGA BRAM 资源限制）
- 不需要刷新

**使用示例：**

SRAM 模块在 `coralnpu_soc.sv` 中实例化，通过 TileLink-UL 总线接口访问。

#### DDR4 内存控制器

**位置**：`fpga/ip/ddr4_stub/`（存根）或内部 DDR4 IP

DDR4 用于大容量外部存储：
- 主内存
- 程序加载区域

**特点：**
- 容量大（通常 GB 级别）
- 访问延迟较高（数十个时钟周期）
- 需要初始化和校准

**DDR4 接口信号：**

```systemverilog
// AXI4 写地址通道
output io_ddr_mem_axi_aw_valid,
input  io_ddr_mem_axi_aw_ready,
output [31:0] io_ddr_mem_axi_aw_bits_addr,

// AXI4 写数据通道
output io_ddr_mem_axi_w_valid,
input  io_ddr_mem_axi_w_ready,
output [255:0] io_ddr_mem_axi_w_bits_data,

// AXI4 读地址通道
output io_ddr_mem_axi_ar_valid,
input  io_ddr_mem_axi_ar_ready,

// AXI4 读数据通道
input  io_ddr_mem_axi_r_valid,
output io_ddr_mem_axi_r_ready,
input  [255:0] io_ddr_mem_axi_r_bits_data,
```

**基本概念解释：**

- **AXI4 协议**：一种标准的片上总线协议，使用握手信号（valid/ready）进行数据传输
- **突发传输（Burst）**：一次传输多个数据，提高带宽利用率
- **校准（Calibration）**：DDR4 启动时需要调整时序参数，确保可靠通信

**DDR4 初始化流程：**

1. 复位后，DDR4 控制器开始校准过程
2. 校准完成后，`ddr_cal_complete_o` 信号变为高电平
3. 此时可以开始访问 DDR4 内存

### 10.4.3 外设 IP

#### UART IP 核

UART（Universal Asynchronous Receiver/Transmitter，通用异步收发器）用于串口通信。

**寄存器地址：**

```c
#define UART1_BASE 0x40010000
#define UART_CTRL_OFFSET 0x10      // 控制寄存器
#define UART_STATUS_OFFSET 0x14    // 状态寄存器
#define UART_WDATA_OFFSET 0x1c     // 写数据寄存器
```

**控制寄存器格式：**

- 位 [31:16]：NCO 值（用于波特率生成）
- 位 [1:0]：使能位（TX 使能、RX 使能）

**状态寄存器：**

- 位 [0]：TX FIFO 满标志

#### GPIO IP 核

GPIO（General Purpose Input/Output，通用输入输出）提供可编程的数字 I/O。

**寄存器地址：**

```c
#define GPIO_BASE 0x40030000
#define GPIO_DATA_IN 0x00      // 输入数据寄存器
#define GPIO_DATA_OUT 0x04     // 输出数据寄存器
#define GPIO_OUT_EN 0x08       // 输出使能寄存器
```

**使用方法：**

1. 设置输出使能：`REG32(GPIO_OUT_EN) = 0xFF;`  // 所有引脚配置为输出
2. 写输出数据：`REG32(GPIO_DATA_OUT) = 0xAA;`
3. 读输入数据：`val = REG32(GPIO_DATA_IN);`

#### SPI Master IP 核

SPI（Serial Peripheral Interface，串行外设接口）用于连接外部 SPI 设备。

**信号定义：**

- `spim_sclk_o`：SPI 时钟输出
- `spim_csb_o`：片选信号（低电平有效）
- `spim_mosi_o`：主机输出从机输入
- `spim_miso_i`：主机输入从机输出

**引脚映射（PMOD4 接口）：**

```
PMOD4_1 (AY38) -> MOSI
PMOD4_2 (BA39) -> MISO
PMOD4_7 (AY40) -> SCLK
PMOD4_8 (BA40) -> CS
```

### 10.4.4 如何集成 IP 核

**步骤 1：创建 IP 核目录**

在 `fpga/ip/` 下创建新目录，例如 `my_ip/`。

**步骤 2：编写 RTL 代码**

创建 Verilog/SystemVerilog 文件，实现 IP 核功能。

**步骤 3：创建 .core 文件**

定义 IP 核的元数据：

```yaml
CAPI=2:
name: "com.google.coralnpu:fpga:my_ip:0.1"
description: "My custom IP core"

filesets:
  files_rtl:
    files:
      - my_ip.sv
    file_type: systemVerilogSource

targets:
  default:
    filesets:
      - files_rtl
```

**步骤 4：在顶层模块中实例化**

在 `coralnpu_soc.sv` 中添加 IP 核实例：

```systemverilog
my_ip u_my_ip (
  .clk_i(clk_i),
  .rst_ni(rst_ni),
  // 其他信号连接
);
```

**步骤 5：添加到构建系统**

在 `fpga/BUILD` 文件中添加依赖：

```python
filegroup(
    name = "rtl_files",
    srcs = [
        # ... 其他文件
        "//fpga/ip/my_ip:rtl_files",
    ],
)
```

**步骤 6：添加约束**

如果 IP 核有外部引脚，在 `pins_nexus.xdc` 中添加引脚约束：

```tcl
set_property -dict { PACKAGE_PIN XX00 IOSTANDARD LVCMOS18 } [get_ports { my_signal }];
```

## 10.5 FPGA 软件

### 10.5.1 UART 库

UART 库提供了简单的串口通信功能，用于调试输出。

**头文件**：`fpga/sw/uart.h`

**API 函数：**

```c
// 初始化 UART，参数为系统时钟频率（MHz）
void uart_init(uint32_t clock_frequency_mhz);

// 发送单个字符
void uart_putc(char c);

// 发送字符串
void uart_puts(const char* s);

// 发送 8 位十六进制数
void uart_puthex8(uint8_t v);

// 发送 32 位十六进制数
void uart_puthex32(uint32_t v);
```

**使用示例：**

```c
#include "fpga/sw/uart.h"

int main() {
  // 初始化 UART（假设时钟为 80 MHz）
  uart_init(80);
  
  // 输出调试信息
  uart_puts("Hello, CoralNPU!\n");
  
  // 输出十六进制数
  uint32_t value = 0xDEADBEEF;
  uart_puts("Value: 0x");
  uart_puthex32(value);
  uart_puts("\n");
  
  return 0;
}
```

**工作原理：**

1. **波特率生成**：
   - UART 使用 NCO（Numerically Controlled Oscillator）生成波特率
   - 公式：`NCO = (波特率 << 20) / 时钟频率`
   - 例如：115200 bps @ 80 MHz = (115200 << 20) / 80000000 ≈ 1509

2. **发送流程**：
   - 检查 TX FIFO 是否满（读取状态寄存器位 [0]）
   - 如果不满，将字符写入 WDATA 寄存器
   - UART 硬件自动将字符串行化并发送

**基本概念解释：**

- **波特率（Baud Rate）**：每秒传输的符号数，对于 UART 通常等于比特率
  - 115200 bps = 每秒传输 115200 位
  - 每个字符 10 位（1 起始位 + 8 数据位 + 1 停止位）= 11520 字符/秒
- **FIFO**：先进先出队列，用于缓冲数据
- **串行化（Serialization）**：将并行数据（8 位）转换为串行数据（1 位）

### 10.5.2 程序加载器

CoralNPU 支持两种方式加载程序到 FPGA：

#### 方法 1：通过比特流嵌入

**适用场景**：程序较小，不需要频繁更新

**步骤：**

1. 编译程序生成二进制文件（.bin）
2. 在综合时指定 `MemInitFile` 参数
3. Vivado 将程序嵌入到 BRAM 中
4. 加载比特流时，程序自动加载到内存

**示例：**

```bash
bazel build //fpga:build_chip_nexus_bitstream \
  --//fpga:MemInitFile=path/to/program.bin
```

#### 方法 2：通过 SPI 加载

**适用场景**：程序较大，需要频繁更新

**步骤：**

1. 加载基础比特流（不包含程序）
2. 通过 SPI 接口将程序传输到 FPGA
3. 程序写入 DDR4 或 SRAM
4. 复位处理器，开始执行

**SPI 加载协议：**

- SPI Slave 接口连接到加载器（如 Raspberry Pi）
- 使用简单的命令协议：
  - 写命令：地址 + 数据
  - 读命令：地址 -> 数据
  - 启动命令：复位处理器

#### 方法 3：通过 JTAG 调试

**适用场景**：开发调试阶段

使用 OpenOCD 或 Xilinx XSDB 通过 JTAG 接口：
1. 连接到 RISC-V 调试模块
2. 停止处理器
3. 写入程序到内存
4. 设置 PC（程序计数器）
5. 恢复执行

### 10.5.3 调试方法

#### 串口调试

最简单的调试方法是使用 UART 输出：

```c
uart_puts("Checkpoint 1\n");
uart_puthex32(register_value);
```

**连接串口：**

```bash
# 使用 minicom
minicom -D /dev/ttyUSB0 -b 115200

# 或使用 screen
screen /dev/ttyUSB0 115200
```

#### LED 指示灯

使用板载 LED 指示程序状态：

- `io_halted`：处理器停止（可能是 WFI 指令或异常）
- `io_fault`：发生错误
- 自定义 GPIO LED：可以用于指示程序执行阶段

#### JTAG 调试

使用 GDB 通过 JTAG 进行源码级调试：

**步骤 1：启动 OpenOCD**

```bash
openocd -f interface/ftdi/digilent-hs1.cfg \
        -f target/riscv.cfg
```

**步骤 2：连接 GDB**

```bash
riscv32-unknown-elf-gdb program.elf
(gdb) target remote localhost:3333
(gdb) load
(gdb) break main
(gdb) continue
```

**GDB 常用命令：**

- `break <函数名>`：设置断点
- `continue`：继续执行
- `step`：单步执行（进入函数）
- `next`：单步执行（不进入函数）
- `print <变量>`：打印变量值
- `info registers`：查看寄存器
- `backtrace`：查看调用栈

#### Vivado ILA（集成逻辑分析仪）

ILA 可以在 FPGA 内部捕获信号波形：

**步骤 1：在 RTL 中插入 ILA 核**

```systemverilog
ila_0 u_ila (
  .clk(clk_i),
  .probe0(signal_to_debug),
  .probe1(another_signal)
);
```

**步骤 2：综合并生成比特流**

Vivado 会生成 `.ltx` 文件（探针定义）。

**步骤 3：使用 Hardware Manager 捕获波形**

1. 打开 Vivado Hardware Manager
2. 连接到 FPGA
3. 加载 `.ltx` 文件
4. 设置触发条件
5. 运行并捕获波形

**适用场景：**

- 调试时序问题
- 观察总线事务
- 分析复杂的硬件行为

#### ChipWhisperer（可选）

如果需要进行功耗分析或侧信道分析，可以使用 ChipWhisperer 工具。

### 10.5.4 性能分析

**资源利用率**

综合后，Vivado 会报告资源使用情况：

```
+----------------------------+--------+-------+
| Resource                   | Used   | Avail |
+----------------------------+--------+-------+
| LUT                        | 45000  | 500K  |
| FF                         | 60000  | 1M    |
| BRAM                       | 200    | 960   |
| DSP                        | 50     | 1200  |
+----------------------------+--------+-------+
```

**基本概念解释：**

- **LUT（Look-Up Table）**：查找表，实现组合逻辑
  - 6 输入 LUT 可以实现任意 6 输入的布尔函数
- **FF（Flip-Flop）**：触发器，存储 1 位数据
- **BRAM（Block RAM）**：块 RAM，片上存储器
- **DSP**：数字信号处理单元，用于乘法、加法等运算

**时钟频率**

时序报告显示最大工作频率：

```
Worst Negative Slack (WNS): 0.5 ns
Timing Met: Yes
Maximum Frequency: 85 MHz
```

- **WNS（Worst Negative Slack）**：最差时序余量
  - 正值：时序满足
  - 负值：时序违例
- **TNS（Total Negative Slack）**：所有违例路径的余量总和

**功耗估计**

Vivado 可以估算功耗：

```
Total On-Chip Power: 15.2 W
  Dynamic Power: 12.5 W
  Static Power: 2.7 W
```

## 10.6 常见问题

### 10.6.1 综合失败

**问题**：综合时报错 "Cannot find module"

**解决方法**：
1. 检查 `.core` 文件中的依赖关系
2. 确保所有源文件都在 `filesets` 中列出
3. 检查文件路径是否正确

### 10.6.2 时序违例

**问题**：布局布线后报告时序违例

**解决方法**：
1. 降低目标时钟频率
2. 添加流水线寄存器
3. 使用更快的 FPGA 速度等级
4. 优化关键路径的逻辑

### 10.6.3 DDR4 初始化失败

**问题**：`ddr_cal_complete_o` 一直为低电平

**解决方法**：
1. 检查 DDR4 时钟是否正确
2. 确认复位信号正确
3. 检查 DDR4 引脚约束
4. 查看 Vivado 的 DDR4 IP 配置

### 10.6.4 UART 无输出

**问题**：串口终端没有收到数据

**解决方法**：
1. 检查波特率设置（115200）
2. 确认 UART 引脚连接正确
3. 检查 TX/RX 是否接反
4. 使用示波器或逻辑分析仪检查 TX 信号

### 10.6.5 程序不执行

**问题**：加载程序后处理器不运行

**解决方法**：
1. 检查程序是否正确加载到内存
2. 确认复位向量地址正确
3. 使用 JTAG 检查 PC 寄存器
4. 检查 `io_halted` 和 `io_fault` LED

## 10.7 进阶主题

### 10.7.1 多时钟域设计

CoralNPU 包含多个时钟域：
- 主时钟（80 MHz）
- DDR4 时钟（动态）
- SPI 时钟（外部）
- JTAG 时钟（外部）

**时钟域交叉（CDC）处理：**

1. **双触发器同步器**：用于单比特信号
2. **异步 FIFO**：用于多比特数据
3. **握手协议**：用于控制信号

**基本概念解释：**

- **亚稳态（Metastability）**：当信号在时钟沿附近变化时，触发器可能进入不稳定状态
- **同步器（Synchronizer）**：使用多级触发器降低亚稳态概率
- **MTBF（Mean Time Between Failures）**：平均故障间隔时间，衡量可靠性

### 10.7.2 部分重配置

Xilinx UltraScale+ 支持部分重配置（Partial Reconfiguration），可以在运行时更新部分 FPGA 逻辑。

**应用场景：**
- 动态加载不同的加速器
- 节省 FPGA 资源
- 快速切换功能

### 10.7.3 高级调试技术

**VIO（Virtual I/O）**：
- 通过 JTAG 动态控制信号
- 无需重新综合即可修改参数

**System ILA**：
- 监控 AXI 总线事务
- 自动解析协议

**Vivado Simulator**：
- 在 PC 上仿真整个设计
- 比 FPGA 调试更快，但不能发现时序问题

## 10.8 总结

本章介绍了 CoralNPU 在 FPGA 上的实现：

1. **FPGA 基础**：了解了 FPGA 的基本概念和优势
2. **开发板支持**：Nexus 开发板的硬件规格和接口
3. **构建流程**：使用 FuseSoC 和 Vivado 进行综合、布局布线
4. **IP 核集成**：内存控制器、UART、GPIO、SPI 等外设
5. **软件开发**：UART 库、程序加载、调试方法
6. **问题排查**：常见问题的解决方法

通过 FPGA 实现，可以在真实硬件上验证 CoralNPU 的设计，为最终的芯片流片做好准备。

**下一步学习建议：**

- 尝试在 FPGA 上运行示例程序
- 学习 Verilog/SystemVerilog 语言
- 深入了解 RISC-V 架构
- 探索 Chisel 硬件构造语言（CoralNPU 的核心使用 Chisel 编写）

**参考资源：**

- Xilinx UltraScale+ 文档：https://www.xilinx.com/products/silicon-devices/fpga/virtex-ultrascale-plus.html
- FuseSoC 文档：https://fusesoc.readthedocs.io/
- RISC-V 规范：https://riscv.org/technical/specifications/
- OpenOCD 文档：https://openocd.org/doc/
