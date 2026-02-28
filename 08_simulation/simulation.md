# 第八章：仿真

## 8.1 仿真概述

### 8.1.1 什么是仿真

仿真（Simulation）是在软件环境中模拟硬件电路行为的过程。对于 CoralNPU 这样的处理器设计，仿真是验证设计正确性的关键步骤。通过仿真，我们可以：

- 在硬件制造之前验证设计的功能
- 调试硬件逻辑错误
- 测试各种边界条件和异常情况
- 评估性能指标（如时钟周期数、吞吐量等）
- 生成波形文件用于调试分析

### 8.1.2 为什么需要仿真

硬件设计不同于软件开发，一旦芯片制造完成，修改成本极高。因此在流片（Tape-out）之前，必须通过充分的仿真验证来确保设计的正确性。仿真的主要优势包括：

1. **低成本验证**：相比制造实际芯片，仿真成本几乎为零
2. **快速迭代**：发现问题后可以立即修改代码并重新仿真
3. **完全可观测性**：可以查看任何内部信号的值和波形
4. **可重复性**：相同的输入总是产生相同的输出，便于调试
5. **覆盖率分析**：可以统计代码覆盖率和功能覆盖率

### 8.1.3 CoralNPU 支持的仿真方式

CoralNPU 项目支持多种仿真工具和方法：

| 仿真方式 | 类型 | 速度 | 用途 |
|---------|------|------|------|
| Verilator | 开源 RTL 仿真器 | 快 | 日常开发和 CI 测试 |
| Cocotb | Python 测试框架 | 中等 | 编写灵活的测试用例 |
| VCS | 商业 RTL 仿真器 | 中等 | 覆盖率分析和高级验证 |
| SystemC/TLM | 系统级建模 | 很快 | 系统级验证和性能评估 |
| coralnpu_sim | 功能模拟器 | 极快 | 软件开发和算法验证 |

不同的仿真方式适用于不同的场景：
- **Verilator**：适合快速验证 RTL 逻辑，是 CI/CD 的首选
- **Cocotb**：适合编写复杂的测试场景，Python 语法简单易学
- **VCS**：适合需要覆盖率分析和形式验证的场景
- **SystemC**：适合系统级验证，可以与其他 IP 集成
- **coralnpu_sim**：适合软件开发，不需要 RTL 仿真的开销

## 8.2 Verilator 仿真

### 8.2.1 Verilator 简介

Verilator 是一个开源的 Verilog/SystemVerilog 仿真器，它将 RTL 代码转换为 C++ 或 SystemC 代码，然后编译成可执行文件。相比传统的事件驱动仿真器（如 ModelSim、VCS），Verilator 采用周期精确（Cycle-accurate）的仿真方式，速度更快，特别适合大规模设计的仿真。

**Verilator 的特点：**
- 开源免费，无需商业许可证
- 仿真速度快，通常比传统仿真器快 10-100 倍
- 生成的 C++ 代码可以与其他 C++ 测试框架集成
- 支持 SystemC 接口
- 支持波形生成（VCD 格式）

**Verilator 的局限性：**
- 不支持所有 SystemVerilog 特性（如延迟语句 `#10`）
- 主要用于综合后的 RTL 代码，不适合行为级建模
- 调试相对困难，需要依赖波形文件

### 8.2.2 构建 Verilator 仿真器

CoralNPU 使用 Bazel 构建系统来管理 Verilator 仿真。构建过程会自动完成以下步骤：

1. 使用 Chisel 生成 SystemVerilog 代码
2. 使用 Verilator 将 SystemVerilog 转换为 C++ 代码
3. 编译 C++ 代码和测试台（Testbench）
4. 链接生成可执行文件

**构建示例：**

```bash
# 构建 CoreMiniAxi 的 Verilator 仿真器
cd /home/curry/code/coralnpu
bazel build //tests/verilator_sim:core_mini_axi_sim

# 构建带向量扩展的核心仿真器
bazel build //tests/verilator_sim:rvv_core_mini_axi_sim
```

**构建选项说明：**

在 `tests/verilator_sim/BUILD` 文件中，可以看到 Verilator 的构建配置：

```python
cc_binary(
    name = "core_mini_axi_sim",
    srcs = [
        "coralnpu/core_mini_axi_tb.cc",
        "@coralnpu_hw//hdl/chisel/src/coralnpu:VCoreMiniAxi_parameters.h",
    ],
    defines = [
        "VERILATOR_MODEL=VCoreMiniAxi",
    ],
    deps = [
        ":elf",
        ":sim_libs",
        "//hdl/chisel/src/coralnpu:core_mini_axi_cc_library",
        "@accellera_systemc//:systemc",
    ],
)
```

这里的关键点：
- `srcs`：包含测试台的 C++ 源文件
- `defines`：定义 Verilator 生成的模型名称
- `deps`：依赖项，包括 ELF 加载器、SystemC 库等

### 8.2.3 运行仿真

构建完成后，可以直接运行仿真器：

```bash
# 运行仿真，加载 ELF 文件
bazel run //tests/verilator_sim:core_mini_axi_sim -- \
    --elf_file=/path/to/your/program.elf

# 运行仿真并生成波形文件
bazel run //tests/verilator_sim:core_mini_axi_sim -- \
    --elf_file=/path/to/your/program.elf \
    --vcd_file=output.vcd
```

**常用命令行参数：**

