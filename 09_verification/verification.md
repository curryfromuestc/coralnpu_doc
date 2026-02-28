# 第九章：验证

## 9.1 验证概述

### 9.1.1 什么是验证

验证（Verification）是确保硬件设计正确实现预期功能的过程。对于处理器设计来说，验证是最重要也是最耗时的环节之一。

**为什么验证很重要？**

想象一下，你写了一个 C 程序，但没有运行测试就直接交给用户使用。程序可能会崩溃、产生错误结果，甚至损坏数据。硬件设计也是一样的，但更严重：
- 硬件一旦制造出来，就无法像软件那样轻易修改
- 芯片制造成本高昂，一个错误可能导致数百万美元的损失
- 硬件错误可能导致系统崩溃、数据损坏，甚至安全漏洞

**验证的目标：**
1. **功能正确性**：设计是否按照规范工作
2. **边界条件**：极端情况下是否正常工作
3. **性能指标**：是否满足时序和性能要求
4. **覆盖率**：是否测试了所有可能的情况

### 9.1.2 验证策略

CoralNPU 采用多层次的验证策略，从小到大、从简单到复杂：

```
┌─────────────────────────────────────────┐
│         形式化验证                       │  ← 数学证明
│    (Formal Verification)                │
├─────────────────────────────────────────┤
│         合规性测试                       │  ← RISC-V 官方测试
│    (Compliance Tests)                   │
├─────────────────────────────────────────┤
│         系统级测试                       │  ← 完整程序
│    (System-Level Tests)                 │
├─────────────────────────────────────────┤
│         集成测试                         │  ← 多模块协同
│    (Integration Tests)                  │
├─────────────────────────────────────────┤
│         单元测试                         │  ← 单个模块
│    (Unit Tests)                         │
└─────────────────────────────────────────┘
```

**1. 单元测试（Unit Tests）**
- 测试单个硬件模块（如 ALU、寄存器堆）
- 使用 ChiselTest 框架
- 快速、易于调试
- 覆盖基本功能和边界条件

**2. 集成测试（Integration Tests）**
- 测试多个模块的协同工作
- 使用 Cocotb（Python）或 UVM（SystemVerilog）
- 验证接口和数据流
- 测试总线协议（如 TileLink、AXI）

**3. 系统级测试（System-Level Tests）**
- 运行完整的程序
- 测试处理器的整体功能
- 包括中断、异常处理
- 验证软硬件协同

**4. 合规性测试（Compliance Tests）**
- 使用 RISC-V 官方测试套件
- 确保符合 RISC-V 规范
- 测试所有指令的正确性
- 验证特权级和 CSR

**5. 形式化验证（Formal Verification）**
- 使用数学方法证明设计正确性
- 验证关键属性（如不会死锁）
- 补充仿真测试的不足

### 9.1.3 验证计划

一个好的验证计划应该包括：

1. **验证目标**：明确要验证什么
2. **测试用例**：设计具体的测试场景
3. **覆盖率目标**：设定代码覆盖率和功能覆盖率目标
4. **验证环境**：选择合适的工具和框架
5. **时间表**：合理安排验证进度

**CoralNPU 的验证指标：**
- 代码覆盖率 > 90%
- 功能覆盖率 > 95%
- 通过所有 RISC-V 合规性测试
- 无已知的关键 bug

## 9.2 单元测试

### 9.2.1 ChiselTest 简介

ChiselTest 是 Chisel 的测试框架，类似于软件开发中的单元测试框架（如 Python 的 pytest 或 Java 的 JUnit）。

**ChiselTest 的特点：**
- 使用 Scala 编写测试
- 直接操作硬件信号
- 支持时钟步进和信号检查
- 易于调试和查看波形

### 9.2.2 编写单元测试

让我们通过一个实际例子来学习如何编写单元测试。以下是 CoralNPU 中 ALU（算术逻辑单元）的测试：

```scala
package coralnpu

import chisel3._
import chisel3.simulator.scalatest.ChiselSim
import org.scalatest.freespec.AnyFreeSpec

class AluSpec extends AnyFreeSpec with ChiselSim {
  val p = new Parameters

  "Initialization" in {
    simulate(new Alu(p)) { dut =>
      // 检查初始状态：输出应该无效
      dut.io.rd.valid.expect(0)
    }
  }

  "Sign Extend Byte" in {
    val test_cases = Seq(
      (0x0000007FL, 0x0000007FL),  // 正数：不变
      (0x00000080L, 0xFFFFFF80L),  // 负数：符号扩展
    )
    simulate(new Alu(p))(test_unary_op(_, 13.U, AluOp.SEXTB, test_cases))
  }
}
```

**代码解释：**

1. **测试类定义**
```scala
class AluSpec extends AnyFreeSpec with ChiselSim
```
- `AnyFreeSpec`：ScalaTest 的测试风格，允许自由组织测试
- `ChiselSim`：提供硬件仿真功能

2. **测试用例**
```scala
"Initialization" in {
  simulate(new Alu(p)) { dut =>
    dut.io.rd.valid.expect(0)
  }
}
```
- `"Initialization" in { ... }`：定义一个测试用例
- `simulate(new Alu(p))`：创建并仿真 ALU 模块
- `dut`：Device Under Test（被测设备）
- `expect(0)`：检查信号值是否为 0

3. **测试数据**
```scala
val test_cases = Seq(
  (0x0000007FL, 0x0000007FL),  // 输入 -> 期望输出
  (0x00000080L, 0xFFFFFF80L),
)
```
- 使用元组 `(输入, 期望输出)` 定义测试数据
- `L` 后缀表示 Long 类型（Scala 没有无符号整数）

### 9.2.3 测试辅助函数

为了避免重复代码，我们可以编写测试辅助函数：

