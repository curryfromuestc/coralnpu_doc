# 6.2 RTL 设计

本节介绍 CoralNPU 项目的 RTL（Register Transfer Level，寄存器传输级）设计流程，包括从 Chisel 生成 Verilog、综合、时序分析、仿真和验证等内容。

## 6.2.1 Verilog 生成

### 什么是 RTL？

RTL（寄存器传输级）是一种硬件描述方式，它描述了数据如何在寄存器之间传输和处理。可以把它理解为硬件的"中间表示"：
- 比门级（Gate Level）更抽象，不需要关心具体用哪些门电路
- 比行为级（Behavioral Level）更具体，明确了寄存器和时钟的使用
- 是综合工具的输入，可以被转换成实际的硬件电路

在 CoralNPU 项目中，我们使用 Chisel（一种基于 Scala 的硬件构造语言）编写硬件设计，然后通过工具链将其转换为 Verilog RTL 代码。

### Chisel 到 Verilog 的转换流程

CoralNPU 使用 CIRCT/FIRRTL 工具链将 Chisel 代码转换为 Verilog：

```
Chisel (Scala) → FIRRTL (中间表示) → Verilog (RTL)
```

**工具链说明：**
- **Chisel**：高层次硬件构造语言，基于 Scala
- **FIRRTL**（Flexible Intermediate Representation for RTL）：一种中间表示格式，类似于软件编译器中的 IR
- **firtool**：CIRCT 项目提供的工具，用于将 FIRRTL 转换为 Verilog
- **CIRCT**（Circuit IR Compilers and Tools）：基于 LLVM/MLIR 的硬件编译器基础设施

### 生成 Verilog 的方法

在 CoralNPU 项目中，Verilog 生成是通过 Bazel 构建系统自动完成的。

#### 1. 使用 chisel_cc_library 规则

在 `rules/chisel.bzl` 中定义了 `chisel_cc_library` 规则，它会：
1. 编译 Chisel 代码为 Scala 可执行文件
2. 运行该可执行文件生成 FIRRTL
3. 使用 firtool 将 FIRRTL 转换为 Verilog
4. 将生成的 Verilog 封装为 Verilator C++ 库

示例（来自 `rules/chisel.bzl`）：

```python
def chisel_cc_library(
        name,
        chisel_lib,           # Chisel 库依赖
        emit_class,           # 生成 Verilog 的主类
        module_name,          # 顶层模块名称
        verilog_deps = [],    # Verilog 依赖
        vopts = [],           # Verilator 选项
        gen_flags = []):      # 生成标志
    # 创建生成 Verilog 的二进制文件
    gen_binary_name = name + "_emit_verilog_binary"
    chisel_binary(
        name = gen_binary_name,
        deps = [chisel_lib],
        main_class = emit_class,
    )
    
    # 生成 Verilog 文件
    native.genrule(
        name = name + "_emit_verilog",
        outs = [module_name + ".sv"],
        cmd = "CHISEL_FIRTOOL_PATH=... ./$(location " + gen_binary_name + ") --target-dir=$(RULEDIR)",
        tools = [gen_binary_name, "firtool"],
    )
```

#### 2. 手动生成 Verilog

如果需要手动生成 Verilog（用于调试或查看生成的代码），可以：

```bash
# 1. 构建 Chisel 生成器
bazel build //hdl/chisel/src/coralnpu:Core_emit_verilog_binary

# 2. 运行生成器
CHISEL_FIRTOOL_PATH=third_party/llvm-firtool \
  bazel-bin/hdl/chisel/src/coralnpu/Core_emit_verilog_binary \
  --target-dir=./output
```

这会在 `./output` 目录下生成 Verilog 文件。

### 生成的 Verilog 文件位置

通过 Bazel 构建系统生成的 Verilog 文件位于：

```
bazel-bin/hdl/chisel/src/<模块路径>/<模块名>.sv
```

例如：
- `bazel-bin/hdl/chisel/src/coralnpu/Core.sv` - 核心模块
- `bazel-bin/hdl/chisel/src/bus/TlulSocket1N.sv` - 总线模块

**注意：** 这些是自动生成的文件，不应该手动编辑。如果需要修改，应该修改源 Chisel 代码。

### FIRRTL 编译选项

firtool 支持多种编译选项来控制生成的 Verilog 质量：