- `--elf_file`：指定要加载的 ELF 可执行文件
- `--vcd_file`：指定输出的波形文件路径（VCD 格式）
- `--max_cycles`：设置最大仿真周期数，防止死循环
- `--trace`：启用波形跟踪（会显著降低仿真速度）

**示例：运行一个简单的测试程序**

```bash
# 首先编译一个测试程序
bazel build //tests/cocotb:noop.elf

# 运行仿真
bazel run //tests/verilator_sim:core_mini_axi_sim -- \
    --elf_file=$(bazel info bazel-bin)/tests/cocotb/noop.elf \
    --vcd_file=/tmp/noop.vcd
```

### 8.2.4 波形查看（VCD 文件）

VCD（Value Change Dump）是一种标准的波形文件格式。生成 VCD 文件后，可以使用波形查看器来分析信号变化。

**推荐的波形查看工具：**

1. **GTKWave**（开源，跨平台）
   ```bash
   # 安装 GTKWave
   sudo apt-get install gtkwave
   
   # 打开波形文件
   gtkwave /tmp/noop.vcd
   ```

2. **Surfer**（现代化的开源波形查看器）
   ```bash
   # 使用 cargo 安装
   cargo install surfer
   
   # 打开波形文件
   surfer /tmp/noop.vcd
   ```

**GTKWave 使用技巧：**

1. **添加信号**：在左侧的信号树中选择信号，点击 "Append" 或按 `Insert` 键
2. **搜索信号**：按 `Ctrl+F` 打开搜索对话框
3. **缩放波形**：
   - 放大：`Ctrl+鼠标滚轮向上` 或点击放大镜图标
   - 缩小：`Ctrl+鼠标滚轮向下`
   - 适应窗口：点击 "Zoom Fit" 按钮
4. **设置信号显示格式**：
   - 右键点击信号 → "Data Format"
   - 可选择十六进制、十进制、二进制等格式
5. **添加标记**：点击波形上的某个时间点，按 `M` 键添加标记

**常用的调试信号：**

- `clock`：时钟信号
- `reset`：复位信号
- `io_*`：顶层模块的输入输出信号
- `pc`：程序计数器（Program Counter）
- `instruction`：当前执行的指令
- `regfile`：寄存器堆的值

### 8.2.5 调试技巧

**1. 使用 printf 调试**

在测试台的 C++ 代码中，可以添加 `printf` 语句来输出调试信息：

```cpp
// 在 core_mini_axi_tb.cc 中
if (top->io_halted) {
    printf("Core halted at cycle %lu, PC = 0x%08x\n", 
           cycle_count, top->io_pc);
}
```

**2. 断言检查**

使用 C++ 的 `assert` 或 SystemC 的 `sc_assert` 来检查关键条件：

```cpp
#include <cassert>

// 检查内存访问是否对齐
assert((address & 0x3) == 0 && "Address must be 4-byte aligned");
```

**3. 条件波形生成**

为了减小波形文件大小和提高仿真速度，可以只在特定条件下生成波形：

```cpp
// 只在检测到错误时才开始记录波形
if (error_detected && !tracing) {
    top->trace(tfp, 99);  // 开始跟踪
    tracing = true;
}
```

**4. 使用 GDB 调试仿真器**

Verilator 生成的是普通的 C++ 程序，可以使用 GDB 进行调试：

```bash
# 使用 GDB 运行仿真器
gdb --args bazel-bin/tests/verilator_sim/core_mini_axi_sim \
    --elf_file=test.elf

# 在 GDB 中设置断点
(gdb) break core_mini_axi_tb.cc:123
(gdb) run
```

**5. 检查仿真是否卡死**

如果仿真长时间没有输出，可能是进入了死循环。可以设置最大周期数：

```bash
bazel run //tests/verilator_sim:core_mini_axi_sim -- \
    --elf_file=test.elf \
    --max_cycles=100000
```

**6. 内存访问跟踪**

在测试台中添加内存访问的日志：

```cpp
// 记录所有内存写操作
if (axi_awvalid && axi_awready) {
    printf("[WRITE] Addr=0x%08x, Data=0x%08x\n", 
           axi_awaddr, axi_wdata);
}
```

## 8.3 Cocotb 仿真

### 8.3.1 Cocotb 简介

Cocotb（Coroutine Co-simulation TestBench）是一个基于 Python 的硬件验证框架。它允许你使用 Python 编写测试用例，而不需要学习 SystemVerilog 或 VHDL 的测试语法。

**Cocotb 的优势：**

- **Python 语法**：简单易学，有丰富的第三方库支持
- **协程支持**：使用 `async/await` 语法编写并发测试
- **灵活性**：可以轻松实现复杂的测试场景和随机化测试
- **可重用性**：测试代码可以在不同的仿真器上运行（Verilator、VCS、Icarus 等）
- **调试友好**：可以使用 Python 的调试工具（如 pdb、ipdb）

**Cocotb 的工作原理：**

Cocotb 通过 VPI（Verilog Procedural Interface）或 VHPI（VHDL Procedural Interface）与仿真器通信。测试代码运行在 Python 解释器中，通过这些接口读写 RTL 信号。

### 8.3.2 编写 Cocotb 测试

**基本测试结构：**

一个典型的 Cocotb 测试文件包含以下部分：