```scala
private def test_unary_op(
    dut: Alu,
    addr: UInt,
    op: AluOp.Type,
    cases: Seq[(Long, Long)]) = {
  val good = cases.map { case (rs1, exp_rd) =>
    // 1. 设置输入
    dut.io.req.valid.poke(true)
    dut.io.req.bits.addr.poke(addr)
    dut.io.req.bits.op.poke(op)
    dut.io.rs1.valid.poke(true)
    dut.io.rs1.data.poke(rs1)
    
    // 2. 推进时钟
    dut.clock.step()
    
    // 3. 检查输出
    val good1 = {
      (dut.io.rd.valid.peek().litValue == 1) && 
      (dut.io.rd.bits.data.peek().litValue == exp_rd)
    }
    
    // 4. 再推进一个时钟周期，检查地址
    dut.io.req.valid.poke(true)
    dut.clock.step()
    val good2 = {
      (dut.io.rd.valid.peek().litValue == 1) && 
      (dut.io.rd.bits.addr.peek().litValue == addr.litValue)
    }
    
    good1 & good2
  }
  
  // 处理测试结果
  if (!ProcessTestResults(good, printfn = info(_))) fail()
}
```

**关键操作：**
- `poke(value)`：设置信号值（类似于软件中的赋值）
- `peek()`：读取信号值（类似于软件中的读取变量）
- `clock.step()`：推进一个时钟周期（硬件是同步的，需要时钟驱动）
- `expect(value)`：检查信号值并在不匹配时报错

### 9.2.4 运行单元测试

使用 Bazel 运行测试：

```bash
# 运行所有测试
bazel test //hdl/chisel/src/coralnpu/scalar:AluTest

# 运行特定测试
bazel test //hdl/chisel/src/coralnpu/scalar:AluTest --test_filter="Sign Extend Byte"

# 查看详细输出
bazel test //hdl/chisel/src/coralnpu/scalar:AluTest --test_output=all
```

**测试输出示例：**
```
INFO: Analyzed target //hdl/chisel/src/coralnpu/scalar:AluTest
INFO: Found 1 test target...
Target //hdl/chisel/src/coralnpu/scalar:AluTest up-to-date:
  bazel-bin/hdl/chisel/src/coralnpu/scalar/AluTest
//hdl/chisel/src/coralnpu/scalar:AluTest                        PASSED in 2.3s

Executed 1 out of 1 test: 1 test passes.
```

### 9.2.5 更多测试示例

**测试二元操作（两个输入）：**

```scala
"XNOR(Not XOR)" in {
  val test_cases = Seq(
    (0x00000000L, 0x00000000L, 0xFFFFFFFFL),  // 0 XNOR 0 = 全1
    (0x00000000L, 0x12345678L, 0xEDCBA987L),  // 0 XNOR x = NOT x
    (0x12345678L, 0x12345678L, 0xFFFFFFFFL),  // x XNOR x = 全1
  )
  simulate(new Alu(p))(testBinaryOp(_, 13.U, AluOp.XNOR, test_cases))
}
```

**测试边界条件：**

```scala
"CLZ (Count Leading Zeros)" in {
  val test_cases = Seq(
    (0L, 32L),              // 全0：32个前导零
    (1L, 31L),              // 最低位为1：31个前导零
    (0x80000000L, 0L),      // 最高位为1：0个前导零
    (0x7FFFFFFFL, 1L),      // 次高位为1：1个前导零
  )
  simulate(new Alu(p))(test_unary_op(_, 13.U, AluOp.CLZ, test_cases))
}
```

### 9.2.6 覆盖率分析

覆盖率（Coverage）衡量测试的完整性。有两种主要类型：

**1. 代码覆盖率（Code Coverage）**
- **行覆盖率**：执行了多少行代码
- **分支覆盖率**：执行了多少条件分支
- **状态覆盖率**：访问了多少状态机状态

**2. 功能覆盖率（Functional Coverage）**
- 测试了多少功能点
- 覆盖了多少输入组合
- 验证了多少边界条件

**生成覆盖率报告：**

```bash
# 使用 Verilator 生成覆盖率
bazel test //hdl/chisel/src/coralnpu/scalar:AluTest \
  --define=use_verilator=true \
  --test_arg=--coverage

# 查看覆盖率报告
genhtml coverage.info -o coverage_html
firefox coverage_html/index.html
```

**提高覆盖率的方法：**
1. 添加更多测试用例
2. 测试边界条件（0、最大值、最小值）
3. 测试错误情况
4. 测试不同的输入组合
5. 使用随机测试生成更多场景

## 9.3 集成测试

### 9.3.1 Cocotb 简介

Cocotb（Coroutine Co-simulation TestBench）是一个使用 Python 编写硬件测试的框架。

**为什么使用 Cocotb？**
- Python 比 SystemVerilog 更易学易用
- 丰富的 Python 库（NumPy、Matplotlib 等）
- 支持异步编程（async/await）
- 易于集成到 CI/CD 流程

**Cocotb 的工作原理：**
```
┌─────────────┐
│ Python 测试 │
│  (Cocotb)   │
└──────┬──────┘
       │ 通过 VPI/VHPI 接口
       ↓
┌─────────────┐
│  仿真器     │
│ (Verilator/ │
│  VCS/Xcelium)│
└──────┬──────┘
       │
       ↓
┌─────────────┐
│  DUT (RTL)  │
└─────────────┘
```

### 9.3.2 编写 Cocotb 测试

让我们看一个实际的 Cocotb 测试示例：

```python
import cocotb
from cocotb.clock import Clock
from cocotb.triggers import RisingEdge, ClockCycles

@cocotb.test()
async def test_counter(dut):
    """测试一个简单的计数器"""
    
    # 1. 启动时钟（10ns 周期 = 100MHz）
    clock = Clock(dut.clock, 10, unit="ns")
    cocotb.start_soon(clock.start())
    
    # 2. 复位
    dut.reset.value = 1
    await ClockCycles(dut.clock, 5)  # 等待 5 个时钟周期
    dut.reset.value = 0
    
    # 3. 测试计数
    for i in range(10):
        await RisingEdge(dut.clock)
        assert dut.count.value == i, f"Expected {i}, got {dut.count.value}"
    
    print("Test passed!")
```

**代码解释：**