```bash
# 常用选项
--split-verilog              # 将大模块拆分为多个文件
--lowering-options=...       # 控制降低（lowering）过程
--disable-all-randomization  # 禁用随机初始化（用于综合）
--strip-debug-info           # 移除调试信息
```

在 CoralNPU 项目中，这些选项通过 `gen_flags` 参数传递给 `chisel_cc_library`。

## 6.2.2 综合

### 什么是综合？

综合（Synthesis）是将 RTL 代码转换为门级网表（Gate-level Netlist）的过程。可以理解为：
- **输入**：Verilog RTL 代码（描述功能和时序）
- **输出**：门级网表（由标准单元库中的门电路组成）
- **目标**：在满足时序、面积、功耗约束的前提下，生成最优的硬件实现

类比软件开发：综合就像编译器的优化过程，将高级代码转换为高效的机器码。

### 综合流程

典型的综合流程包括以下步骤：

```
RTL Verilog → 解析 → 优化 → 映射 → 门级网表
                ↓       ↓      ↓
              约束    时序    面积
```

1. **解析（Elaboration）**：读取 RTL 代码，构建设计的内部表示
2. **优化（Optimization）**：进行逻辑优化，简化电路
3. **映射（Mapping）**：将逻辑映射到标准单元库中的门电路
4. **优化迭代**：根据时序和面积约束进行多轮优化

### 综合工具

CoralNPU 项目支持多种综合工具：

#### Synopsys Design Compiler

Design Compiler 是业界标准的综合工具。基本使用流程：

```tcl
# 1. 读取标准单元库
set target_library "your_tech_lib.db"
set link_library "* $target_library"

# 2. 读取 RTL 代码
read_verilog Core.sv

# 3. 设置顶层模块
current_design Core

# 4. 链接设计
link

# 5. 设置约束（见下文）
source constraints.sdc

# 6. 编译（综合）
compile_ultra

# 7. 生成报告
report_timing
report_area
report_power

# 8. 输出网表
write -format verilog -output Core_synth.v
```

#### 开源综合工具

对于 FPGA 或开源流程，可以使用：
- **Yosys**：开源综合工具
- **Vivado Synthesis**：Xilinx FPGA 综合工具
- **Quartus**：Intel FPGA 综合工具

### 综合约束

综合约束（Constraints）告诉综合工具设计的性能要求。主要包括：

#### 1. 时序约束

时序约束定义了时钟和时序路径的要求：

```tcl
# 创建时钟（100MHz，周期 10ns）
create_clock -name clk -period 10 [get_ports clk]

# 设置输入延迟（相对于时钟）
set_input_delay -clock clk -max 2.0 [all_inputs]

# 设置输出延迟
set_output_delay -clock clk -max 2.0 [all_outputs]

# 设置时钟不确定性（jitter）
set_clock_uncertainty 0.5 [get_clocks clk]

# 设置时钟转换时间
set_clock_transition 0.1 [get_clocks clk]
```

**解释：**
- `create_clock`：定义时钟信号，周期 10ns 表示频率 100MHz
- `set_input_delay`：输入信号相对于时钟边沿的延迟，这里是 2ns
- `set_clock_uncertainty`：时钟的不确定性（抖动），这里是 0.5ns

#### 2. 面积约束

```tcl
# 设置最大面积（单位：门数量或 um²）
set_max_area 50000

# 设置最大动态功耗
set_max_dynamic_power 100
```

#### 3. 环境约束

```tcl
# 设置工作条件（温度、电压等）
set_operating_conditions -max slow_corner -min fast_corner

# 设置负载
set_load 0.1 [all_outputs]

# 设置驱动强度
set_driving_cell -lib_cell BUFX2 [all_inputs]
```

### 面积和时序优化

综合工具会在面积和时序之间进行权衡：

#### 面积优化策略

```tcl
# 使用面积优先的编译策略
compile_ultra -area_high_effort_script

# 允许更长的编译时间以获得更小的面积
set_app_var compile_ultra_ungroup_dw false
```

#### 时序优化策略

```tcl
# 使用时序优先的编译策略
compile_ultra -timing_high_effort_script

# 允许面积增加以满足时序
set_max_area 0  # 0 表示不限制面积
```

#### 多目标优化

```tcl
# 平衡面积和时序
compile_ultra -gate_clock -no_autoungroup

# 设置时序裕量（margin）
set_clock_uncertainty 0.5 [get_clocks clk]
```