```python
import cocotb
from cocotb.clock import Clock
from cocotb.triggers import RisingEdge, Timer

@cocotb.test()
async def my_first_test(dut):
    """这是一个简单的测试用例"""
    
    # 1. 启动时钟
    clock = Clock(dut.clock, 10, units="ns")  # 10ns 周期 = 100MHz
    cocotb.start_soon(clock.start())
    
    # 2. 复位
    dut.reset.value = 1
    await Timer(100, units="ns")
    dut.reset.value = 0
    
    # 3. 测试逻辑
    dut.input_signal.value = 0x42
    await RisingEdge(dut.clock)
    
    # 4. 检查结果
    assert dut.output_signal.value == 0x42, "Output mismatch!"
```

**关键概念解释：**

1. **`@cocotb.test()` 装饰器**：标记一个函数为测试用例
2. **`async def`**：定义异步函数（协程），这是 Python 的语法
3. **`await`**：等待某个事件发生（如时钟上升沿）
4. **`dut`**：Device Under Test，即被测试的硬件模块

**CoralNPU 的 Cocotb 测试示例：**

让我们看一个实际的例子，来自 `tests/cocotb/core_mini_axi_sim.py`：

```python
@cocotb.test()
async def core_mini_axi_basic_write_read_memory(dut):
    """基本的内存读写测试"""
    
    # 创建 AXI 接口对象
    core_mini_axi = CoreMiniAxiInterface(dut)
    await core_mini_axi.init()
    await core_mini_axi.reset()
    cocotb.start_soon(core_mini_axi.clock.start())
    
    # 写入一个字（32 位）
    await core_mini_axi.write_word(0x100, 0x42)
    await core_mini_axi.write_word(0x104, 0x43)
    
    # 读取并验证
    rdata = (await core_mini_axi.read(0x100, 16)).view(np.uint32)
    assert (rdata[0:2] == np.array([0x42, 0x43])).all()
```

这个测试做了以下事情：
1. 初始化 AXI 接口（这是一个封装好的类，简化了 AXI 协议的操作）
2. 复位核心
3. 启动时钟
4. 向地址 `0x100` 和 `0x104` 写入数据
5. 读取数据并验证

### 8.3.3 运行 Cocotb 测试

**使用 Bazel 运行测试：**

```bash
# 运行单个测试
bazel test //tests/cocotb:core_mini_axi_sim_cocotb

# 运行所有 Cocotb 测试
bazel test //tests/cocotb/...

# 运行特定的测试用例
bazel test //tests/cocotb:core_mini_axi_sim_cocotb \
    --test_filter=core_mini_axi_basic_write_read_memory
```

**查看测试结果：**

```bash
# 测试通过时的输出
INFO: Analyzed target //tests/cocotb:core_mini_axi_sim_cocotb (0 packages loaded)
INFO: Found 1 test target...
Target //tests/cocotb:core_mini_axi_sim_cocotb up-to-date:
  bazel-bin/tests/cocotb/core_mini_axi_sim_cocotb
INFO: Elapsed time: 45.234s, Critical Path: 44.12s
//tests/cocotb:core_mini_axi_sim_cocotb                          PASSED in 42.3s

# 测试失败时，可以查看详细日志
bazel test //tests/cocotb:core_mini_axi_sim_cocotb --test_output=all
```

**调试失败的测试：**

如果测试失败，Bazel 会告诉你日志文件的位置：

```bash
# 查看测试日志
cat bazel-testlogs/tests/cocotb/core_mini_axi_sim_cocotb/test.log
```

### 8.3.4 测试示例

**示例 1：测试 WFI 指令**

WFI（Wait For Interrupt）是 RISC-V 的等待中断指令。这个测试验证 WFI 在不同发射槽（Issue Slot）中的行为：

```python
@cocotb.test()
async def core_mini_axi_run_wfi_in_all_slots(dut):
    """测试 WFI 指令在每个发射槽中的行为"""
    core_mini_axi = CoreMiniAxiInterface(dut)
    await core_mini_axi.init()
    await core_mini_axi.reset()
    cocotb.start_soon(core_mini_axi.clock.start())
    r = runfiles.Create()
    
    # 测试 4 个发射槽
    for slot in range(0, 4):
        # 加载对应的 ELF 文件
        elf_path = r.Rlocation(f"coralnpu_hw/tests/cocotb/wfi_slot_{slot}.elf")
        with open(elf_path, "rb") as f:
            await core_mini_axi.reset()
            entry_point = await core_mini_axi.load_elf(f)
            await core_mini_axi.execute_from(entry_point)
            
            # 等待核心进入 WFI 状态
            await core_mini_axi.wait_for_wfi()
            
            # 触发中断
            await core_mini_axi.raise_irq()
            
            # 等待核心停止
            await core_mini_axi.wait_for_halted()
```

**示例 2：测试 AXI 总线协议**

这个测试验证 AXI 总线的握手协议，特别是 `BVALID` 信号必须保持高电平直到 `BREADY` 信号到来：

```python
@cocotb.test()
async def core_mini_axi_slow_bready(dut):
    """测试 BVALID 在 BREADY 延迟时保持高电平"""
    core_mini_axi = CoreMiniAxiInterface(dut)
    await core_mini_axi.init()
    await core_mini_axi.reset()
    cocotb.start_soon(core_mini_axi.clock.start())
    
    wdata = np.arange(16, dtype=np.uint8)
    for i in range(100):
        # 随机延迟 BREADY 信号
        bready_delay = random.randint(0, 50)
        await core_mini_axi.write(0x100, wdata, bready_delay=bready_delay)
```

**示例 3：向量运算测试**

测试 RISC-V 向量扩展（RVV）的算术运算：

