# 第十三章：调试

## 13.1 调试概述

调试是处理器开发和应用开发中非常重要的环节。CoralNPU 实现了符合 RISC-V Debug Specification 的调试模块，提供了完整的调试支持。

### 13.1.1 调试方法

CoralNPU 支持以下几种调试方法：

1. **硬件调试**：通过 JTAG 接口连接外部调试器，可以暂停、恢复、单步执行处理器，读写寄存器和内存
2. **软件调试**：使用 GDB（GNU Debugger）进行源代码级调试
3. **追踪调试**：通过 RVVI（RISC-V Verification Interface）追踪指令执行流程
4. **仿真调试**：在仿真环境中进行调试，可以观察内部信号

### 13.1.2 调试工具

CoralNPU 调试需要以下工具：

- **OpenOCD**：开源的片上调试器，负责与 JTAG 接口通信
- **GDB**：GNU 调试器，提供源代码级调试功能
- **JTAG 适配器**：硬件调试时需要 JTAG 调试器（如 J-Link、FT2232 等）
- **仿真器**：Verilator 或其他 HDL 仿真工具

## 13.2 调试模块

### 13.2.1 RISC-V Debug Module

CoralNPU 的调试模块实现了 RISC-V Debug Specification v0.13.2 的子集。调试模块是一个独立的硬件模块，通过专用接口连接到处理器核心。

**调试模块的主要功能：**

- 暂停（halt）和恢复（resume）处理器执行
- 单步执行指令
- 读写通用寄存器（GPR）
- 读写浮点寄存器（FPR）
- 读写控制状态寄存器（CSR）
- 读写内存（ITCM、DTCM）
- 复位控制

**架构说明：**

调试模块通过一个请求/响应协议与外部调试器通信。外部调试器（如 OpenOCD）通过读写内存映射的 CSR 寄存器来控制调试模块。

### 13.2.2 调试寄存器

调试模块提供了两组寄存器：

#### AXI CSR 接口寄存器

这些寄存器映射到 CoralNPU 的 CSR 地址空间，用于与调试模块通信：

| 地址       | 名称      | 描述                                           |
|-----------|-----------|------------------------------------------------|
| 0x30800   | req_addr  | 写入目标调试模块寄存器地址                      |
| 0x30804   | req_data  | 写入调试模块操作的数据                          |
| 0x30808   | req_op    | 写入操作类型（READ=1, WRITE=2）来启动调试命令   |
| 0x3080c   | rsp_data  | 命令完成后，数据结果在这里                      |
| 0x30810   | rsp_op    | 命令完成后，状态结果在这里（SUCCESS=0, FAILED=2）|
| 0x30814   | status    | 状态寄存器。位0表示模块是否就绪，位1表示响应是否可用 |

**解释：**
- `req_addr`、`req_data`、`req_op` 是用来发送命令给调试模块的
- `rsp_data`、`rsp_op` 是调试模块返回的结果
- `status` 寄存器用来查询调试模块的状态

#### 内部调试模块寄存器

调试模块内部实现了以下寄存器，通过 AXI CSR 接口访问：

| 地址   | 名称        | 描述                           |
|--------|-------------|--------------------------------|
| 0x04   | data0       | 抽象命令的数据寄存器            |
| 0x10   | dmcontrol   | 调试模块控制寄存器              |
| 0x11   | dmstatus    | 调试模块状态寄存器              |
| 0x12   | hartinfo    | Hart 信息寄存器                |
| 0x16   | abstractcs  | 抽象命令状态寄存器              |
| 0x17   | command     | 抽象命令寄存器                 |

**重要寄存器详解：**

**dmcontrol (0x10) - 调试模块控制寄存器**

| 位    | 名称      | 描述                           |
|-------|-----------|--------------------------------|
| 31    | haltreq   | 请求暂停处理器                  |
| 30    | resumereq | 请求恢复处理器                  |
| 29:2  | reserved  | 保留                           |
| 1     | ndmreset  | 复位调试模块                    |
| 0     | dmactive  | 激活调试模块                    |

**dmstatus (0x11) - 调试模块状态寄存器**