1. **异步函数**
```python
@cocotb.test()
async def test_counter(dut):
```
- `@cocotb.test()`：标记为 Cocotb 测试
- `async`：异步函数（Python 的协程）
- `dut`：Device Under Test（被测设备）

2. **时钟生成**
```python
clock = Clock(dut.clock, 10, unit="ns")
cocotb.start_soon(clock.start())
```
- 创建一个 10ns 周期的时钟
- `start_soon()`：在后台启动时钟

3. **等待时钟**
```python
await RisingEdge(dut.clock)
```
- `await`：等待事件发生（类似于 Verilog 的 @(posedge clock)）
- `RisingEdge`：时钟上升沿

4. **信号操作**
```python
dut.reset.value = 1      # 设置信号
value = dut.count.value  # 读取信号
```

### 9.3.3 CoralNPU 的 Cocotb 测试

CoralNPU 有 87 个 Cocotb 测试文件，覆盖各种功能。让我们看几个例子：

**1. TileLink 完整性测试**

```python
@cocotb.test()
async def test_tlul_integrity(dut):
    """测试 TileLink-UL 的数据完整性保护"""
    
    # 设置
    await setup_dut(dut)
    tl_if = TileLinkULInterface(dut)
    
    # 创建一个写请求
    req = create_a_channel_req(
        opcode=TileLinkOp.PutFullData,
        address=0x1000,
        data=0xDEADBEEF,
        size=2  # 4 字节
    )
    
    # 计算完整性位（ECC）
    req["user"]["cmd_intg"] = get_cmd_intg(req)
    req["user"]["data_intg"] = get_data_intg(req["data"])
    
    # 发送请求
    await tl_if.send_a_channel(req)
    
    # 接收响应
    rsp = await tl_if.recv_d_channel()
    
    # 验证响应
    assert rsp["error"] == 0, "Transaction failed"
    assert rsp["user"]["rsp_intg"] == get_rsp_intg(rsp), "Integrity check failed"
```

**2. 性能计数器测试**

```python
@cocotb.test()
async def inst_cycle_counter_test(dut):
    """测试指令和周期计数器"""
    
    # 创建测试环境
    fixture = await Fixture.Create(dut)
    
    # 加载 ELF 程序
    elf_path = "tests/cocotb/tutorial/counters/inst_cycle_counter_example.elf"
    await fixture.load_elf_and_lookup_symbols(
        elf_path,
        ['cycle_count_lo', 'cycle_count_hi', 
         'inst_count_lo', 'inst_count_hi']
    )
    
    # 运行到停机
    await fixture.run_to_halt()
    
    # 读取结果
    cycle_count_lo = await fixture.read_word('cycle_count_lo')
    cycle_count_hi = await fixture.read_word('cycle_count_hi')
    cycle_count = (cycle_count_hi << 32) | cycle_count_lo
    
    inst_count_lo = await fixture.read_word('inst_count_lo')
    inst_count_hi = await fixture.read_word('inst_count_hi')
    instruction_count = (inst_count_hi << 32) | inst_count_lo
    
    print(f"{instruction_count} instructions executed in {cycle_count} cycles")
    print(f"CPI = {cycle_count / instruction_count:.2f}")
```

### 9.3.4 运行 Cocotb 测试

```bash
# 运行单个测试
bazel run //tests/cocotb/tutorial/counters:cocotb_counter_test

# 运行所有 TileLink 测试
bazel test //tests/cocotb/tlul/...

# 使用 VCS 仿真器
bazel test //tests/cocotb/tlul:test_tlul_integrity --define=simulator=vcs

# 生成波形
bazel test //tests/cocotb/tlul:test_tlul_integrity --test_arg=--wave
```

**查看波形：**
```bash
# 使用 GTKWave 查看波形
gtkwave dump.vcd

# 使用 Verdi 查看波形（如果有 VCS）
verdi -ssf dump.fsdb
```


### 9.3.5 测试工具和辅助函数

CoralNPU 提供了丰富的测试工具库，简化测试编写：

**1. TileLink 接口封装**

```python
from coralnpu_test_utils.TileLinkULInterface import TileLinkULInterface

# 创建接口
tl_if = TileLinkULInterface(dut)

# 读操作
data = await tl_if.read(address=0x1000, size=2)

# 写操作
await tl_if.write(address=0x1000, data=0xDEADBEEF, size=2)
```

**2. 仿真测试夹具**

```python
from coralnpu_test_utils.sim_test_fixture import Fixture

# 创建夹具
fixture = await Fixture.Create(dut)

# 加载程序
await fixture.load_elf("program.elf")

# 运行到停机
await fixture.run_to_halt(timeout_cycles=100000)

# 读取内存
data = await fixture.read_word(address=0x80000000)
```

**3. 随机测试生成**

```python
import random

@cocotb.test()
async def random_test(dut):
    """随机测试"""
    await setup_dut(dut)
    
    for _ in range(1000):
        # 生成随机地址和数据
        addr = random.randint(0, 0xFFFF) & ~0x3  # 对齐到 4 字节
        data = random.randint(0, 0xFFFFFFFF)
        
        # 写入
        await tl_if.write(addr, data)
        
        # 读回验证
        read_data = await tl_if.read(addr)
        assert read_data == data, f"Mismatch at {addr:08x}"
```

## 9.4 合规性测试

### 9.4.1 RISC-V 测试套件

RISC-V 官方提供了一套完整的测试套件（riscv-tests），用于验证处理器是否符合 RISC-V 规范。

**测试套件包含：**
- **ISA 测试**：测试每条指令的功能
- **特权级测试**：测试 M/S/U 模式切换
- **虚拟内存测试**：测试页表和 TLB
- **中断和异常测试**：测试中断处理
- **原子操作测试**：测试 A 扩展
- **浮点测试**：测试 F/D 扩展
- **向量测试**：测试 V 扩展

### 9.4.2 测试文件结构

RISC-V 测试文件通常是汇编语言编写的：