```python
@cocotb.test()
async def rvv_arithmetic_test(dut):
    """测试向量加法指令"""
    core = RvvCoreInterface(dut)
    await core.init()
    await core.reset()
    cocotb.start_soon(core.clock.start())
    
    # 加载向量加法测试程序
    with open("rvv_add_int8_m1.elf", "rb") as f:
        entry_point = await core.load_elf(f)
        
        # 准备输入数据
        input1 = np.arange(16, dtype=np.uint8)
        input2 = np.arange(16, dtype=np.uint8) * 2
        
        # 写入输入缓冲区
        await core.write_memory(input1_addr, input1)
        await core.write_memory(input2_addr, input2)
        
        # 执行程序
        await core.execute_from(entry_point)
        await core.wait_for_halted()
        
        # 读取结果
        output = await core.read_memory(output_addr, 16)
        expected = input1 + input2
        
        # 验证结果
        assert (output == expected).all(), "Vector addition failed!"
```

**编写自己的测试用例：**

1. **创建测试文件**：在 `tests/cocotb/` 目录下创建 `.py` 文件
2. **导入必要的模块**：
   ```python
   import cocotb
   import numpy as np
   from cocotb.triggers import RisingEdge, Timer
   ```
3. **定义测试函数**：使用 `@cocotb.test()` 装饰器
4. **在 BUILD 文件中注册测试**：
   ```python
   cocotb_test_suite(
       name = "my_test",
       model = ":core_mini_axi_model",
       test_module = "my_test",
       test_cases = ["my_test_case"],
   )
   ```

## 8.4 VCS 仿真

### 8.4.1 VCS 简介

VCS（Verilog Compiler Simulator）是 Synopsys 公司的商业仿真器，是业界最流行的 RTL 仿真工具之一。VCS 提供了强大的功能，包括：

- **高性能仿真**：支持多线程并行仿真
- **覆盖率分析**：代码覆盖率、功能覆盖率、断言覆盖率
- **形式验证**：支持 SystemVerilog Assertions (SVA)
- **调试工具**：集成 Verdi 波形查看器和调试器
- **UVM 支持**：完整的 UVM（Universal Verification Methodology）支持

**VCS 的使用场景：**

- 需要详细的覆盖率报告
- 需要使用 UVM 进行高级验证
- 需要形式验证和断言检查
- 需要与其他 Synopsys 工具集成（如 Design Compiler、PrimeTime）

### 8.4.2 环境配置

使用 VCS 需要先配置环境变量。VCS 是商业软件，需要有效的许可证。

**设置环境变量：**

```bash
# 设置 VCS 安装路径
export VCS_HOME=/path/to/your/vcs/installation

# 设置许可证文件
export LM_LICENSE_FILE=/path/to/license.dat

# 更新库路径和可执行文件路径
export LD_LIBRARY_PATH="${VCS_HOME}/linux64/lib:${LD_LIBRARY_PATH}"
export PATH="${VCS_HOME}/bin:${PATH}"
```

**验证安装：**

```bash
# 检查 VCS 版本
vcs -ID

# 检查许可证
vcs -check_license
```

### 8.4.3 运行 VCS 仿真

CoralNPU 使用 Bazel 集成了 VCS 仿真。默认情况下，VCS 支持是禁用的，需要使用 `--config=vcs` 标志来启用。

**运行 VCS 仿真：**

```bash
# 运行 VCS 仿真测试
bazel test --config=vcs //tests/cocotb:core_mini_axi_sim_vcs

# 运行所有 VCS 测试
bazel test --config=vcs //tests/cocotb/...
```

**VCS 构建配置：**

在 `tests/cocotb/BUILD` 文件中，可以看到 VCS 的配置：

```python
vcs_testbench_test(
    name = "core_mini_axi_vcs",
    srcs = ["core_mini_axi_sim.py"],
    module = "CoreMiniAxi",
    deps = [":core_mini_axi"],
    vcs_args = VCS_BUILD_ARGS,
    defines = VCS_DEFINES,
)
```

**VCS 编译选项：**

常用的 VCS 编译选项包括：

- `-full64`：使用 64 位模式
- `-sverilog`：启用 SystemVerilog 支持
- `-timescale=1ns/1ps`：设置时间刻度
- `-debug_access+all`：启用完整的调试信息
- `-cm line+cond+fsm+tgl+branch`：启用覆盖率收集

### 8.4.4 波形查看（Verdi）

Verdi 是 Synopsys 的波形查看和调试工具，功能比 GTKWave 更强大。

**生成 FSDB 波形文件：**

VCS 支持生成 FSDB（Fast Signal Database）格式的波形文件，这是 Verdi 的原生格式：

```bash
# 在仿真时生成 FSDB 文件
bazel test --config=vcs //tests/cocotb:core_mini_axi_sim_vcs \
    --test_arg=--fsdb_file=output.fsdb
```

**使用 Verdi 查看波形：**

```bash
# 启动 Verdi
verdi -ssf output.fsdb &

# 或者使用 Verdi 的交互模式
verdi -ssf output.fsdb -nologo
```

**Verdi 的主要功能：**

1. **信号浏览器**：层次化显示所有信号
2. **波形窗口**：显示信号波形
3. **源代码窗口**：显示 RTL 源代码，可以点击信号跳转
4. **原理图视图**：显示模块的连接关系
5. **事务调试**：支持 TLM 事务级调试
6. **断言调试**：显示 SVA 断言的状态

**Verdi 使用技巧：**