### 综合结果分析

综合完成后，需要检查以下报告：

```tcl
# 时序报告（最关键）
report_timing -max_paths 10 -nworst 1

# 面积报告
report_area -hierarchy

# 功耗报告
report_power -hierarchy

# 约束检查
report_constraint -all_violators
```

**关键指标：**
- **Slack**：时序裕量，正值表示满足时序，负值表示违例
- **Total Area**：总面积，包括组合逻辑和时序逻辑
- **Dynamic Power**：动态功耗，与翻转率相关


## 6.2.3 时序分析

### 什么是时序分析？

时序分析（Timing Analysis）是验证数字电路能否在指定时钟频率下正确工作的过程。

**基本概念：**
- **建立时间（Setup Time）**：数据在时钟边沿到来之前必须稳定的时间
- **保持时间（Hold Time）**：数据在时钟边沿之后必须保持稳定的时间
- **时钟周期（Clock Period）**：两个连续时钟边沿之间的时间
- **传播延迟（Propagation Delay）**：信号通过组合逻辑的延迟

**时序违例：**
- **Setup Violation**：数据到达太晚，在时钟边沿前没有足够的建立时间
- **Hold Violation**：数据变化太快，在时钟边沿后没有足够的保持时间

### 静态时序分析（STA）

静态时序分析（Static Timing Analysis, STA）是一种不需要仿真就能验证时序的方法。

#### STA 基本原理

STA 分析所有可能的路径，计算：

```
路径延迟 = 发射寄存器时钟延迟 + 组合逻辑延迟 + 捕获寄存器建立时间
```

**时序裕量（Slack）计算：**

```
Slack = 要求时间 - 到达时间
```

- Slack > 0：满足时序要求
- Slack = 0：刚好满足时序
- Slack < 0：时序违例

#### 使用 PrimeTime 进行 STA

PrimeTime 是 Synopsys 的静态时序分析工具：

```tcl
# 1. 读取网表和库
read_verilog Core_synth.v
link_design Core

# 2. 读取时序约束
read_sdc constraints.sdc

# 3. 读取寄生参数（RC 延迟）
read_parasitics Core.spef

# 4. 更新时序
update_timing

# 5. 报告时序
report_timing -delay_type max -max_paths 10
report_timing -delay_type min -max_paths 10

# 6. 检查约束
check_timing
report_constraint -all_violators
```

### 时序约束

时序约束使用 SDC（Synopsys Design Constraints）格式定义。

#### 基本时序约束

```tcl
# 1. 定义时钟
create_clock -name clk -period 10.0 [get_ports clk]

# 2. 定义虚拟时钟（用于 I/O 约束）
create_clock -name vclk -period 10.0

# 3. 设置时钟组（异步时钟）
set_clock_groups -asynchronous \
  -group [get_clocks clk_a] \
  -group [get_clocks clk_b]

# 4. 设置时钟延迟
set_clock_latency -source 1.0 [get_clocks clk]
set_clock_latency 0.5 [get_clocks clk]

# 5. 设置时钟不确定性
set_clock_uncertainty -setup 0.5 [get_clocks clk]
set_clock_uncertainty -hold 0.2 [get_clocks clk]
```

#### I/O 时序约束

```tcl
# 输入延迟约束
set_input_delay -clock clk -max 3.0 [get_ports data_in]
set_input_delay -clock clk -min 1.0 [get_ports data_in]

# 输出延迟约束
set_output_delay -clock clk -max 2.0 [get_ports data_out]
set_output_delay -clock clk -min 0.5 [get_ports data_out]

# 输入转换时间
set_input_transition 0.2 [get_ports data_in]

# 输出负载
set_load 0.1 [get_ports data_out]
```

#### 多周期路径

有些路径可能需要多个时钟周期才能完成：

```tcl
# 设置多周期路径（2 个时钟周期）
set_multicycle_path -setup 2 -from [get_pins reg_a/Q] -to [get_pins reg_b/D]
set_multicycle_path -hold 1 -from [get_pins reg_a/Q] -to [get_pins reg_b/D]
```

#### 假路径

某些路径在实际使用中不会被触发，可以标记为假路径：

```tcl
# 设置假路径（不进行时序检查）
set_false_path -from [get_ports test_mode] -to [all_registers]
```