```assembly
# rv32ui-p-add.S - 测试 ADD 指令

#include "riscv_test.h"
#include "test_macros.h"

RVTEST_RV32U
RVTEST_CODE_BEGIN

  #-------------------------------------------------------------
  # 算术测试
  #-------------------------------------------------------------

  TEST_RR_OP( 2,  add, 0x00000000, 0x00000000, 0x00000000 );
  TEST_RR_OP( 3,  add, 0x00000002, 0x00000001, 0x00000001 );
  TEST_RR_OP( 4,  add, 0x0000000a, 0x00000003, 0x00000007 );

  TEST_RR_OP( 5,  add, 0xffff8000, 0x00000000, 0xffff8000 );
  TEST_RR_OP( 6,  add, 0x80000000, 0x80000000, 0x00000000 );
  TEST_RR_OP( 7,  add, 0x7ffff800, 0x80000000, 0xfffff800 );

  #-------------------------------------------------------------
  # 源/目标测试
  #-------------------------------------------------------------

  TEST_RR_SRC1_EQ_DEST( 8, add, 24, 13, 11 );
  TEST_RR_SRC2_EQ_DEST( 9, add, 25, 14, 11 );
  TEST_RR_SRC12_EQ_DEST( 10, add, 26, 13 );

  TEST_PASSFAIL

RVTEST_CODE_END
```

**测试宏解释：**

```assembly
TEST_RR_OP( 测试编号, 指令, 期望结果, 源操作数1, 源操作数2 )
```

例如：
```assembly
TEST_RR_OP( 2, add, 0x00000000, 0x00000000, 0x00000000 );
```
展开后相当于：
```assembly
  li x1, 0x00000000      # 加载源操作数1
  li x2, 0x00000000      # 加载源操作数2
  add x3, x1, x2         # 执行 ADD
  li x4, 0x00000000      # 加载期望结果
  bne x3, x4, fail       # 如果不相等，跳转到失败
```

### 9.4.3 运行 RISC-V 测试

CoralNPU 集成了 RISC-V 测试套件，可以通过 Bazel 运行：

```bash
# 运行所有 RV32I 测试
bazel test //third_party/riscv-tests:rv32ui-p-all

# 运行特定测试
bazel test //third_party/riscv-tests:rv32ui-p-add

# 运行 RV32M（乘除法）测试
bazel test //third_party/riscv-tests:rv32um-p-all

# 运行 RV32F（单精度浮点）测试
bazel test //third_party/riscv-tests:rv32uf-p-all

# 运行 RV32V（向量）测试
bazel test //third_party/riscv-tests:rv32uv-p-all
```

**测试输出示例：**
```
INFO: Running test: rv32ui-p-add
[PASS] Test 2: add 0x00000000, 0x00000000 = 0x00000000
[PASS] Test 3: add 0x00000001, 0x00000001 = 0x00000002
[PASS] Test 4: add 0x00000003, 0x00000007 = 0x0000000a
...
[PASS] All tests passed!
Test rv32ui-p-add: PASSED
```

### 9.4.4 测试覆盖范围

CoralNPU 支持以下 RISC-V 扩展的测试：

| 扩展 | 描述 | 测试数量 |
|------|------|----------|
| RV32I | 基础整数指令集 | 40+ |
| RV32M | 乘除法扩展 | 8 |
| RV32A | 原子操作扩展 | 11 |
| RV32F | 单精度浮点 | 25+ |
| RV32D | 双精度浮点 | 25+ |
| RV32C | 压缩指令 | 20+ |
| RV32V | 向量扩展 | 100+ |
| Zicsr | CSR 指令 | 5 |
| Zifencei | 指令屏障 | 2 |

### 9.4.5 调试失败的测试

如果测试失败，可以通过以下方法调试：

**1. 查看测试日志**
```bash
bazel test //third_party/riscv-tests:rv32ui-p-add --test_output=all
```

**2. 生成波形**
```bash
bazel test //third_party/riscv-tests:rv32ui-p-add \
  --test_arg=--wave \
  --test_output=all
```

**3. 使用调试器**
```bash
# 使用 GDB 调试
bazel run //third_party/riscv-tests:rv32ui-p-add_debug

# 在 GDB 中
(gdb) break main
(gdb) run
(gdb) stepi  # 单步执行指令
(gdb) info registers  # 查看寄存器
```

**4. 添加打印语句**

在 Chisel 代码中添加 printf：
```scala
when(io.commit.valid) {
  printf(p"[COMMIT] PC=${Hexadecimal(io.commit.bits.pc)} " +
         p"Inst=${Hexadecimal(io.commit.bits.inst)} " +
         p"Rd=${io.commit.bits.rd} " +
         p"Data=${Hexadecimal(io.commit.bits.data)}\n")
}
```

## 9.5 RVVI 追踪和验证

### 9.5.1 RVVI 简介

RVVI（RISC-V Verification Interface）是一个标准化的追踪接口，用于记录处理器的执行状态。

**RVVI 的作用：**
- 记录每条指令的执行
- 追踪寄存器和内存变化
- 对比不同实现的行为
- 支持形式化验证

**RVVI 追踪信息包括：**
- 程序计数器（PC）
- 指令编码
- 源寄存器值
- 目标寄存器值
- 内存访问地址和数据
- 异常和中断信息

### 9.5.2 RVVI 追踪格式

RVVI 追踪是一个结构化的数据流：

```
┌─────────────────────────────────────┐
│ RVVI Trace Entry                    │
├─────────────────────────────────────┤
│ PC:        0x80000000               │
│ Inst:      0x00000513 (addi a0,x0,0)│
│ Rd:        10 (a0)                  │
│ Rd Value:  0x00000000               │
│ Rs1:       0 (x0)                   │
│ Rs1 Value: 0x00000000               │
│ Rs2:       -                        │
│ Rs2 Value: -                        │
│ Mem Addr:  -                        │
│ Mem Data:  -                        │
│ Exception: None                     │
└─────────────────────────────────────┘
```

### 9.5.3 生成 RVVI 追踪

在 CoralNPU 中启用 RVVI 追踪：