| 位    | 名称       | 描述                           |
|-------|------------|--------------------------------|
| 11    | allrunning | 所有 hart 都在运行              |
| 10    | anyrunning | 任意 hart 在运行                |
| 9     | allhalted  | 所有 hart 都已暂停              |
| 8     | anyhalted  | 任意 hart 已暂停                |
| 7:4   | reserved   | 保留                           |
| 3:0   | version    | 调试模块版本                    |

**abstractcs (0x16) - 抽象命令状态寄存器**

| 位    | 名称     | 描述                           |
|-------|----------|--------------------------------|
| 12    | busy     | 抽象命令正在执行                |
| 11    | reserved | 保留                           |
| 10:8  | cmderr   | 抽象命令错误码                  |
| 7:0   | reserved | 保留                           |

**command (0x17) - 抽象命令寄存器**

| 位      | 名称      | 描述                           |
|---------|-----------|--------------------------------|
| 31:24   | cmdtype   | 抽象命令类型                    |
| 23:0    | control   | 命令特定的控制信息              |

### 13.2.3 调试功能

#### 暂停和恢复处理器

**暂停处理器：**

1. 写入 `0x10`（dmcontrol 地址）到 `req_addr` (0x30800)
2. 写入 `0x80000001`（设置 haltreq 位和 dmactive 位）到 `req_data` (0x30804)
3. 写入 `2`（WRITE 操作）到 `req_op` (0x30808)
4. 等待操作完成

**恢复处理器：**

1. 写入 `0x10` 到 `req_addr`
2. 写入 `0x40000001`（设置 resumereq 位和 dmactive 位）到 `req_data`
3. 写入 `2` 到 `req_op`
4. 等待操作完成

#### 读写寄存器

调试模块使用"抽象命令"来读写处理器寄存器。

**寄存器编号：**

- `0x0000-0x0FFF`：CSR 寄存器
- `0x1000-0x101F`：通用寄存器（GPR）
  - 例如：`x0` = 0x1000, `x1` = 0x1001, ..., `a0` (x10) = 0x100A
- `0x1020-0x103F`：浮点寄存器（FPR）
  - 例如：`f0` = 0x1020, `f1` = 0x1021, ...

**读取通用寄存器 a0 的步骤：**

1. 写入 `0x17`（command 地址）到 `req_addr`
2. 写入 `0x0000100A`（cmdtype=0, write=0, regno=0x100A）到 `req_data`
   - cmdtype=0 表示"访问寄存器"命令
   - write=0 表示读取
   - regno=0x100A 表示 a0 寄存器
3. 写入 `2` 到 `req_op`
4. 等待命令完成（轮询 abstractcs 的 busy 位）
5. 读取 data0 寄存器获取 a0 的值

**写入通用寄存器 a0 的步骤：**

1. 写入要设置的值到 data0 寄存器（地址 0x04）
2. 写入 `0x17` 到 `req_addr`
3. 写入 `0x0001100A`（cmdtype=0, write=1, regno=0x100A）到 `req_data`
4. 写入 `2` 到 `req_op`
5. 等待命令完成

#### 读写内存

调试模块也支持通过抽象命令读写内存（ITCM 和 DTCM）。

**读取内存的步骤：**

1. 写入内存地址到 data1 寄存器（地址 0x05）
2. 发送"访问内存"抽象命令（cmdtype=2）
3. 从 data0 寄存器读取数据

## 13.3 JTAG 接口

### 13.3.1 JTAG 协议

JTAG（Joint Test Action Group）是一种标准的调试接口协议，广泛用于嵌入式系统调试。

**JTAG 的基本概念：**

- **TAP（Test Access Port）**：JTAG 接口的硬件端口
- **IR（Instruction Register）**：指令寄存器，用于选择操作
- **DR（Data Register）**：数据寄存器，用于传输数据
- **状态机**：JTAG 有一个状态机控制操作流程

CoralNPU 的 JTAG TAP 使用 5 位指令寄存器（IR length = 5）。

### 13.3.2 JTAG 引脚

JTAG 接口需要以下信号：

| 信号名 | 方向 | 描述                           |
|--------|------|--------------------------------|
| TCK    | 输入 | 测试时钟（Test Clock）          |
| TMS    | 输入 | 测试模式选择（Test Mode Select）|
| TDI    | 输入 | 测试数据输入（Test Data In）    |
| TDO    | 输出 | 测试数据输出（Test Data Out）   |
| TRST   | 输入 | 测试复位（Test Reset，可选）    |