### 关键路径分析

关键路径（Critical Path）是决定最大工作频率的路径。

#### 识别关键路径

```tcl
# 报告最慢的 10 条路径
report_timing -delay_type max -max_paths 10 -nworst 1

# 报告特定路径
report_timing -from [get_pins reg_a/Q] -to [get_pins reg_b/D]

# 报告所有违例路径
report_timing -slack_lesser_than 0.0
```

#### 关键路径报告解读

典型的时序报告格式：

```
Startpoint: reg_a (rising edge-triggered flip-flop clocked by clk)
Endpoint: reg_b (rising edge-triggered flip-flop clocked by clk)
Path Group: clk
Path Type: max

Point                                    Incr       Path
---------------------------------------------------------
clock clk (rise edge)                    0.00       0.00
clock network delay (ideal)              0.50       0.50
reg_a/CK (DFFX1)                         0.00       0.50 r
reg_a/Q (DFFX1)                          0.30       0.80 r
U1/A (AND2X1)                            0.00       0.80 r
U1/Y (AND2X1)                            0.25       1.05 r
U2/A (OR2X1)                             0.00       1.05 r
U2/Y (OR2X1)                             0.30       1.35 r
reg_b/D (DFFX1)                          0.00       1.35 r
data arrival time                                   1.35

clock clk (rise edge)                   10.00      10.00
clock network delay (ideal)              0.50      10.50
clock uncertainty                       -0.50      10.00
reg_b/CK (DFFX1)                         0.00      10.00 r
library setup time                      -0.15       9.85
data required time                                  9.85
---------------------------------------------------------
data required time                                  9.85
data arrival time                                  -1.35
---------------------------------------------------------
slack (MET)                                         8.50
```

**解读：**
- **Incr**：增量延迟（这一级的延迟）
- **Path**：累计延迟（从起点到当前点的总延迟）
- **r/f**：上升沿（rising）或下降沿（falling）
- **Slack = 8.50**：时序裕量为 8.5ns，满足时序要求

### 时序收敛策略

当设计存在时序违例时，需要采取措施进行时序收敛（Timing Closure）。

#### 1. RTL 级优化

```scala
// 不好的写法：组合逻辑链太长
val result = a + b + c + d + e + f

// 好的写法：插入流水线寄存器
val stage1 = RegNext(a + b + c)
val stage2 = RegNext(stage1 + d + e + f)
```

#### 2. 综合优化

```tcl
# 增加综合努力程度
compile_ultra -timing_high_effort_script

# 允许寄存器复制以减少扇出
set_app_var compile_register_replication true

# 允许边界优化
set_boundary_optimization true
```

#### 3. 物理综合

```tcl
# 使用物理综合（考虑布局布线延迟）
compile_ultra -gate_clock -spg

# 设置布局约束
set_app_var physopt_enable_via_res_support true
```

#### 4. 时钟树优化

```tcl
# 优化时钟树以减少时钟偏斜
set_clock_tree_options -target_skew 0.1

# 使用专用时钟缓冲器
set_dont_touch [get_cells clk_buf*]
```

#### 5. 降低时钟频率

如果其他方法都无效，可能需要降低目标频率：

```tcl
# 从 100MHz 降低到 90MHz
create_clock -name clk -period 11.11 [get_ports clk]
```

### 时序分析最佳实践

1. **早期时序规划**：在 RTL 设计阶段就考虑时序
2. **合理的流水线**：在关键路径上插入寄存器
3. **避免组合逻辑环**：确保所有反馈路径都经过寄存器
4. **时钟域交叉处理**：使用同步器处理异步时钟域
5. **约束完整性**：确保所有路径都有时序约束

## 6.2.4 RTL 仿真

### 什么是 RTL 仿真？

RTL 仿真（RTL Simulation）是在软件环境中模拟硬件行为的过程。与实际硬件相比：
- **优点**：快速迭代、易于调试、可以观察内部信号
- **缺点**：速度较慢（通常比实际硬件慢 1000-10000 倍）

### Verilator 仿真

Verilator 是一个开源的 Verilog 仿真器，它将 Verilog 代码转换为 C++ 代码，然后编译为可执行文件。

#### Verilator 的特点

- **高性能**：比传统事件驱动仿真器快 10-100 倍
- **周期精确**：只支持同步设计，不支持延迟和事件
- **开源免费**：适合 CI/CD 集成