```scala
// 在 Parameters 中启用 RVVI
val p = new Parameters {
  override val enableRVVI = true
}

// 实例化核心
val core = Module(new Core(p))

// 连接 RVVI 输出
val rvvi = core.io.rvvi
when(rvvi.valid) {
  printf(p"RVVI: PC=${Hexadecimal(rvvi.pc)} " +
         p"Inst=${Hexadecimal(rvvi.inst)} " +
         p"Rd=${rvvi.rd} " +
         p"RdData=${Hexadecimal(rvvi.rd_data)}\n")
}
```

### 9.5.4 RVVI 验证

RVVI 可以用于对比不同实现：

```python
import cocotb
from rvvi_validator import RVVIValidator

@cocotb.test()
async def rvvi_validation_test(dut):
    """使用 RVVI 验证处理器行为"""
    
    # 创建验证器
    validator = RVVIValidator()
    
    # 加载参考模型的追踪
    validator.load_reference_trace("reference.rvvi")
    
    # 运行 DUT 并收集追踪
    fixture = await Fixture.Create(dut)
    await fixture.load_elf("test.elf")
    
    while not fixture.halted():
        await fixture.step()
        
        # 获取 RVVI 追踪
        if dut.rvvi_valid.value:
            trace = {
                'pc': dut.rvvi_pc.value,
                'inst': dut.rvvi_inst.value,
                'rd': dut.rvvi_rd.value,
                'rd_data': dut.rvvi_rd_data.value,
            }
            
            # 验证
            validator.check(trace)
    
    # 报告结果
    if validator.all_passed():
        print("RVVI validation passed!")
    else:
        print(f"RVVI validation failed: {validator.errors}")
```

### 9.5.5 使用 RVVI 调试

RVVI 追踪对调试非常有用：

**1. 对比执行轨迹**
```bash
# 生成 CoralNPU 的追踪
bazel run //tests:generate_rvvi_trace -- test.elf > coralnpu.rvvi

# 生成参考模型（如 Spike）的追踪
spike --isa=rv32imafdc test.elf > spike.rvvi

# 对比
diff coralnpu.rvvi spike.rvvi
```

**2. 定位错误指令**
```bash
# 找到第一个不匹配的指令
diff -u coralnpu.rvvi spike.rvvi | head -20
```

输出示例：
```diff
--- spike.rvvi
+++ coralnpu.rvvi
@@ -1234,7 +1234,7 @@
 PC: 0x80001000  Inst: 0x00a50533  Rd: 10  Data: 0x00000005
 PC: 0x80001004  Inst: 0x00b50533  Rd: 10  Data: 0x00000006
-PC: 0x80001008  Inst: 0x00c50533  Rd: 10  Data: 0x00000007
+PC: 0x80001008  Inst: 0x00c50533  Rd: 10  Data: 0x00000008
```

这表明在 PC=0x80001008 处，CoralNPU 计算出的结果是 0x8，而正确结果应该是 0x7。

## 9.6 UVM 验证环境

### 9.6.1 UVM 简介

UVM（Universal Verification Methodology）是一个标准化的验证方法学，使用 SystemVerilog 编写。

**UVM 的特点：**
- 结构化的验证环境
- 可重用的验证组件
- 支持约束随机测试
- 功能覆盖率收集
- 工业标准

**UVM 环境结构：**
```
┌─────────────────────────────────────────┐
│              UVM Test                   │
│         (测试场景定义)                   │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│           UVM Environment               │
│         (验证环境容器)                   │
│  ┌──────────────────────────────────┐  │
│  │      Scoreboard (记分板)          │  │
│  │      (检查结果正确性)              │  │
│  └──────────────────────────────────┘  │
│  ┌──────────────┐  ┌──────────────┐   │
│  │ AXI Master   │  │ AXI Slave    │   │
│  │   Agent      │  │   Agent      │   │
│  └──────┬───────┘  └───────┬──────┘   │
└─────────┼──────────────────┼───────────┘
          │                  │
          ▼                  ▼
     ┌────────────────────────────┐
     │          DUT               │
     │    (被测设计)               │
     └────────────────────────────┘
```

### 9.6.2 CoralNPU 的 UVM 环境

CoralNPU 提供了一个基础的 UVM 测试环境，位于 `tests/uvm/` 目录。

**目录结构：**
```
tests/uvm/
├── common/                    # 公共组件
│   ├── coralnpu_axi_master/  # AXI Master Agent
│   ├── coralnpu_axi_slave/   # AXI Slave Agent
│   ├── coralnpu_irq/         # 中断接口
│   └── transaction_item/     # 事务定义
├── env/                      # UVM 环境
│   └── coralnpu_env_pkg.sv
├── tb/                       # 顶层测试平台
│   └── coralnpu_tb_top.sv
├── tests/                    # 测试用例
│   └── coralnpu_test_pkg.sv
├── Makefile                  # 编译和运行脚本
└── README.md                 # 使用说明
```

### 9.6.3 运行 UVM 测试

**1. 编译**
```bash
cd tests/uvm
make compile
```

这会：
- 生成 DUT 的 Verilog
- 编译 UVM 测试平台
- 创建仿真可执行文件

**2. 运行测试**
```bash
# 运行默认测试
make run

# 运行特定测试
make run UVM_TESTNAME=coralnpu_random_test

# 指定测试程序
make run TEST_ELF=/path/to/program.elf

# 设置详细级别
make run UVM_VERBOSITY=UVM_HIGH
```

**3. 查看结果**
```bash
# 查看日志
cat sim_work/logs/coralnpu_base_test.log

# 查看波形
verdi -ssf sim_work/waves/coralnpu_base_test.fsdb
```

### 9.6.4 编写 UVM 测试

一个简单的 UVM 测试示例：

```systemverilog
class coralnpu_simple_test extends coralnpu_base_test;
  `uvm_component_utils(coralnpu_simple_test)

  function new(string name = "coralnpu_simple_test", uvm_component parent = null);
    super.new(name, parent);
  endfunction

  virtual task run_phase(uvm_phase phase);
    coralnpu_axi_sequence seq;
    
    phase.raise_objection(this);
    
    // 创建序列
    seq = coralnpu_axi_sequence::type_id::create("seq");
    
    // 配置序列
    seq.num_transactions = 100;
    seq.randomize();
    
    // 启动序列
    seq.start(env.axi_master_agent.sequencer);
    
    // 等待完成
    #10000ns;
    
    phase.drop_objection(this);
  endtask