1. **快速查找信号**：按 `Ctrl+F` 打开搜索对话框
2. **添加信号到波形**：在信号浏览器中选择信号，右键 → "Add to Waveform"
3. **设置触发条件**：可以设置信号变化时的触发条件，自动定位到感兴趣的时间点
4. **比较波形**：可以加载多个波形文件进行比较
5. **导出波形**：可以将波形导出为图片或 PDF

### 8.4.5 覆盖率分析

VCS 支持多种覆盖率分析：

**1. 代码覆盖率（Code Coverage）**

- **行覆盖率（Line Coverage）**：哪些代码行被执行了
- **条件覆盖率（Condition Coverage）**：条件表达式的真假分支是否都被覆盖
- **FSM 覆盖率（FSM Coverage）**：状态机的状态和转换是否都被覆盖
- **翻转覆盖率（Toggle Coverage）**：信号是否从 0 翻转到 1，从 1 翻转到 0

**启用覆盖率收集：**

```bash
# 运行仿真并收集覆盖率
bazel test --config=vcs //tests/cocotb:core_mini_axi_sim_vcs \
    --test_arg=--coverage

# 查看覆盖率报告
urg -dir coverage.vdb
```

**2. 功能覆盖率（Functional Coverage）**

功能覆盖率需要在 SystemVerilog 代码中定义 covergroup：

```systemverilog
covergroup cg_memory_access @(posedge clk);
    address: coverpoint addr {
        bins low = {[0:1023]};
        bins mid = {[1024:2047]};
        bins high = {[2048:4095]};
    }
    access_type: coverpoint write {
        bins read = {0};
        bins write = {1};
    }
    cross address, access_type;
endgroup
```

**查看覆盖率报告：**

```bash
# 生成 HTML 格式的覆盖率报告
urg -dir coverage.vdb -format both

# 在浏览器中打开报告
firefox urgReport/dashboard.html
```

## 8.5 SystemC 仿真

### 8.5.1 SystemC/TLM 简介

SystemC 是一个基于 C++ 的系统级建模语言，用于硬件/软件协同设计和系统级验证。TLM（Transaction-Level Modeling）是 SystemC 的一个重要特性，它在更高的抽象层次上建模，不关心时钟周期级的细节，而是关注事务（Transaction）的传输。

**SystemC 的特点：**

- **高抽象层次**：比 RTL 仿真快 10-1000 倍
- **C++ 语法**：可以利用 C++ 的所有特性和库
- **系统级建模**：适合建模整个 SoC 系统
- **软硬件协同仿真**：可以将软件和硬件模型集成在一起

**TLM 的优势：**

- **快速仿真**：不需要模拟每个时钟周期
- **早期软件开发**：在 RTL 完成之前就可以开始软件开发
- **系统级验证**：可以验证系统级的行为和性能
- **IP 集成**：可以轻松集成第三方 IP 的 TLM 模型

**TLM 的抽象层次：**

1. **TLM-1**：基于 FIFO 和事件的通信
2. **TLM-2.0**：标准化的总线事务接口，支持 loosely-timed 和 approximately-timed 两种时序模型

### 8.5.2 SystemC 仿真模型

CoralNPU 的 SystemC 仿真位于 `tests/systemc/` 目录。主要包含：

- **Xbar.h**：交叉开关（Crossbar）的 TLM 模型
- **instruction_trace.cc**：指令跟踪模块
- **sysc_tb.h**：SystemC 测试台框架

**SystemC 仿真架构：**

```
┌─────────────────┐
│  Traffic Gen    │  (生成测试事务)
└────────┬────────┘
         │ TLM-2.0
         ↓
┌─────────────────┐
│  TLM2AXI Bridge │  (TLM 转 AXI)
└────────┬────────┘
         │ AXI
         ↓
┌─────────────────┐
│  CoreMiniAxi    │  (Verilator RTL 模型)
└────────┬────────┘
         │ AXI
         ↓
┌─────────────────┐
│  AXI2TLM Bridge │  (AXI 转 TLM)
└────────┬────────┘
         │ TLM-2.0
         ↓
┌─────────────────┐
│  Memory Model   │  (内存模型)
└─────────────────┘
```

这个架构的优点是：
- 可以使用 TLM 的高速内存模型
- 可以使用 TLM 的流量生成器
- 核心部分仍然使用精确的 RTL 模型

### 8.5.3 运行 SystemC 仿真

**构建 SystemC 仿真：**

```bash
# 构建 SystemC 仿真器
cd /home/curry/code/coralnpu
bazel build //tests/vcs_sim:core_mini_axi_sysc_sim
```

**运行仿真：**

```bash
# 运行 SystemC 仿真
bazel run //tests/vcs_sim:core_mini_axi_sysc_sim -- \
    --elf_file=/path/to/program.elf

# 生成波形文件（VCD 格式）
bazel run //tests/vcs_sim:core_mini_axi_sysc_sim -- \
    --elf_file=/path/to/program.elf \
    --vcd_file=output.vcd
```

**SystemC 仿真的主要代码：**

让我们看一下 `tests/vcs_sim/main.cc` 的关键部分：