#### 在 CoralNPU 中使用 Verilator

CoralNPU 项目通过 Bazel 集成了 Verilator。在 `rules/chisel.bzl` 中：

```python
# 生成 Verilator C++ 库
verilator_cc_library(
    name = "Core_cc",
    module = ":Core_verilog",
    module_top = "Core",
    vopts = ["--pins-bv", "2"],
    systemc = False,
)
```

#### 编写 Verilator 测试

```cpp
#include "VCore.h"  // Verilator 生成的头文件
#include "verilated.h"

int main(int argc, char** argv) {
    Verilated::commandArgs(argc, argv);
    
    // 创建模块实例
    VCore* core = new VCore;
    
    // 复位
    core->reset = 1;
    core->clk = 0;
    core->eval();
    core->clk = 1;
    core->eval();
    core->reset = 0;
    
    // 运行仿真
    for (int i = 0; i < 1000; i++) {
        // 时钟下降沿
        core->clk = 0;
        core->eval();
        
        // 时钟上升沿
        core->clk = 1;
        core->eval();
        
        // 检查输出
        printf("Cycle %d: output = %d\n", i, core->output);
    }
    
    // 清理
    delete core;
    return 0;
}
```

#### Verilator 命令行选项

```bash
# 基本用法
verilator --cc Core.sv --exe testbench.cpp

# 常用选项
verilator \
  --cc                    # 生成 C++ 代码
  --exe                   # 生成可执行文件
  --build                 # 自动编译
  --trace                 # 生成波形文件
  --trace-fst             # 使用 FST 格式（更快）
  --top-module Core       # 指定顶层模块
  -Wall                   # 启用所有警告
  --assert                # 启用断言检查
  Core.sv testbench.cpp
```

### VCS 仿真

VCS（Verilog Compiler Simulator）是 Synopsys 的商业仿真器，支持完整的 SystemVerilog 特性。

#### VCS 的特点

- **完整的语言支持**：支持 SystemVerilog、UVM、SystemC
- **高级调试**：支持断点、观察点、代码覆盖率
- **混合仿真**：支持 Verilog + SystemC 混合仿真

#### 在 CoralNPU 中使用 VCS

CoralNPU 项目在 `rules/vcs.bzl` 中定义了 VCS 仿真规则：

```python
# VCS 测试规则
vcs_testbench_test(
    name = "core_test",
    deps = ":Core_verilog",
    module = "Core",
)
```

#### VCS 编译和运行

```bash
# 1. 编译 Verilog 代码
vcs -full64 -sverilog Core.sv testbench.sv -o simv

# 2. 运行仿真
./simv

# 3. 使用 GUI 调试
./simv -gui

# 4. 生成波形
./simv -ucli -do "dump -file waves.fsdb -type FSDB"
```

#### VCS 常用选项

```bash
vcs \
  -full64              # 64 位模式
  -sverilog            # SystemVerilog 模式
  -debug_access+all    # 启用完整调试
  -kdb                 # 生成 KDB 数据库（用于调试）
  +vcs+fsdbon          # 启用 FSDB 波形
  -timescale=1ns/1ps   # 设置时间精度
  -cm line+tgl+fsm     # 启用代码覆盖率
  -o simv              # 输出文件名
  Core.sv testbench.sv
```

### 波形查看

波形（Waveform）是仿真过程中信号随时间变化的图形表示，是调试硬件的重要工具。

#### 生成波形文件

**Verilator（VCD 格式）：**

```cpp
#include "verilated_vcd_c.h"

int main() {
    Verilated::traceEverOn(true);
    VerilatedVcdC* tfp = new VerilatedVcdC;
    
    VCore* core = new VCore;
    core->trace(tfp, 99);  // 跟踪深度
    tfp->open("waves.vcd");
    
    // 仿真循环
    for (int i = 0; i < 1000; i++) {
        core->clk = 0;
        core->eval();
        tfp->dump(2*i);      // 记录波形
        
        core->clk = 1;
        core->eval();
        tfp->dump(2*i + 1);
    }
    
    tfp->close();
    delete core;
}
```

**VCS（FSDB 格式）：**

```systemverilog
initial begin
    $fsdbDumpfile("waves.fsdb");
    $fsdbDumpvars(0, testbench);
end
```

#### 波形查看工具