**解释：**
- TCK 是时钟信号，所有 JTAG 操作都在这个时钟的边沿进行
- TMS 用于控制 JTAG 状态机的状态转换
- TDI 和 TDO 用于串行传输数据
- TRST 用于复位 JTAG 接口（有些系统不使用这个信号）

### 13.3.3 JTAG 连接

**硬件连接：**

在 FPGA 或 ASIC 上，需要将 JTAG 调试器（如 J-Link、FT2232）连接到 CoralNPU 的 JTAG 引脚。

**仿真连接：**

在仿真环境中，CoralNPU 使用 OpenOCD 的 `remote_bitbang` 适配器。这是一个通过 TCP 端口模拟 JTAG 信号的方法。

仿真器会在指定端口（默认 44853）监听 OpenOCD 的连接。

## 13.4 OpenOCD

### 13.4.1 OpenOCD 简介

OpenOCD（Open On-Chip Debugger）是一个开源的片上调试工具，支持多种调试接口和目标处理器。

**OpenOCD 的作用：**

- 与 JTAG 接口通信
- 实现 RISC-V Debug Specification 协议
- 提供 GDB 服务器，让 GDB 可以连接
- 支持多种调试适配器

### 13.4.2 配置文件

CoralNPU 的 OpenOCD 配置文件位于 `jtag-sim.cfg`：

```tcl
adapter driver remote_bitbang
remote_bitbang port 44853
jtag newtap riscv tap -irlen 5 -expected-id 0x04f5484d
target create riscv.tap.0 riscv -chain-position riscv.tap -rtos hwthread
riscv set_mem_access abstract
gdb_breakpoint_override hard
```

**配置说明：**

- `adapter driver remote_bitbang`：使用 remote_bitbang 适配器（用于仿真）
- `remote_bitbang port 44853`：连接到端口 44853
- `jtag newtap riscv tap -irlen 5`：创建一个 JTAG TAP，IR 长度为 5
- `-expected-id 0x04f5484d`：期望的 JTAG ID
- `target create riscv.tap.0 riscv`：创建一个 RISC-V 目标
- `riscv set_mem_access abstract`：使用抽象命令访问内存
- `gdb_breakpoint_override hard`：使用硬件断点

**解释：**
- `remote_bitbang` 是一种通过网络模拟 JTAG 信号的方式，适合仿真环境
- `-irlen 5` 表示 JTAG 指令寄存器是 5 位的
- `abstract` 内存访问方式表示通过调试模块的抽象命令来读写内存

### 13.4.3 如何使用 OpenOCD

**启动 OpenOCD：**

```bash
openocd -f jtag-sim.cfg
```

OpenOCD 会连接到仿真器，并在端口 3333 上启动 GDB 服务器。

**OpenOCD 输出示例：**

```
Open On-Chip Debugger 0.12.0
Info : Listening on port 6666 for tcl connections
Info : Listening on port 4444 for telnet connections
Info : clock speed 1000 kHz
Info : JTAG tap: riscv.tap tap/device found: 0x04f5484d
Info : datacount=1 progbufsize=0
Info : Examined RISC-V core; found 1 harts
Info :  hart 0: XLEN=32, misa=0x40101105
Info : starting gdb server for riscv.tap.0 on 3333
Info : Listening on port 3333 for gdb connections
```

**通过 telnet 连接 OpenOCD：**

```bash
telnet localhost 4444
```

可以在 telnet 中执行 OpenOCD 命令，例如：

```
> halt
> reg pc
> resume
```

## 13.5 GDB 调试

### 13.5.1 连接 GDB

GDB 是 GNU 提供的强大调试器，支持源代码级调试。

**启动 GDB：**

```bash
riscv32-unknown-elf-gdb your_program.elf
```

**在 GDB 中连接到 OpenOCD：**

```gdb
(gdb) target remote localhost:3333
```

连接成功后，GDB 会显示当前处理器状态。

### 13.5.2 常用调试命令

**加载程序：**

```gdb
(gdb) load
```

这会将 ELF 文件加载到处理器内存中。

**查看源代码：**

```gdb
(gdb) list
```

显示当前位置的源代码。

**查看反汇编：**

```gdb
(gdb) disassemble
```

显示当前函数的汇编代码。

**继续执行：**

```gdb
(gdb) continue
```

或简写为 `c`。

**退出 GDB：**