```cpp
int sc_main(int argc, char** argv) {
    // 创建顶层模块
    sc_top top("top");
    
    // 创建时钟（100ns 周期 = 10MHz）
    sc_clock clock("clock", 100, SC_NS);
    
    // 创建交叉开关
    Xbar xbar("xbar");
    
    // 创建 TLM 到 AXI 的桥接
    tlm2axi_bridge<...> tlm2axi_bridge("tlm2axi_bridge");
    
    // 创建 AXI 到 TLM 的桥接
    axi2tlm_bridge<...> axi2tlm_bridge("axi2tlm_bridge");
    
    // 连接信号
    top.clock(clock);
    tlm2axi_bridge.clk(clock);
    axi2tlm_bridge.clk(clock);
    
    // 连接 AXI 信号
    top.slave_rready(axi_signals.rready);
    top.slave_rvalid(axi_signals.rvalid);
    // ... 更多信号连接
    
    // 启动仿真
    sc_start();
    
    return 0;
}
```

**关键概念解释：**

1. **`sc_main`**：SystemC 的入口函数，类似于 C++ 的 `main` 函数
2. **`sc_clock`**：SystemC 的时钟生成器，参数是周期和时间单位
3. **`sc_start()`**：启动 SystemC 仿真，会一直运行直到没有事件或调用 `sc_stop()`
4. **TLM socket**：TLM 的通信接口，类似于 C++ 的函数调用

### 8.5.4 TLM 事务

TLM-2.0 定义了标准的事务接口。一个 TLM 事务包含：

- **命令**：读（READ）或写（WRITE）
- **地址**：访问的地址
- **数据**：读写的数据
- **字节使能**：哪些字节是有效的
- **响应状态**：成功、错误等

**发起一个 TLM 事务：**

```cpp
// 创建事务对象
tlm::tlm_generic_payload trans;
trans.set_command(tlm::TLM_WRITE_COMMAND);
trans.set_address(0x1000);
trans.set_data_ptr(data_buffer);
trans.set_data_length(4);
trans.set_byte_enable_ptr(nullptr);  // 所有字节都有效
trans.set_response_status(tlm::TLM_INCOMPLETE_RESPONSE);

// 发送事务（阻塞调用）
socket->b_transport(trans, delay);

// 检查响应
if (trans.get_response_status() != tlm::TLM_OK_RESPONSE) {
    std::cerr << "Transaction failed!" << std::endl;
}
```

**TLM 的时序模型：**

1. **Loosely-Timed (LT)**：
   - 不关心精确的时序
   - 使用 `b_transport`（阻塞传输）
   - 仿真速度最快
   - 适合软件开发和功能验证

2. **Approximately-Timed (AT)**：
   - 关心大致的时序
   - 使用 `nb_transport`（非阻塞传输）
   - 仿真速度较慢，但更接近真实硬件
   - 适合性能分析

### 8.5.5 SystemC 调试

**1. 使用 `sc_trace` 生成波形：**

```cpp
// 创建 VCD 文件
sc_trace_file* tf = sc_create_vcd_trace_file("output");

// 添加要跟踪的信号
sc_trace(tf, clock, "clock");
sc_trace(tf, top.reset, "reset");
sc_trace(tf, top.io_halted, "halted");

// 运行仿真
sc_start();

// 关闭跟踪文件
sc_close_vcd_trace_file(tf);
```

**2. 使用 `SC_REPORT` 输出调试信息：**

```cpp
SC_REPORT_INFO("MyModule", "Starting simulation");
SC_REPORT_WARNING("MyModule", "Unexpected value detected");
SC_REPORT_ERROR("MyModule", "Transaction failed");
```

**3. 使用 GDB 调试：**

SystemC 程序是普通的 C++ 程序，可以使用 GDB 调试：

```bash
gdb --args bazel-bin/tests/vcs_sim/core_mini_axi_sysc_sim \
    --elf_file=test.elf

(gdb) break sc_main
(gdb) run
```

## 8.6 NPU 功能模拟器

### 8.6.1 coralnpu_sim 简介

`coralnpu_sim` 是 CoralNPU 的功能模拟器（Functional Simulator），它不模拟 RTL 的时序细节，而是直接模拟指令的功能。这种模拟器的主要优点是：

- **极快的速度**：比 RTL 仿真快 1000-10000 倍
- **易于调试**：可以直接在 Python 中调试
- **适合软件开发**：软件工程师不需要了解硬件细节
- **适合算法验证**：可以快速验证算法的正确性

**功能模拟器 vs RTL 仿真：**

| 特性 | 功能模拟器 | RTL 仿真 |
|------|-----------|---------|
| 速度 | 极快 | 慢 |
| 精度 | 功能级 | 周期精确 |
| 时序 | 不精确 | 精确 |
| 用途 | 软件开发、算法验证 | 硬件验证 |
| 调试 | 容易 | 困难 |

### 8.6.2 coralnpu_sim 架构

`coralnpu_sim` 是用 C++ 实现的，并通过 pybind11 提供 Python 接口。主要组件包括：

- **指令解码器**：解析 RISC-V 指令
- **寄存器堆**：模拟通用寄存器和 CSR
- **内存模型**：模拟 ITCM、DTCM 和外部内存
- **向量单元**：模拟 RISC-V 向量扩展（RVV）
- **执行引擎**：执行指令并更新状态

**代码位置：**

- C++ 实现：`sw/coralnpu_sim/coralnpu_v2_sim_pybind.cc`
- Python 封装：`sw/coralnpu_sim/coralnpu_v2_sim_utils.py`
- 测试用例：`sw/coralnpu_sim/coralnpu_v2_sim_test.py`

### 8.6.3 使用 coralnpu_sim

**基本用法：**