1. **GTKWave**（开源，支持 VCD/FST）

```bash
# 安装
sudo apt-get install gtkwave

# 打开波形
gtkwave waves.vcd
```

2. **Verdi**（商业，支持 FSDB）

```bash
# 打开波形
verdi -ssf waves.fsdb
```

3. **DVE**（VCS 自带）

```bash
# 打开波形
dve -full64 -vpd waves.vpd
```

#### 波形调试技巧

1. **添加关键信号**：只添加需要观察的信号，避免波形文件过大
2. **使用标记**：在关键时刻添加标记（marker）
3. **信号分组**：将相关信号分组显示
4. **使用触发器**：设置触发条件，只记录感兴趣的时间段

```systemverilog
// 条件触发
initial begin
    $fsdbDumpfile("waves.fsdb");
    wait (error_flag == 1);  // 等待错误发生
    $fsdbDumpvars(0, testbench);
    #1000;
    $finish;
end
```


## 6.2.5 RTL 验证

### 什么是 RTL 验证？

RTL 验证（RTL Verification）是确保硬件设计功能正确的过程。与软件测试类似，但硬件验证更加复杂：
- **并行性**：硬件是并行执行的，需要考虑所有可能的时序组合
- **不可修改性**：芯片流片后无法修改，必须在设计阶段发现所有问题
- **验证成本**：验证通常占整个项目工作量的 60-70%

### 功能验证

功能验证（Functional Verification）确保设计实现了规范要求的功能。

#### 验证方法学

1. **定向测试（Directed Test）**
   - 手动编写测试用例
   - 针对特定功能或边界条件
   - 适合简单模块和回归测试

```systemverilog
// 定向测试示例
initial begin
    // 测试加法
    a = 10;
    b = 20;
    #10;
    assert(sum == 30) else $error("Add failed");
    
    // 测试溢出
    a = 32'hFFFFFFFF;
    b = 1;
    #10;
    assert(sum == 0) else $error("Overflow failed");
end
```

2. **随机测试（Random Test）**
   - 使用随机激励
   - 覆盖更多场景
   - 需要自检机制（scoreboard）

```systemverilog
// 随机测试示例
initial begin
    repeat(1000) begin
        a = $random;
        b = $random;
        #10;
        // 使用参考模型检查
        check_result(a, b, sum);
    end
end
```

3. **约束随机验证（Constrained Random Verification）**
   - SystemVerilog 的高级特性
   - 在约束范围内生成随机激励
   - 平衡覆盖率和效率

```systemverilog
class transaction;
    rand bit [31:0] addr;
    rand bit [31:0] data;
    
    // 约束：地址必须对齐
    constraint addr_align {
        addr[1:0] == 2'b00;
    }
    
    // 约束：地址范围
    constraint addr_range {
        addr >= 32'h1000;
        addr <= 32'h2000;
    }
endclass
```

#### 验证环境

典型的验证环境包括：

```
┌─────────────────────────────────────────┐
│          Verification Environment        │
│                                          │
│  ┌──────────┐      ┌──────────────┐    │
│  │ Driver   │─────>│     DUT      │    │
│  └──────────┘      │  (设计)      │    │
│                    └──────────────┘    │
│                           │             │
│                           v             │
│  ┌──────────┐      ┌──────────────┐    │
│  │ Monitor  │<─────│   Interface  │    │
│  └──────────┘      └──────────────┘    │
│       │                                 │
│       v                                 │
│  ┌──────────┐      ┌──────────────┐    │
│  │Scoreboard│<─────│Reference Model│    │
│  └──────────┘      └──────────────┘    │
└─────────────────────────────────────────┘
```

- **Driver**：生成激励，驱动 DUT 输入
- **Monitor**：观察 DUT 输出
- **Scoreboard**：比较实际输出和期望输出
- **Reference Model**：功能正确的参考实现

#### 使用 Chisel 进行验证

CoralNPU 使用 ChiselTest 框架进行单元测试：