```gdb
(gdb) quit
```

### 13.5.3 断点、单步执行

**设置断点：**

```gdb
(gdb) break main
(gdb) break file.c:42
(gdb) break *0x80000100
```

- `break main`：在 main 函数入口设置断点
- `break file.c:42`：在源文件的第 42 行设置断点
- `break *0x80000100`：在地址 0x80000100 设置断点

**查看断点：**

```gdb
(gdb) info breakpoints
```

**删除断点：**

```gdb
(gdb) delete 1
```

删除编号为 1 的断点。

**单步执行：**

```gdb
(gdb) step
```

或简写为 `s`。执行一行源代码，如果是函数调用会进入函数内部。

```gdb
(gdb) next
```

或简写为 `n`。执行一行源代码，如果是函数调用不会进入函数内部。

**汇编级单步：**

```gdb
(gdb) stepi
```

或简写为 `si`。执行一条汇编指令。

```gdb
(gdb) nexti
```

或简写为 `ni`。执行一条汇编指令，不进入函数调用。

### 13.5.4 查看寄存器和内存

**查看所有寄存器：**

```gdb
(gdb) info registers
```

或简写为 `info reg`。

**查看特定寄存器：**

```gdb
(gdb) print $pc
(gdb) print $sp
(gdb) print $a0
```

- `$pc`：程序计数器（Program Counter）
- `$sp`：栈指针（Stack Pointer）
- `$a0`：参数寄存器 a0

**查看内存：**

```gdb
(gdb) x/10x 0x80000000
```

以十六进制格式显示从地址 0x80000000 开始的 10 个字（word）。

```gdb
(gdb) x/10i $pc
```

显示从当前 PC 开始的 10 条指令。

**格式说明：**
- `x` 命令用于查看内存
- `/10x` 表示显示 10 个单位，以十六进制格式
- `/10i` 表示显示 10 条指令

**查看变量：**

```gdb
(gdb) print variable_name
(gdb) print *pointer
```

**查看调用栈：**

```gdb
(gdb) backtrace
```

或简写为 `bt`。显示函数调用栈。

**切换栈帧：**

```gdb
(gdb) frame 1
```

切换到栈帧 1。

## 13.6 追踪功能

### 13.6.1 RVVI 追踪

RVVI（RISC-V Verification Interface）是一种标准的指令追踪接口，用于验证和调试。

**RVVI 的作用：**

- 记录每条指令的执行信息
- 包括 PC、指令编码、寄存器变化、内存访问等
- 可以用于对比参考模型，验证处理器正确性
- 可以用于性能分析

**RVVI 追踪信息包括：**

- 指令地址（PC）
- 指令编码
- 指令类型
- 源寄存器和目标寄存器
- 内存访问地址和数据
- 异常和中断信息

### 13.6.2 指令追踪

在仿真环境中，可以启用指令追踪功能。

**Verilator 仿真追踪：**

```bash
./sim +trace
```

这会生成波形文件（VCD 或 FST 格式），可以用 GTKWave 等工具查看。

**查看波形：**

```bash
gtkwave dump.vcd
```

在波形查看器中，可以观察：
- 处理器内部信号
- 流水线状态
- 寄存器变化
- 内存访问

### 13.6.3 性能追踪

**性能计数器：**

CoralNPU 实现了 RISC-V 的性能计数器 CSR：

- `mcycle`：周期计数器
- `minstret`：指令计数器
- `mhpmcounter3-31`：硬件性能监控计数器

**读取性能计数器：**

在程序中可以使用 CSR 指令读取：

```c
uint32_t cycles, instret;
asm volatile ("csrr %0, mcycle" : "=r"(cycles));
asm volatile ("csrr %0, minstret" : "=r"(instret));
printf("Cycles: %u, Instructions: %u\n", cycles, instret);
```

**解释：**
- `csrr` 是 RISC-V 的 CSR 读取指令
- `mcycle` 记录了处理器运行的总周期数
- `minstret` 记录了处理器执行的总指令数
- 通过这两个值可以计算 CPI（Cycles Per Instruction）

**在 GDB 中读取：**

```gdb
(gdb) print/x $mcycle
(gdb) print/x $minstret
```

## 13.7 常见问题

### 13.7.1 程序无法运行

**问题：程序加载后无法执行或立即崩溃**