endclass
```

### 9.6.5 UVM 的优势和局限

**优势：**
- 工业标准，广泛使用
- 强大的约束随机测试
- 完善的覆盖率收集
- 可重用的验证 IP

**局限：**
- 学习曲线陡峭
- SystemVerilog 不如 Python 灵活
- 需要商业仿真器（VCS、Xcelium）
- 仿真速度较慢

**何时使用 UVM：**
- 大型复杂设计
- 需要约束随机测试
- 工业级验证要求
- 团队熟悉 UVM

**何时使用 Cocotb：**
- 快速原型验证
- 算法验证
- 需要 Python 生态
- 开源工具链


## 9.7 形式化验证

### 9.7.1 形式化验证概述

形式化验证（Formal Verification）使用数学方法证明设计的正确性，而不是通过运行测试用例。

**形式化验证 vs 仿真测试：**

| 特性 | 仿真测试 | 形式化验证 |
|------|----------|------------|
| 方法 | 运行测试用例 | 数学证明 |
| 覆盖率 | 有限的测试场景 | 所有可能的情况 |
| 速度 | 较慢（需要运行很多测试） | 较快（对于小规模设计） |
| 适用范围 | 任何规模的设计 | 小到中等规模 |
| 结果 | 发现 bug | 证明正确或找到反例 |

**形式化验证的类型：**

1. **等价性检查（Equivalence Checking）**
   - 证明两个设计在功能上等价
   - 常用于验证优化后的设计

2. **模型检查（Model Checking）**
   - 验证设计满足特定属性
   - 例如：不会死锁、总是响应请求

3. **定理证明（Theorem Proving）**
   - 使用逻辑推理证明设计正确性
   - 需要人工参与

### 9.7.2 断言（Assertions）

断言是形式化验证的基础，用于描述设计应该满足的属性。

**SystemVerilog 断言（SVA）：**

```systemverilog
// 立即断言：在当前时刻检查
always_comb begin
  assert (valid -> ready) 
    else $error("Valid without ready!");
end

// 并发断言：检查时序关系
property req_ack;
  @(posedge clk) req |-> ##[1:5] ack;
endproperty

assert property (req_ack)
  else $error("Request not acknowledged within 5 cycles!");

// 覆盖率断言：检查是否发生
cover property (@(posedge clk) req && ack);
```

**断言示例解释：**

1. **立即断言**
```systemverilog
assert (valid -> ready)
```
- `->` 表示"蕴含"（implication）
- 意思是：如果 valid 为真，则 ready 也必须为真
- 类似于 C 语言的 `assert()`

2. **时序断言**
```systemverilog
req |-> ##[1:5] ack
```
- `|->` 表示"时序蕴含"
- `##[1:5]` 表示"在 1 到 5 个时钟周期后"
- 意思是：如果 req 为真，则在 1-5 个周期后 ack 必须为真

3. **覆盖率断言**
```systemverilog
cover property (req && ack)
```
- 检查 req 和 ack 同时为真的情况是否发生过
- 用于确保测试覆盖了这个场景

### 9.7.3 CoralNPU 中的断言

CoralNPU 在关键模块中使用断言来捕获设计错误：

**1. 流水线断言**

```scala
// 在 Chisel 中使用 assert
when(io.commit.valid) {
  // 检查 PC 对齐
  assert(io.commit.bits.pc(1, 0) === 0.U, 
         "PC must be 4-byte aligned")
  
  // 检查寄存器编号
  assert(io.commit.bits.rd < 32.U, 
         "Register number must be < 32")
  
  // 检查不写入 x0
  when(io.commit.bits.rd === 0.U) {
    assert(!io.commit.bits.rd_wen, 
           "Cannot write to x0")
  }
}
```

**2. 总线协议断言**

```scala
// TileLink-UL 协议检查
when(io.tl.a.valid) {
  // 检查地址对齐
  assert(io.tl.a.bits.address(1, 0) === 0.U, 
         "Address must be aligned")
  
  // 检查 size 字段
  assert(io.tl.a.bits.size <= log2Ceil(dataWidth/8).U,
         "Size exceeds bus width")
}

// 检查握手协议
assert(!(io.tl.a.valid && !io.tl.a.ready && io.tl.a.valid_prev),
       "Valid must remain stable until ready")
```

**3. 内存一致性断言**

```scala
// 检查写后读一致性
when(io.mem_write.valid && io.mem_read.valid) {
  when(io.mem_write.bits.addr === io.mem_read.bits.addr) {
    assert(io.mem_read.bits.data === io.mem_write.bits.data,
           "Read-after-write must return written data")
  }
}
```

### 9.7.4 使用形式化验证工具

**1. JasperGold（Cadence）**

```bash
# 启动 JasperGold
jg &

# 在 GUI 中：
# 1. 加载设计
# 2. 添加断言
# 3. 运行形式化验证
# 4. 查看结果和反例
```

**2. SymbiYosys（开源）**

```bash
# 安装 SymbiYosys
pip install symbiyosys

# 创建配置文件 config.sby
cat > config.sby << 'EOFSBY'
[tasks]
prove
cover

[options]
prove: mode prove
cover: mode cover
depth 20

[engines]
smtbmc

[script]
read_verilog design.v
prep -top top_module

[files]
design.v
EOFSBY

# 运行验证
sby -f config.sby
```

**3. 在 Chisel 中使用形式化验证**

```scala
import chisel3.formal._

class FormalTest extends Module {
  val io = IO(new Bundle {
    val a = Input(UInt(8.W))
    val b = Input(UInt(8.W))
    val sum = Output(UInt(8.W))
  })
  
  io.sum := io.a + io.b
  
  // 形式化验证属性
  when(past(io.a) + past(io.b) < 256.U) {
    assert(io.sum === past(io.a) + past(io.b))
  }
}
```