```scala
import chiseltest._
import org.scalatest.flatspec.AnyFlatSpec

class AluTest extends AnyFlatSpec with ChiselScalatestTester {
  "Alu" should "perform addition" in {
    test(new Alu) { dut =>
      // 设置输入
      dut.io.op.poke(AluOp.Add)
      dut.io.a.poke(10.U)
      dut.io.b.poke(20.U)
      
      // 推进时钟
      dut.clock.step()
      
      // 检查输出
      dut.io.result.expect(30.U)
    }
  }
  
  "Alu" should "handle overflow" in {
    test(new Alu) { dut =>
      dut.io.op.poke(AluOp.Add)
      dut.io.a.poke("hFFFFFFFF".U)
      dut.io.b.poke(1.U)
      dut.clock.step()
      dut.io.result.expect(0.U)
    }
  }
}
```

#### 运行 Chisel 测试

```bash
# 运行所有测试
bazel test //hdl/chisel/src/coralnpu/scalar:AluTest

# 运行特定测试
bazel test //hdl/chisel/src/coralnpu/scalar:AluTest --test_filter="Alu should perform addition"

# 生成波形
bazel test //hdl/chisel/src/coralnpu/scalar:AluTest --test_arg=--with-waveform
```

### 代码覆盖率

代码覆盖率（Code Coverage）衡量测试执行了多少代码，帮助发现未测试的部分。

#### 覆盖率类型

1. **行覆盖率（Line Coverage）**
   - 每一行代码是否被执行
   - 最基本的覆盖率指标

2. **分支覆盖率（Branch Coverage）**
   - 每个条件分支是否都被执行
   - 包括 if-else、case 等

```systemverilog
// 分支覆盖示例
if (condition) begin
    // 分支 1：condition = true
    result = a;
end else begin
    // 分支 2：condition = false
    result = b;
end
// 需要两个测试用例才能达到 100% 分支覆盖
```

3. **条件覆盖率（Condition Coverage）**
   - 每个布尔子表达式的真假值是否都被测试

```systemverilog
// 条件覆盖示例
if (a && b) begin
    result = 1;
end
// 需要测试：
// - a=0, b=0
// - a=0, b=1
// - a=1, b=0
// - a=1, b=1
```

4. **翻转覆盖率（Toggle Coverage）**
   - 每个信号是否经历了 0→1 和 1→0 的翻转
   - 用于检测未使用的信号

5. **FSM 覆盖率（FSM Coverage）**
   - 状态机的每个状态是否都被访问
   - 每个状态转换是否都被执行

#### 使用 VCS 收集覆盖率

```bash
# 1. 编译时启用覆盖率
vcs -full64 -sverilog \
  -cm line+tgl+fsm+cond+branch \  # 启用多种覆盖率
  -cm_dir coverage.vdb \           # 覆盖率数据库
  Core.sv testbench.sv

# 2. 运行仿真
./simv -cm line+tgl+fsm+cond+branch

# 3. 查看覆盖率报告
urg -dir coverage.vdb

# 4. 使用 GUI 查看
verdi -cov -covdir coverage.vdb
```

#### 覆盖率报告解读

典型的覆盖率报告：

```
Module: Core
  Line Coverage:      95.2%  (1234/1296 lines)
  Branch Coverage:    88.7%  (456/514 branches)
  Condition Coverage: 82.3%  (234/284 conditions)
  Toggle Coverage:    91.5%  (1832/2000 bits)
  FSM Coverage:       100.0% (12/12 states)
```

**目标：**
- 行覆盖率：> 95%
- 分支覆盖率：> 90%
- FSM 覆盖率：100%

**注意：** 100% 覆盖率不等于没有 bug，只是说明代码都被执行过。

#### 提高覆盖率的方法

1. **分析未覆盖的代码**

```bash
# 查看未覆盖的行
urg -dir coverage.vdb -show uncovered
```

2. **添加针对性测试**

```systemverilog
// 针对未覆盖的错误处理分支
initial begin
    // 触发错误条件
    invalid_input = 1;
    #10;
    assert(error_flag == 1);
end
```

3. **排除不可达代码**

```systemverilog
// 使用覆盖率指令排除
// synopsys coverage off
if (SIMULATION_ONLY) begin
    $display("Debug info");
end
// synopsys coverage on
```

### 断言（Assertions）

断言（Assertions）是嵌入在 RTL 代码中的检查，用于验证设计的属性。

#### SystemVerilog 断言（SVA）

SystemVerilog 提供了强大的断言语言：

```systemverilog
// 立即断言（Immediate Assertion）
always @(posedge clk) begin
    assert (valid -> ready)
        else $error("Valid without ready");
end

// 并发断言（Concurrent Assertion）
property req_ack;
    @(posedge clk) req |-> ##[1:3] ack;
endproperty

assert property (req_ack)
    else $error("Request not acknowledged");
```