**可能原因和解决方法：**

1. **链接脚本错误**
   - 检查链接脚本中的内存地址是否正确
   - 确保程序入口地址在有效的内存范围内
   - CoralNPU 的 ITCM 通常从 0x80000000 开始

2. **栈指针未初始化**
   - 检查启动代码是否正确设置了栈指针（sp）
   - 栈应该在 DTCM 的有效范围内

3. **中断向量表错误**
   - 检查 `mtvec` CSR 是否指向正确的中断向量表
   - 确保中断处理程序存在

4. **权限问题**
   - 检查是否在正确的特权级别（Machine mode）运行
   - 某些 CSR 只能在 Machine mode 访问

**调试步骤：**

```gdb
(gdb) load
(gdb) break _start
(gdb) continue
(gdb) info registers
(gdb) stepi
```

逐条指令执行，观察寄存器变化。

### 13.7.2 调试连接失败

**问题：OpenOCD 无法连接到目标**

**可能原因和解决方法：**

1. **仿真器未启动**
   - 确保仿真器正在运行并监听 JTAG 端口
   - 检查端口号是否正确（默认 44853）

2. **端口被占用**
   - 使用 `netstat -an | grep 44853` 检查端口状态
   - 如果端口被占用，关闭占用端口的进程或更改端口号

3. **JTAG ID 不匹配**
   - OpenOCD 配置中的 `expected-id` 应该与实际 JTAG ID 匹配
   - 可以在配置中注释掉 `-expected-id` 选项来跳过检查

4. **OpenOCD 版本问题**
   - 使用较新版本的 OpenOCD（0.11.0 或更高）
   - 某些旧版本可能不支持 RISC-V

**调试步骤：**

```bash
# 检查仿真器是否在运行
ps aux | grep sim

# 检查端口
netstat -an | grep 44853

# 尝试连接
telnet localhost 44853

# 启动 OpenOCD 并查看详细输出
openocd -f jtag-sim.cfg -d3
```

### 13.7.3 其他常见问题

**问题：断点不起作用**

**解决方法：**

- 确保使用硬件断点（CoralNPU 配置为 `gdb_breakpoint_override hard`）
- 检查断点地址是否在有效的代码区域
- 某些优化可能导致断点位置不准确，尝试使用 `-O0` 编译

**问题：无��读取某些寄存器**

**解决方法：**

- 某些 CSR 只在特定条件下可读
- 检查处理器是否处于正确的特权级别
- 浮点寄存器需要浮点扩展支持

**问题：内存访问失败**

**解决方法：**

- 检查内存地址是否在有效范围内
- ITCM 和 DTCM 的地址范围在配置中定义
- 某些外设地址可能不支持调试访问

**问题：单步执行时程序行为异常**

**解决方法：**

- 某些代码（如中断处理、定时器相关）在单步执行时可能表现不同
- 尝试使用断点而不是单步执行
- 检查是否有竞态条件或时序相关的问题

**问题：GDB 显示 "Cannot access memory at address 0x..."**

**解决方法：**

- 地址可能不在有效的内存范围内
- 检查 MMU 或 PMP（Physical Memory Protection）配置
- 确保调试模块有权限访问该地址

**获取帮助：**

如果遇到无法解决的问题，可以：

1. 查看 OpenOCD 和 GDB 的详细日志输出
2. 检查 CoralNPU 的文档和示例
3. 在仿真环境中启用波形追踪，观察内部信号
4. 参考 RISC-V Debug Specification 文档

---

**本章小结：**

本章介绍了 CoralNPU 的调试功能和工具。调试是处理器开发和应用开发的重要环节，掌握调试工具的使用可以大大提高开发效率。

主要内容包括：

- 调试模块的架构和寄存器
- JTAG 接口和连接方法
- OpenOCD 的配置和使用
- GDB 调试的常用命令
- 追踪和性能分析
- 常见问题的解决方法

通过本章的学习，你应该能够：

- 理解 CoralNPU 的调试架构
- 配置和使用 OpenOCD 连接到 CoralNPU
- 使用 GDB 进行源代码级调试
- 设置断点、单步执行、查看寄存器和内存
- 使用追踪功能分析程序行为
- 解决常见的调试问题

在下一章中，我们将介绍 CoralNPU 的集成指南，讲解如何将 CoralNPU 集成到你的系统中。