### 9.7.5 形式化验证的最佳实践

**1. 从简单开始**
- 先验证小模块
- 逐步增加复杂度
- 不要一开始就验证整个处理器

**2. 明确属性**
- 清楚地定义要验证的属性
- 使用自然语言描述，然后转换为断言
- 例如："每个请求必须在 10 个周期内得到响应"

**3. 使用假设（Assumptions）**
- 约束输入空间
- 排除不可能的情况
- 加速验证过程

```systemverilog
// 假设：地址总是对齐的
assume property (@(posedge clk) addr[1:0] == 2'b00);

// 在这个假设下验证
assert property (@(posedge clk) valid |-> ready);
```

**4. 分析反例**
- 形式化工具会生成反例（counterexample）
- 仔细分析反例，判断是设计错误还是断言错误
- 修复后重新验证

**5. 结合仿真**
- 形式化验证不能替代仿真
- 两者互补：形式化验证证明属性，仿真验证功能
- 使用形式化验证发现的反例作为仿真测试用例

## 9.8 覆盖率分析

### 9.8.1 覆盖率的重要性

覆盖率（Coverage）衡量测试的完整性。高覆盖率意味着：
- 测试了更多的代码路径
- 发现 bug 的概率更高
- 对设计质量更有信心

**覆盖率目标：**
- 代码覆盖率：> 90%
- 功能覆盖率：> 95%
- 关键路径：100%

### 9.8.2 代码覆盖率

代码覆盖率衡量测试执行了多少代码。

**1. 行覆盖率（Line Coverage）**
- 执行了多少行代码
- 最基本的覆盖率指标

```scala
val result = if (condition) {
  a + b  // 这行被执行了吗？
} else {
  a - b  // 这行被执行了吗？
}
```

**2. 分支覆盖率（Branch Coverage）**
- 每个条件的真假分支都执行了吗

```scala
when(valid && ready) {  // 需要测试 4 种情况：
  // 00, 01, 10, 11
}
```

**3. 条件覆盖率（Condition Coverage）**
- 每个布尔子表达式的真假都测试了吗

```scala
if (a > 0 && b < 10) {
  // 需要测试：
  // a > 0 为真和假
  // b < 10 为真和假
}
```

**4. 状态覆盖率（FSM Coverage）**
- 访问了所有状态吗
- 执行了所有状态转换吗

```scala
val state = RegInit(sIdle)
switch(state) {
  is(sIdle) { ... }    // 访问了吗？
  is(sBusy) { ... }    // 访问了吗？
  is(sDone) { ... }    // 访问了吗？
}
```

### 9.8.3 收集代码覆盖率

**使用 Verilator 收集覆盖率：**

```bash
# 1. 编译时启用覆盖率
bazel test //tests/cocotb:test_name \
  --define=use_verilator=true \
  --test_arg=--coverage

# 2. 运行测试（自动收集覆盖率）

# 3. 生成报告
verilator_coverage --annotate coverage_html coverage.dat

# 4. 查看报告
firefox coverage_html/index.html
```

**使用 VCS 收集覆盖率：**

```bash
# 1. 编译时启用覆盖率
vcs -cm line+cond+fsm+tgl+branch \
    -cm_dir coverage.vdb \
    design.v testbench.v

# 2. 运行仿真
./simv -cm line+cond+fsm+tgl+branch

# 3. 生成报告
urg -dir coverage.vdb

# 4. 查看报告
firefox urgReport/dashboard.html
```

### 9.8.4 功能覆盖率

功能覆盖率衡量测试了多少功能点，而不仅仅是代码行。

**定义覆盖点（Coverpoints）：**

```systemverilog
covergroup alu_cg @(posedge clk);
  // 操作码覆盖
  opcode: coverpoint alu_op {
    bins add = {ALU_ADD};
    bins sub = {ALU_SUB};
    bins and_op = {ALU_AND};
    bins or_op = {ALU_OR};
    bins xor_op = {ALU_XOR};
  }
  
  // 操作数覆盖
  operand_a: coverpoint rs1_data {
    bins zero = {0};
    bins small = {[1:100]};
    bins large = {[101:$]};
  }
  
  // 交叉覆盖：操作码 × 操作数
  cross opcode, operand_a;
endgroup
```

**在 Cocotb 中收集功能覆盖率：**

```python
from cocotb_coverage.coverage import CoverPoint, CoverCross

# 定义覆盖点
@CoverPoint("alu.opcode", 
            bins=["ADD", "SUB", "AND", "OR", "XOR"])
@CoverPoint("alu.operand_a",
            bins=[0, (1, 100), (101, 0xFFFFFFFF)])
@CoverCross("alu.cross", 
            items=["alu.opcode", "alu.operand_a"])
async def alu_test(dut, opcode, operand_a, operand_b):
    """ALU 测试"""
    dut.opcode.value = opcode
    dut.rs1_data.value = operand_a
    dut.rs2_data.value = operand_b
    await RisingEdge(dut.clk)
    # 检查结果...

# 运行测试
for op in ["ADD", "SUB", "AND", "OR", "XOR"]:
    for a in [0, 50, 200, 0xFFFFFFFF]:
        for b in [0, 1, 100]:
            await alu_test(dut, op, a, b)

# 生成覆盖率报告
coverage_db.report_coverage()
```

### 9.8.5 提高覆盖率

**1. 分析未覆盖的代码**

```bash
# 查看覆盖率报告，找到未覆盖的行
# 例如：
# File: alu.v
# Line 123: if (overflow) begin  // 0% covered
#             handle_overflow();
#           end
```

**2. 添加针对性测试**

```python
@cocotb.test()
async def test_overflow(dut):
    """测试溢出情况"""
    # 设置会导致溢出的输入
    dut.rs1_data.value = 0x7FFFFFFF  # 最大正数
    dut.rs2_data.value = 1
    dut.opcode.value = ALU_ADD
    await RisingEdge(dut.clk)
    
    # 检查溢出标志
    assert dut.overflow.value == 1
```

**3. 使用约束随机测试**