#### 常用断言模式

1. **握手协议**

```systemverilog
// valid-ready 握手
property valid_ready_handshake;
    @(posedge clk) valid && !ready |=> valid;
endproperty
assert property (valid_ready_handshake);
```

2. **互斥条件**

```systemverilog
// read 和 write 不能同时为 1
property read_write_mutex;
    @(posedge clk) !(read && write);
endproperty
assert property (read_write_mutex);
```

3. **稳定性**

```systemverilog
// valid 为 1 时，data 必须保持稳定
property data_stable;
    @(posedge clk) valid && !ready |=> $stable(data);
endproperty
assert property (data_stable);
```

4. **最终性**

```systemverilog
// 每个请求最终都会得到响应
property req_eventually_ack;
    @(posedge clk) req |-> ##[1:$] ack;
endproperty
assert property (req_eventually_ack);
```

#### 在 Chisel 中使用断言

Chisel 提供了 `assert`、`assume` 和 `cover` 语句：

```scala
// 断言：检查条件必须为真
assert(io.valid || !io.ready, "Ready without valid")

// 假设：告诉形式验证工具的前提条件
assume(io.addr < 1024.U, "Address out of range")

// 覆盖点：标记感兴趣的场景
cover(io.error, "Error occurred")
```

#### 断言的好处

1. **早期发现问题**：在仿真时立即检测到违例
2. **文档化设计意图**：断言描述了设计的预期行为
3. **形式验证**：断言可以用于形式验证工具
4. **回归测试**：确保修改不会破坏已有的属性

### 验证最佳实践

1. **分层验证**
   - 单元测试：验证单个模块
   - 集成测试：验证模块间的交互
   - 系统测试：验证整个系统

2. **自动化**
   - 使用 CI/CD 自动运行测试
   - 每次提交都运行回归测试

```bash
# 在 CI 中运行所有测试
bazel test //...
```

3. **覆盖率驱动**
   - 设定覆盖率目标（如 95%）
   - 定期检查覆盖率报告
   - 针对未覆盖的代码添加测试

4. **使用断言**
   - 在关键接口添加断言
   - 检查协议的正确性
   - 验证不变量（invariants）

5. **参考模型**
   - 使用高层次模型（如 Python、C++）作为参考
   - 自动比较 RTL 和参考模型的输出

```python
# Python 参考模型示例
def alu_reference(op, a, b):
    if op == "ADD":
        return (a + b) & 0xFFFFFFFF
    elif op == "SUB":
        return (a - b) & 0xFFFFFFFF
    # ...
```

6. **代码审查**
   - 审查 RTL 代码和测试代码
   - 检查边界条件和错误处理
   - 确保测试覆盖了所有功能

## 6.2.6 总结

本节介绍了 CoralNPU 项目的 RTL 设计流程：

1. **Verilog 生成**：使用 Chisel 和 CIRCT/FIRRTL 工具链生成 Verilog RTL 代码
2. **综合**：使用 Design Compiler 等工具将 RTL 转换为门级网表，并进行面积和时序优化
3. **时序分析**：使用静态时序分析（STA）验证设计能否在目标频率下工作
4. **RTL 仿真**：使用 Verilator 或 VCS 进行功能仿真和调试
5. **RTL 验证**：通过功能验证、代码覆盖率和断言确保设计的正确性

### 关键要点

- **RTL 是硬件设计的核心**：它是综合、仿真、验证的基础
- **时序是关键**：必须满足时序约束才能正确工作
- **验证很重要**：验证占项目工作量的大部分，必须充分测试
- **工具链集成**：CoralNPU 使用 Bazel 集成了整个工具链，简化了构建流程

### 下一步

- 第 6.3 节将介绍外设设计
- 第 6.4 节将介绍 SoC 集成
- 第 9 章将深入介绍验证方法学

### 参考资源

- Chisel 官方文档：https://www.chisel-lang.org/
- CIRCT 项目：https://circt.llvm.org/
- Verilator 文档：https://verilator.org/guide/latest/
- SystemVerilog LRM：IEEE 1800-2017
- "Writing Testbenches" by Janick Bergeron
- "SystemVerilog for Verification" by Chris Spear