```python
from coralnpu_v2_sim_utils import CoralNPUV2Simulator
import numpy as np

# 1. 创建模拟器实例
sim = CoralNPUV2Simulator()

# 2. 加载 ELF 文件
elf_path = "path/to/your/program.elf"
entry_point, symbol_map = sim.get_elf_entry_and_symbol(
    elf_path, ["input_buffer", "output_buffer"]
)

# 3. 加载程序
sim.load_program(elf_path, entry_point)

# 4. 准备输入数据
input_data = np.arange(16, dtype=np.uint8)
input_addr = symbol_map["input_buffer"]
sim.write_memory(input_addr, input_data)

# 5. 运行模拟器
sim.run()
sim.wait()

# 6. 读取结果
output_addr = symbol_map["output_buffer"]
output_data = sim.read_memory(output_addr, 16)

# 7. 验证结果
print(f"Output: {output_data}")
print(f"Cycle count: {sim.get_cycle_count()}")
```

**高级配置：**

```python
# 配置高内存模式（更大的 ITCM 和 DTCM）
sim = CoralNPUV2Simulator(highmem_ld=True)

# 配置不在 EBREAK 时退出（用于调试）
sim = CoralNPUV2Simulator(exit_on_ebreak=False)

# 单步执行
sim.load_program(elf_path, entry_point)
for i in range(100):
    steps = sim.step(1)  # 执行一条指令
    pc = sim.read_register("pc")
    print(f"Step {i}: PC = {pc}")
```

### 8.6.4 API 参考

**CoralNPUV2Simulator 类的主要方法：**

1. **`load_program(elf_path, entry_point=None)`**
   - 加载 ELF 文件到内存
   - 如果不指定 `entry_point`，会从 ELF 文件中读取

2. **`run()`**
   - 开始运行模拟器（异步）

3. **`wait()`**
   - 等待模拟器运行完成

4. **`step(num_steps)`**
   - 单步执行指定数量的指令
   - 返回实际执行的指令数

5. **`get_cycle_count()`**
   - 获取当前的周期计数

6. **`read_memory(address, length)`**
   - 读取内存，返回 numpy 数组（uint8）

7. **`write_memory(address, data)`**
   - 写入内存，`data` 必须是 numpy 数组

8. **`read_register(name)`**
   - 读取寄存器的值（返回十六进制字符串）
   - 支持的寄存器名：`"pc"`, `"x0"` - `"x31"`, `"f0"` - `"f31"`, CSR 寄存器等

9. **`get_elf_entry_and_symbol(filename, symbol_names)`**
   - 从 ELF 文件中读取入口点和符号地址
   - 返回 `(entry_point, symbol_map)` 元组

### 8.6.5 与 RTL 仿真的区别

**功能模拟器的局限性：**

1. **不模拟时序**：
   - 不能检测时序违规（如 setup/hold time）
   - 不能评估真实的性能（如 IPC、延迟）

2. **不模拟硬件细节**：
   - 不模拟流水线停顿
   - 不模拟缓存行为
   - 不模拟总线仲裁

3. **不能发现硬件 bug**：
   - 只能验证功能正确性
   - 不能发现 RTL 实现的错误

**何时使用功能模拟器：**

- ✅ 开发和调试软件
- ✅ 验证算法正确性
- ✅ 快速迭代测试
- ✅ 生成参考结果（Golden Reference）

**何时使用 RTL 仿真：**

- ✅ 验证硬件设计
- ✅ 评估性能
- ✅ 调试硬件 bug
- ✅ 生成覆盖率报告

**最佳实践：**

1. **先用功能模拟器验证算法**：快速迭代，确保算法正确
2. **再用 RTL 仿真验证硬件**：确保硬件实现正确
3. **使用功能模拟器生成参考结果**：RTL 仿真的输出应该与功能模拟器一致

### 8.6.6 示例：完整的测试流程

让我们看一个完整的例子，展示如何使用 `coralnpu_sim` 测试一个向量加法程序：

```python
import unittest
from coralnpu_v2_sim_utils import CoralNPUV2Simulator
import numpy as np

class TestVectorAdd(unittest.TestCase):
    def setUp(self):
        """测试前的准备工作"""
        self.sim = CoralNPUV2Simulator()
        self.elf_path = "rvv_add_int8_m1.elf"
    
    def test_vector_add(self):
        """测试向量加法"""
        # 1. 加载 ELF 并获取符号地址
        entry_point, symbol_map = self.sim.get_elf_entry_and_symbol(
            self.elf_path, ["in_buf_1", "in_buf_2", "out_buf"]
        )
        
        # 2. 检查符号是否存在
        self.assertIn("in_buf_1", symbol_map)
        self.assertIn("in_buf_2", symbol_map)
        self.assertIn("out_buf", symbol_map)
        
        in_buf_1_addr = symbol_map["in_buf_1"]
        in_buf_2_addr = symbol_map["in_buf_2"]
        out_buf_addr = symbol_map["out_buf"]
        
        # 3. 加载程序
        self.sim.load_program(self.elf_path, entry_point)
        
        # 4. 准备输入数据
        input1 = np.arange(16, dtype=np.uint8)
        input2 = np.arange(16, dtype=np.uint8) * 2
        
        self.sim.write_memory(in_buf_1_addr, input1)
        self.sim.write_memory(in_buf_2_addr, input2)
        
        # 5. 运行模拟器
        self.sim.run()
        self.sim.wait()
        
        # 6. 读取结果
        output = self.sim.read_memory(out_buf_addr, 16)
        
        # 7. 验证结果
        expected = input1 + input2
        self.assertTrue(
            np.array_equal(output, expected),
            f"Expected {expected}, got {output}"
        )
        
        # 8. 检查周期数
        cycle_count = self.sim.get_cycle_count()
        print(f"Cycle count: {cycle_count}")
        self.assertGreater(cycle_count, 0)
        self.assertLess(cycle_count, 1000)  # 合理的上限

if __name__ == "__main__":
    unittest.main()
```