```python
import random

@cocotb.test()
async def random_test(dut):
    """随机测试"""
    for _ in range(10000):
        # 随机生成输入
        opcode = random.choice([ALU_ADD, ALU_SUB, ALU_AND, ALU_OR])
        rs1 = random.randint(0, 0xFFFFFFFF)
        rs2 = random.randint(0, 0xFFFFFFFF)
        
        # 执行测试
        await alu_test(dut, opcode, rs1, rs2)
```

**4. 覆盖边界条件**

```python
# 边界值测试
boundary_values = [
    0,           # 零
    1,           # 最小正数
    0x7FFFFFFF,  # 最大正数（有符号）
    0x80000000,  # 最小负数（有符号）
    0xFFFFFFFF,  # 全1
]

for a in boundary_values:
    for b in boundary_values:
        await alu_test(dut, ALU_ADD, a, b)
```

### 9.8.6 覆盖率报告解读

**典型的覆盖率报告：**

```
Coverage Summary:
==================
Line Coverage:      92.5% (3700/4000 lines)
Branch Coverage:    88.3% (530/600 branches)
Condition Coverage: 85.0% (340/400 conditions)
FSM Coverage:       95.0% (38/40 states)

Uncovered Lines:
----------------
File: core.v
  Line 234: if (debug_mode) begin
  Line 456: case (exception_code)
            8'h0B: // Reserved exception

File: fpu.v
  Line 89: if (denormal_input) begin

Uncovered Branches:
-------------------
File: alu.v
  Line 123: if (overflow) begin
    True branch: NOT COVERED
    False branch: covered
```

**如何解读：**
- **高覆盖率（> 90%）**：测试较完整
- **中等覆盖率（70-90%）**：需要添加更多测试
- **低覆盖率（< 70%）**：测试不足，存在风险

**未覆盖代码的常见原因：**
1. **错误处理代码**：很少触发
2. **调试代码**：正常运行时不执行
3. **保留功能**：未实现的功能
4. **死代码**：永远不会执行的代码（应该删除）

## 9.9 持续集成（CI）中的验证

### 9.9.1 自动化测试

将验证集成到 CI/CD 流程中，确保每次代码变更都经过测试。

**GitHub Actions 示例：**

```yaml
name: CoralNPU Verification

on: [push, pull_request]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      
      - name: Setup Bazel
        uses: bazelbuild/setup-bazelisk@v1
      
      - name: Run Unit Tests
        run: |
          bazel test //hdl/chisel/src/...
      
      - name: Upload Test Results
        uses: actions/upload-artifact@v2
        with:
          name: test-results
          path: bazel-testlogs/

  integration-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      
      - name: Run Cocotb Tests
        run: |
          bazel test //tests/cocotb/...
      
      - name: Generate Coverage Report
        run: |
          bazel coverage //tests/cocotb/...
          genhtml -o coverage_html bazel-out/_coverage/_coverage_report.dat
      
      - name: Upload Coverage
        uses: codecov/codecov-action@v2
        with:
          files: ./coverage.xml

  riscv-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      
      - name: Run RISC-V Compliance Tests
        run: |
          bazel test //third_party/riscv-tests/...
```

### 9.9.2 回归测试

回归测试（Regression Testing）确保新代码不会破坏现有功能。

**回归测试策略：**
1. **每次提交都运行快速测试**（< 5 分钟）
2. **每天运行完整测试套件**（夜间构建）
3. **每周运行长时间测试**（压力测试、随机测试）

**测试分级：**
```bash
# 快速测试（冒烟测试）
bazel test //tests:smoke_tests

# 标准测试
bazel test //tests:standard_tests

# 完整测试
bazel test //...

# 长时间测试
bazel test //tests:stress_tests --test_timeout=3600
```

### 9.9.3 测试报告和通知

**生成测试报告：**

```bash
# 生成 JUnit 格式的报告
bazel test //... --test_output=xml

# 生成 HTML 报告
python scripts/generate_test_report.py \
  --input bazel-testlogs \
  --output test_report.html
```

**发送通知：**

```yaml
# 在 GitHub Actions 中发送 Slack 通知
- name: Notify Slack
  if: failure()
  uses: 8398a7/action-slack@v3
  with:
    status: ${{ job.status }}
    text: 'CoralNPU tests failed!'
    webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

## 9.10 验证最佳实践

### 9.10.1 测试金字塔

遵循测试金字塔原则：

```
        ┌─────────┐
        │ 系统测试 │  ← 少量，慢，高价值
        ├─────────┤
        │ 集成测试 │  ← 中等数量
        ├─────────┤
        │ 单元测试 │  ← 大量，快，低成本
        └─────────┘
```

- **大量单元测试**：快速、易于调试
- **适量集成测试**：验证模块协同
- **少量系统测试**：验证整体功能

### 9.10.2 测试驱动开发（TDD）

先写测试，再写代码：

1. **写一个失败的测试**
2. **写最少的代码让测试通过**
3. **重构代码**
4. **重复**

### 9.10.3 代码审查

- 审查测试代码和设计代码
- 确保测试覆盖了关键场景
- 检查测试的可读性和可维护性

### 9.10.4 文档化

- 记录验证计划
- 文档化测试用例
- 记录已知问题和限制

## 9.11 总结

验证是硬件设计中最重要的环节之一。CoralNPU 采用多层次的验证策略：

1. **单元测试**：使用 ChiselTest 测试单个模块
2. **集成测试**：使用 Cocotb 和 UVM 测试模块协同
3. **合规性测试**：使用 RISC-V 测试套件验证指令集
4. **形式化验证**：使用断言和形式化工具证明属性
5. **覆盖率分析**：确保测试的完整性

**关键要点：**
- 验证要尽早开始，贯穿整个开发过程
- 使用多种验证方法互补
- 追求高覆盖率，但不要盲目追求 100%
- 自动化测试，集成到 CI/CD
- 持续改进验证流程

**下一步：**
- 阅读第 10 章：FPGA 实现
- 学习如何在 FPGA 上运行和测试 CoralNPU
- 了解硬件调试技术