**运行测试：**

```bash
# 使用 Bazel 运行测试
bazel test //sw/coralnpu_sim:coralnpu_v2_sim_test

# 或者直接运行 Python 脚本
python3 sw/coralnpu_sim/coralnpu_v2_sim_test.py
```

## 8.7 仿真最佳实践

### 8.7.1 选择合适的仿真工具

根据你的需求选择合适的仿真工具：

| 需求 | 推荐工具 |
|------|---------|
| 快速验证功能 | coralnpu_sim |
| 日常开发和 CI | Verilator + Cocotb |
| 覆盖率分析 | VCS |
| 系统级验证 | SystemC/TLM |
| 性能评估 | Verilator 或 VCS |
| 软件开发 | coralnpu_sim |

### 8.7.2 编写可维护的测试

1. **使用描述性的测试名称**：
   ```python
   # 好的命名
   @cocotb.test()
   async def test_axi_write_burst_with_backpressure(dut):
       pass
   
   # 不好的命名
   @cocotb.test()
   async def test1(dut):
       pass
   ```

2. **添加注释和文档字符串**：
   ```python
   @cocotb.test()
   async def test_vector_load_store(dut):
       """测试向量加载和存储指令
       
       这个测试验证：
       1. 向量加载指令（vle8.v）可以正确加载数据
       2. 向量存储指令（vse8.v）可以正确存储数据
       3. 数据在内存中的布局是正确的
       """
       pass
   ```

3. **使用辅助函数**：
   ```python
   async def write_and_verify(core, addr, data):
       """写入数据并验证"""
       await core.write_memory(addr, data)
       readback = await core.read_memory(addr, len(data))
       assert (readback == data).all(), "Data mismatch!"
   ```

4. **参数化测试**：
   ```python
   @pytest.mark.parametrize("size", [1, 2, 4, 8, 16, 32])
   def test_different_sizes(size):
       """测试不同大小的数据传输"""
       data = np.random.randint(0, 255, size, dtype=np.uint8)
       # ... 测试逻辑
   ```

### 8.7.3 调试技巧

1. **从简单的测试开始**：
   - 先测试最基本的功能（如读写内存）
   - 再逐步增加复杂度

2. **使用断言**：
   ```python
   assert dut.signal.value == expected, \
       f"Expected {expected}, got {dut.signal.value}"
   ```

3. **打印中间结果**：
   ```python
   print(f"PC = {pc}, Instruction = {instruction:08x}")
   ```

4. **生成波形文件**：
   - 即使测试通过，也可以生成波形文件检查时序
   - 使用条件波形生成来减小文件大小

5. **使用调试器**：
   - Python 测试可以使用 `pdb` 或 `ipdb`
   - C++ 测试可以使用 `gdb`

### 8.7.4 性能优化

1. **禁用不必要的波形生成**：
   - 波形生成会显著降低仿真速度
   - 只在需要调试时才生成波形

2. **使用并行测试**：
   ```bash
   # Bazel 会自动并行运行测试
   bazel test //tests/cocotb/... --jobs=8
   ```

3. **使用增量编译**：
   - Bazel 会自动缓存编译结果
   - 只重新编译修改过的文件

4. **优化测试数据大小**：
   - 使用较小的测试数据集
   - 只测试关键的边界条件

### 8.7.5 持续集成（CI）

CoralNPU 使用 Bazel 和 CI 系统来自动运行测试：

1. **每次提交都运行测试**：
   ```bash
   # 在提交前运行所有测试
   bazel test //...
   ```

2. **使用测试标签**：
   ```python
   # 在 BUILD 文件中
   cocotb_test_suite(
       name = "quick_test",
       tags = ["quick"],
       # ...
   )
   
   cocotb_test_suite(
       name = "slow_test",
       tags = ["slow", "manual"],
       # ...
   )
   ```
   
   ```bash
   # 只运行快速测试
   bazel test //... --test_tag_filters=quick
   
   # 排除慢速测试
   bazel test //... --test_tag_filters=-slow
   ```

3. **生成测试报告**：
   ```bash
   # 生成 XML 格式的测试报告
   bazel test //... --test_output=xml
   ```

## 8.8 总结

本章介绍了 CoralNPU 项目支持的多种仿真方式：

- **Verilator**：开源、快速的 RTL 仿真器，适合日常开发
- **Cocotb**：基于 Python 的测试框架，易于编写复杂测试
- **VCS**：商业仿真器，支持覆盖率分析和高级验证
- **SystemC/TLM**：系统级建模，适合系统级验证
- **coralnpu_sim**：功能模拟器，适合软件开发和算法验证

每种仿真方式都有其适用场景，选择合适的工具可以大大提高开发效率。在实际项目中，通常会结合使用多种仿真方式：

1. 使用 **coralnpu_sim** 快速验证算法
2. 使用 **Cocotb + Verilator** 进行日常开发和 CI 测试
3. 使用 **VCS** 进行覆盖率分析和最终验证
4. 使用 **SystemC** 进行系统级验证

希望本章能帮助你理解 CoralNPU 的仿真流程，并能够有效地使用这些工具进行开发和验证。
