# 第十五章：高级主题

本章介绍 CoralNPU 的高级特性和优化技术，包括自定义指令扩展、性能调优、低功耗设计和安全特性。这些内容面向需要深入定制和优化 CoralNPU 的高级用户。

## 15.1 自定义指令

CoralNPU 基于 RISC-V 架构，支持添加自定义指令来扩展处理器功能。本节介绍如何设计和实现自定义指令。

### 15.1.1 指令编码空间

RISC-V 规范为自定义指令预留了编码空间。CoralNPU 使用以下编码空间：

**自定义指令编码格式：**

```
31                                    7    6    0
+-------------------------------------+----+----+
|           funct7/imm                | rd | op |
+-------------------------------------+----+----+
```

- `op[6:0]`: 操作码字段
  - `0001011` (custom-0): 自定义指令空间 0
  - `0101011` (custom-1): 自定义指令空间 1
  - `1011011` (custom-2): 自定义指令空间 2
  - `1111011` (custom-3): 自定义指令空间 3

**注意：** CoralNPU 回收了 C 扩展（压缩指令）的编码空间用于 SIMD 寄存器索引（6 位）和向量指令。

### 15.1.2 添加自定义指令的步骤

#### 步骤 1：定义指令编码

首先在 `Decode.scala` 中定义新指令的位模式。例如，添加一个自定义的"计数前导零"指令：

```scala
// 在 DecodedInstruction 类中添加新字段
val myCustomOp = Bool()

// 在 DecodeInstruction 对象中添加解码逻辑
d.myCustomOp := op === BitPat("b0110000_00000_?????_001_?????_0001011")
```

**解释：**
- `BitPat` 是 Chisel 中用于匹配位模式的类型
- `?????` 表示这些位可以是任意值（通常是寄存器地址）
- `0001011` 是 custom-0 操作码

#### 步骤 2：在执行单元中实现功能

在相应的执行单元（如 ALU）中添加新指令的实现：

```scala
// 在 Alu.scala 中
object AluOp extends ChiselEnum {
  val ADD, SUB, XOR, OR, AND = Value
  // ... 其他操作
  val MY_CUSTOM_OP = Value  // 添加新操作
}

// 在 ALU 的执行逻辑中
val result = MuxCase(0.U, Seq(
  (io.op === AluOp.ADD) -> (rs1 + rs2),
  (io.op === AluOp.SUB) -> (rs1 - rs2),
  // ... 其他操作
  (io.op === AluOp.MY_CUSTOM_OP) -> myCustomLogic(rs1, rs2)
))
```

#### 步骤 3：连接调度逻辑

在 `Dispatch.scala` 中将新指令路由到正确的执行单元：

```scala
// 在 DispatchV2 类的 try-dispatch 循环中
val alu = MuxUpTo1H(MakeValid(false.B, AluOp.ADD), Seq(
  // ... 现有映射
  d.myCustomOp -> MakeValid(true.B, AluOp.MY_CUSTOM_OP),
))
```

#### 步骤 4：更新未定义指令检测

确保新指令不会被标记为未定义：

```scala
// 在 DecodeInstruction 对象的末尾
val decoded = Cat(
  d.lui, d.auipc,
  // ... 所有现有指令
  d.myCustomOp,  // 添加新指令
  d.rvv.map(_.valid).getOrElse(false.B),
  d.float.map(_.valid).getOrElse(false.B)
)

d.undef := decoded === 0.U
```

### 15.1.3 软件支持

为了在 C/C++ 代码中使用自定义指令，需要定义内联汇编宏：

```c
// custom_instructions.h
#ifndef CUSTOM_INSTRUCTIONS_H
#define CUSTOM_INSTRUCTIONS_H

#include <stdint.h>

// 自定义指令：my_custom_op rd, rs1, rs2
// 编码：0110000 rs2 rs1 001 rd 0001011
static inline uint32_t my_custom_op(uint32_t rs1, uint32_t rs2) {
    uint32_t rd;
    asm volatile (
        ".insn r 0x0B, 0x1, 0x30, %0, %1, %2"
        : "=r"(rd)
        : "r"(rs1), "r"(rs2)
    );
    return rd;
}

#endif // CUSTOM_INSTRUCTIONS_H
```

**使用示例：**

```c
#include "custom_instructions.h"

int main() {
    uint32_t a = 0x12345678;
    uint32_t b = 0xABCDEF00;
    uint32_t result = my_custom_op(a, b);
    return 0;
}
```

### 15.1.4 验证自定义指令

创建测试用例验证新指令：

```c
// test_custom_instruction.c
#include <stdint.h>
#include "custom_instructions.h"

uint32_t test_data[8] __attribute__((section(".data")));
uint32_t results[8] __attribute__((section(".data")));

int main() {
    // 初始化测试数据
    for (int i = 0; i < 8; i++) {
        test_data[i] = i * 0x11111111;
    }
    
    // 执行自定义指令
    for (int i = 0; i < 8; i++) {
        results[i] = my_custom_op(test_data[i], 0x12345678);
    }
    
    return 0;
}
```

## 15.2 性能调优

CoralNPU 提供了多种性能优化技术，本节介绍如何充分利用硬件特性提升程序性能。

### 15.2.1 指令级并行（ILP）

CoralNPU 是一个 4 路超标量处理器，每个周期最多可以调度 4 条指令。要充分利用这一特性：

**优化前（串行代码）：**

```c
int sum = 0;
for (int i = 0; i < 100; i++) {
    sum += array[i];  // 每次循环只有一条有效指令
}
```

**优化后（展开循环）：**

```c
int sum0 = 0, sum1 = 0, sum2 = 0, sum3 = 0;
for (int i = 0; i < 100; i += 4) {
    sum0 += array[i];      // 这 4 条指令可以
    sum1 += array[i+1];    // 在同一周期
    sum2 += array[i+2];    // 并行调度
    sum3 += array[i+3];
}
int sum = sum0 + sum1 + sum2 + sum3;
```

**关键点：**
- 避免数据依赖：确保指令之间没有 RAW（读后写）冲突
- 循环展开：减少分支指令，增加可并行指令数量
- 使用独立的累加器：避免累加器成为瓶颈

### 15.2.2 流水线优化

CoralNPU 的流水线包括取指、译码/调度、执行/写回三个阶段。不同指令的延迟如下：

| 指令类型 | 延迟（周期） | 说明 |
|---------|------------|------|
| ALU     | 1          | 加减、逻辑运算 |
| 分支    | 1          | 条件分支、跳转 |
| 乘法    | 2          | mul, mulh 等 |
| 除法    | 可变       | div, rem 等 |
| 访存    | 2+         | load, store |

**优化技巧：**

1. **隐藏访存延迟：**

```c
// 不好的做法：访存后立即使用
int a = array[i];
int b = a * 2;  // 等待访存完成

// 好的做法：在访存和使用之间插入其他指令
int a = array[i];
int c = x + y;      // 独立计算
int d = z * 2;      // 独立计算
int b = a * 2;      // 此时访存可能已完成
```

2. **避免分支预测失败：**

CoralNPU 使用简单的静态分支预测：
- 向后分支（循环）：预测为taken
- 向前分支：预测为not-taken

```c
// 不好：频繁的不可预测分支
for (int i = 0; i < n; i++) {
    if (array[i] > threshold) {  // 难以预测
        result[i] = process(array[i]);
    }
}

// 好：使用条件移动避免分支
for (int i = 0; i < n; i++) {
    int temp = process(array[i]);
    int cond = (array[i] > threshold);
    result[i] = cond ? temp : 0;  // 编译器可能生成条件移动指令
}
```

### 15.2.3 内存访问优化

#### Cache 友好的访问模式

CoralNPU 的 L1 数据缓存配置：
- 大小：16KB（双 bank，每个 8KB）
- 行大小：256 位（32 字节）
- 关联度：4 路组相联

**优化策略：**

1. **顺序访问：**

```c
// 好：顺序访问，充分利用 cache line
for (int i = 0; i < N; i++) {
    sum += array[i];
}

// 不好：跨步访问，cache 利用率低
for (int i = 0; i < N; i += 8) {
    sum += array[i];
}
```

2. **数据对齐：**

```c
// 确保数据结构对齐到 cache line
struct __attribute__((aligned(32))) AlignedData {
    int data[8];  // 正好一个 cache line
};
```

3. **阻塞（Blocking）技术：**

```c
// 矩阵乘法：分块访问以提高 cache 命中率
#define BLOCK_SIZE 8
for (int ii = 0; ii < N; ii += BLOCK_SIZE) {
    for (int jj = 0; jj < N; jj += BLOCK_SIZE) {
        for (int kk = 0; kk < N; kk += BLOCK_SIZE) {
            // 在小块内进行计算
            for (int i = ii; i < ii + BLOCK_SIZE; i++) {
                for (int j = jj; j < jj + BLOCK_SIZE; j++) {
                    for (int k = kk; k < kk + BLOCK_SIZE; k++) {
                        C[i][j] += A[i][k] * B[k][j];
                    }
                }
            }
        }
    }
}
```

#### TCM 使用

对于频繁访问的小数据，使用 TCM（紧耦合内存）可以获得确定性的低延迟：

```c
// 将关键数据放在 DTCM
uint32_t critical_data[256] __attribute__((section(".data")));

// DTCM 访问延迟固定，适合实时任务
for (int i = 0; i < 256; i++) {
    critical_data[i] = process(i);
}
```

### 15.2.4 向量化优化

CoralNPU 支持 RISC-V 向量扩展（RVV），向量寄存器宽度为 128 位。

**标量代码：**

```c
void add_arrays(int32_t *a, int32_t *b, int32_t *c, int n) {
    for (int i = 0; i < n; i++) {
        c[i] = a[i] + b[i];
    }
}
```

**向量化代码：**

```c
#include <riscv_vector.h>

void add_arrays_vec(int32_t *a, int32_t *b, int32_t *c, int n) {
    size_t vl;
    for (size_t i = 0; i < n; i += vl) {
        vl = vsetvl_e32m1(n - i);  // 设置向量长度
        vint32m1_t va = vle32_v_i32m1(a + i, vl);  // 加载向量
        vint32m1_t vb = vle32_v_i32m1(b + i, vl);
        vint32m1_t vc = vadd_vv_i32m1(va, vb, vl);  // 向量加法
        vse32_v_i32m1(c + i, vc, vl);  // 存储向量
    }
}
```

**性能提升：**
- 128 位向量寄存器可以同时处理 4 个 32 位整数
- 理论加速比：4x（实际取决于访存带宽）

### 15.2.5 编译器优化选项

使用合适的编译器选项可以显著提升性能：

```bash
# 基本优化
riscv32-unknown-elf-gcc -O2 -march=rv32im -mabi=ilp32 program.c

# 激进优化
riscv32-unknown-elf-gcc -O3 -march=rv32im -mabi=ilp32 \
    -funroll-loops \          # 循环展开
    -finline-functions \      # 函数内联
    -ffast-math \             # 快速数学运算
    program.c

# 启用向量扩展
riscv32-unknown-elf-gcc -O3 -march=rv32imv -mabi=ilp32 program.c
```

**优化级别说明：**
- `-O0`: 无优化，便于调试
- `-O1`: 基本优化，编译速度快
- `-O2`: 推荐的优化级别，平衡性能和代码大小
- `-O3`: 激进优化，可能增加代码大小
- `-Os`: 优化代码大小

## 15.3 低功耗技术

CoralNPU 实现了多种低功耗技术，适用于电池供电和热受限的应用场景。

### 15.3.1 时钟门控（Clock Gating）

时钟门控是最基本的低功耗技术，通过关闭空闲模块的时钟来降低动态功耗。

#### 硬件实现

CoralNPU 使用专用的时钟门控单元：

```systemverilog
// ClockGate.sv
module ClockGate(
  input         clk_i,
  input         enable,  // '1' 时钟通过, '0' 时钟关闭
  input         te,      // 测试使能
  output        clk_o
);

`ifdef USE_TSMC12FFC
  // TSMC 12nm 工艺的时钟门控单元
  CKLNQD10BWP6T20P96CPDLVT u_cg(
    .TE(te),
    .E(enable),
    .CP(clk_i),
    .Q(clk_o)
  );
`else
  // 通用实现（用于仿真）
  logic en_latch;
  always_latch begin
    if (!clk_i) begin
      en_latch = enable | te;
    end
  end
  assign clk_o = en_latch & clk_i;
`endif

endmodule
```

**关键特性：**
- 使用 latch 避免毛刺（glitch）
- 在时钟低电平时采样使能信号
- 支持测试模式（te 信号）

#### 自动时钟门控

CoralNPU 在以下情况自动关闭时钟：

1. **WFI 指令：** 执行 `wfi`（等待中断）指令时，核心进入空闲状态

```c
// 等待中断，自动关闭时钟
void wait_for_interrupt() {
    asm volatile ("wfi");
}
```

2. **空闲检测：** 当所有执行单元空闲时，部分模块的时钟被门控

```scala
// CoreAxi.scala 中的时钟门控逻辑
val coreIdle = (scoreboard.regd === 0.U) && 
               !lsuActive && 
               rvvIdle
val clockGateEnable = !coreIdle || wfi
```

#### 手动时钟门控

通过 CSR 寄存器可以手动控制时钟门控：

```c
// 通过 CSR 控制时钟门控
#define CORALNPU_BASE 0x70000000
#define CSR_RESET_CONTROL (CORALNPU_BASE + 0x30000)

void coralnpu_clock_gate(int enable) {
    volatile uint32_t *reset_ctrl = (uint32_t*)CSR_RESET_CONTROL;
    if (enable) {
        *reset_ctrl |= (1 << 1);   // 设置 CLOCK_GATE 位
    } else {
        *reset_ctrl &= ~(1 << 1);  // 清除 CLOCK_GATE 位
    }
}
```

### 15.3.2 电源门控（Power Gating）

电源门控通过切断电源来消除静态功耗，适用于长时间空闲的场景。

#### 电源域划分

CoralNPU 可以划分为多个电源域：

```
+------------------+
|   Always-On      |  <- 时钟、复位、唤醒逻辑
+------------------+
|   Core Domain    |  <- CPU 核心
+------------------+
|   Cache Domain   |  <- L1 I/D Cache
+------------------+
|   TCM Domain     |  <- ITCM/DTCM
+------------------+
```

**电源门控序列：**

1. 保存状态（如果需要）
2. 关闭时钟
3. 隔离电源域（插入隔离单元）
4. 切断电源
5. （唤醒时）恢复电源
6. 移除隔离
7. 恢复时钟
8. 恢复状态

#### 软件控制

```c
// 电源管理接口（示例）
typedef enum {
    POWER_DOMAIN_CORE = 0,
    POWER_DOMAIN_CACHE = 1,
    POWER_DOMAIN_TCM = 2
} power_domain_t;

void power_domain_off(power_domain_t domain) {
    // 1. 保存状态
    save_domain_state(domain);
    
    // 2. 关闭时钟
    clock_gate_domain(domain, 1);
    
    // 3. 切断电源
    power_switch_domain(domain, 0);
}

void power_domain_on(power_domain_t domain) {
    // 1. 恢复电源
    power_switch_domain(domain, 1);
    
    // 2. 等待电源稳定
    delay_us(10);
    
    // 3. 恢复时钟
    clock_gate_domain(domain, 0);
    
    // 4. 恢复状态
    restore_domain_state(domain);
}
```

### 15.3.3 动态电压频率调节（DVFS）

DVFS 根据工作负载动态调整电压和频率，在性能和功耗之间取得平衡。

#### 工作点定义

```c
typedef struct {
    uint32_t voltage_mv;  // 电压（毫伏）
    uint32_t freq_mhz;    // 频率（兆赫兹）
    uint32_t power_mw;    // 功耗（毫瓦）
} operating_point_t;

const operating_point_t op_points[] = {
    {800, 100, 50},   // 低功耗模式
    {900, 200, 120},  // 平衡模式
    {1000, 400, 300}, // 高性能模式
    {1100, 600, 550}  // 超频模式
};
```

#### DVFS 控制器

```c
typedef enum {
    DVFS_MODE_LOW_POWER = 0,
    DVFS_MODE_BALANCED = 1,
    DVFS_MODE_HIGH_PERF = 2,
    DVFS_MODE_TURBO = 3
} dvfs_mode_t;

void dvfs_set_mode(dvfs_mode_t mode) {
    const operating_point_t *op = &op_points[mode];
    
    // 降频时：先降频，再降压
    if (op->freq_mhz < current_freq) {
        set_frequency(op->freq_mhz);
        delay_us(10);
        set_voltage(op->voltage_mv);
    }
    // 升频时：先升压，再升频
    else {
        set_voltage(op->voltage_mv);
        delay_us(100);  // 等待电压稳定
        set_frequency(op->freq_mhz);
    }
}
```

#### 自适应 DVFS

```c
// 基于负载的自适应 DVFS
void adaptive_dvfs_update() {
    uint32_t load = get_cpu_load_percent();
    
    if (load > 80) {
        dvfs_set_mode(DVFS_MODE_HIGH_PERF);
    } else if (load > 50) {
        dvfs_set_mode(DVFS_MODE_BALANCED);
    } else {
        dvfs_set_mode(DVFS_MODE_LOW_POWER);
    }
}
```

### 15.3.4 低功耗设计技巧

#### 1. 减少不必要的计算

```c
// 不好：每次都计算
for (int i = 0; i < n; i++) {
    result[i] = expensive_function(data[i]);
}

// 好：缓存结果
uint32_t cache[256];
int cache_valid[256] = {0};

for (int i = 0; i < n; i++) {
    uint8_t key = data[i] & 0xFF;
    if (!cache_valid[key]) {
        cache[key] = expensive_function(data[i]);
        cache_valid[key] = 1;
    }
    result[i] = cache[key];
}
```

#### 2. 使用低功耗指令

```c
// 使用 WFI 等待事件，而不是忙等待
// 不好：忙等待
while (!flag) {
    // 空循环，浪费功耗
}

// 好：使用 WFI
while (!flag) {
    asm volatile ("wfi");  // 等待中断，自动门控时钟
}
```

#### 3. 批处理

```c
// 批量处理数据，减少唤醒次数
#define BATCH_SIZE 64

void process_data_batch() {
    static uint32_t buffer[BATCH_SIZE];
    static int count = 0;
    
    buffer[count++] = get_data();
    
    if (count >= BATCH_SIZE) {
        // 一次性处理整批数据
        for (int i = 0; i < BATCH_SIZE; i++) {
            process(buffer[i]);
        }
        count = 0;
        
        // 处理完后进入低功耗模式
        asm volatile ("wfi");
    }
}
```

#### 4. 功耗预算

```c
// 功耗预算管理
typedef struct {
    uint32_t budget_mw;      // 功耗预算（毫瓦）
    uint32_t current_mw;     // 当前功耗
    uint32_t peak_mw;        // 峰值功耗
} power_budget_t;

power_budget_t power_budget = {
    .budget_mw = 500,
    .current_mw = 0,
    .peak_mw = 0
};

int power_budget_check(uint32_t additional_mw) {
    if (power_budget.current_mw + additional_mw > power_budget.budget_mw) {
        // 超出预算，降低性能
        dvfs_set_mode(DVFS_MODE_LOW_POWER);
        return 0;
    }
    return 1;
}
```


## 15.4 安全特性

CoralNPU 实现了多层安全机制，保护系统免受恶意代码和硬件攻击。

### 15.4.1 内存保护

#### 物理内存保护（PMP）

CoralNPU 支持 RISC-V 物理内存保护（PMP）机制，通过 CSR 寄存器配置内存访问权限。

**PMP 寄存器：**

```
pmpcfg0-pmpcfg3:  配置寄存器（每个包含 4 个 PMP 条目的配置）
pmpaddr0-pmpaddr15: 地址寄存器（定义保护区域）
```

**配置示例：**

```c
#include <stdint.h>

// PMP 配置位
#define PMP_R     0x01  // 读权限
#define PMP_W     0x02  // 写权限
#define PMP_X     0x04  // 执行权限
#define PMP_A_OFF 0x00  // 禁用
#define PMP_A_TOR 0x08  // Top of Range
#define PMP_A_NA4 0x10  // 自然对齐 4 字节
#define PMP_A_NAPOT 0x18 // 自然对齐 2 的幂次

// 设置 PMP 条目
void pmp_set_region(int entry, uint32_t addr, uint32_t size, uint8_t perm) {
    // 计算 NAPOT 编码
    uint32_t napot = (addr >> 2) | ((size - 1) >> 3);
    
    // 写入地址寄存器
    switch(entry) {
        case 0:
            asm volatile ("csrw pmpaddr0, %0" :: "r"(napot));
            break;
        case 1:
            asm volatile ("csrw pmpaddr1, %0" :: "r"(napot));
            break;
        // ... 其他条目
    }
    
    // 写入配置寄存器
    uint32_t cfg;
    asm volatile ("csrr %0, pmpcfg0" : "=r"(cfg));
    cfg &= ~(0xFF << (entry * 8));
    cfg |= ((perm | PMP_A_NAPOT) << (entry * 8));
    asm volatile ("csrw pmpcfg0, %0" :: "r"(cfg));
}

// 配置内存保护
void setup_memory_protection() {
    // 保护 ITCM（只读+执行）
    pmp_set_region(0, 0x00000000, 0x2000, PMP_R | PMP_X);
    
    // 保护 DTCM（读写，不可执行）
    pmp_set_region(1, 0x00010000, 0x8000, PMP_R | PMP_W);
    
    // 保护 CSR 区域（读写）
    pmp_set_region(2, 0x00030000, 0x1000, PMP_R | PMP_W);
    
    // 禁止访问其他区域
    pmp_set_region(3, 0x00000000, 0xFFFFFFFF, 0);
}
```

**PMP 违规处理：**

```c
// PMP 违规会触发访问异常
void handle_access_fault(uint32_t mcause, uint32_t mepc, uint32_t mtval) {
    if (mcause == 1) {  // 指令访问异常
        printf("Instruction access fault at PC=0x%08x, addr=0x%08x\n", 
               mepc, mtval);
    } else if (mcause == 5) {  // 加载访问异常
        printf("Load access fault at PC=0x%08x, addr=0x%08x\n", 
               mepc, mtval);
    } else if (mcause == 7) {  // 存储访问异常
        printf("Store access fault at PC=0x%08x, addr=0x%08x\n", 
               mepc, mtval);
    }
    
    // 终止程序或采取其他措施
    asm volatile ("ebreak");
}
```

#### 栈保护

```c
// 栈溢出检测
#define STACK_CANARY 0xDEADBEEF
#define STACK_SIZE 4096

uint32_t stack[STACK_SIZE / 4] __attribute__((aligned(16)));
uint32_t *stack_canary = &stack[0];

void stack_init() {
    *stack_canary = STACK_CANARY;
}

void stack_check() {
    if (*stack_canary != STACK_CANARY) {
        // 检测到栈溢出
        printf("Stack overflow detected!\n");
        asm volatile ("ebreak");
    }
}

// 在关键函数中检查栈
void critical_function() {
    stack_check();
    // ... 函数体
    stack_check();
}
```

### 15.4.2 安全启动

安全启动确保只有经过验证的代码才能在 CoralNPU 上执行。

#### 启动流程

```
+------------------+
| ROM Bootloader   |  <- 不可变，验证第一阶段
+------------------+
        |
        v
+------------------+
| Stage 1 Loader   |  <- 验证签名
+------------------+
        |
        v
+------------------+
| Stage 2 Loader   |  <- 加载应用程序
+------------------+
        |
        v
+------------------+
| Application      |  <- 用户代码
+------------------+
```

#### 签名验证

```c
#include <stdint.h>

// 简化的 RSA 签名验证（实际应使用加密库）
typedef struct {
    uint32_t modulus[64];   // 2048 位模数
    uint32_t exponent;      // 公钥指数
} rsa_public_key_t;

typedef struct {
    uint32_t signature[64]; // 2048 位签名
    uint32_t hash[8];       // SHA-256 哈希
} image_signature_t;

// 验证镜像签名
int verify_image_signature(const uint8_t *image, uint32_t size,
                           const image_signature_t *sig,
                           const rsa_public_key_t *pubkey) {
    // 1. 计算镜像的 SHA-256 哈希
    uint32_t computed_hash[8];
    sha256(image, size, computed_hash);
    
    // 2. 使用公钥解密签名
    uint32_t decrypted_hash[8];
    rsa_verify(sig->signature, pubkey, decrypted_hash);
    
    // 3. 比较哈希值
    for (int i = 0; i < 8; i++) {
        if (computed_hash[i] != decrypted_hash[i]) {
            return 0;  // 验证失败
        }
    }
    
    return 1;  // 验证成功
}

// 安全启动入口
void secure_boot() {
    // 从 ROM 读取公钥
    const rsa_public_key_t *pubkey = (rsa_public_key_t*)0x1000;
    
    // 从 Flash 读取镜像和签名
    const uint8_t *image = (uint8_t*)0x10000;
    uint32_t image_size = *(uint32_t*)0x10000;
    const image_signature_t *sig = (image_signature_t*)(0x10000 + image_size);
    
    // 验证签名
    if (!verify_image_signature(image, image_size, sig, pubkey)) {
        // 验证失败，停止启动
        printf("Signature verification failed!\n");
        while(1) {
            asm volatile ("wfi");
        }
    }
    
    // 验证成功，跳转到应用程序
    void (*app_entry)() = (void(*)())image;
    app_entry();
}
```

#### 安全启动配置

```c
// 启动配置（存储在 OTP 或 eFuse）
typedef struct {
    uint32_t secure_boot_enable;  // 是否启用安全启动
    uint32_t debug_disable;       // 是否禁用调试
    uint32_t key_revocation;      // 密钥撤销位图
    uint8_t  root_key_hash[32];   // 根密钥哈希
} boot_config_t;

// 读取启动配置
const boot_config_t* get_boot_config() {
    // 从 OTP 读取配置
    return (boot_config_t*)OTP_BASE_ADDR;
}

// 检查启动配置
void check_boot_config() {
    const boot_config_t *cfg = get_boot_config();
    
    if (cfg->secure_boot_enable) {
        // 启用安全启动
        secure_boot();
    }
    
    if (cfg->debug_disable) {
        // 禁用调试接口
        disable_debug_interface();
    }
}
```

### 15.4.3 加密支持

CoralNPU 可以集成硬件加密加速器，提供高性能的加密操作。

#### AES 加密

```c
// AES-128 加密（假设有硬件加速器）
#define AES_BASE 0x40000000
#define AES_KEY   (AES_BASE + 0x00)
#define AES_IV    (AES_BASE + 0x10)
#define AES_DATA  (AES_BASE + 0x20)
#define AES_CTRL  (AES_BASE + 0x30)
#define AES_STATUS (AES_BASE + 0x34)

typedef enum {
    AES_MODE_ECB = 0,
    AES_MODE_CBC = 1,
    AES_MODE_CTR = 2
} aes_mode_t;

void aes_encrypt(const uint8_t *key, const uint8_t *iv,
                 const uint8_t *input, uint8_t *output,
                 uint32_t length, aes_mode_t mode) {
    volatile uint32_t *aes_key = (uint32_t*)AES_KEY;
    volatile uint32_t *aes_iv = (uint32_t*)AES_IV;
    volatile uint32_t *aes_data = (uint32_t*)AES_DATA;
    volatile uint32_t *aes_ctrl = (uint32_t*)AES_CTRL;
    volatile uint32_t *aes_status = (uint32_t*)AES_STATUS;
    
    // 设置密钥
    for (int i = 0; i < 4; i++) {
        aes_key[i] = ((uint32_t*)key)[i];
    }
    
    // 设置 IV（如果需要）
    if (mode != AES_MODE_ECB) {
        for (int i = 0; i < 4; i++) {
            aes_iv[i] = ((uint32_t*)iv)[i];
        }
    }
    
    // 加密数据
    for (uint32_t i = 0; i < length; i += 16) {
        // 写入输入数据
        for (int j = 0; j < 4; j++) {
            aes_data[j] = ((uint32_t*)(input + i))[j];
        }
        
        // 启动加密
        *aes_ctrl = (1 << 0) | (mode << 1);  // START | MODE
        
        // 等待完成
        while ((*aes_status & 0x1) == 0);
        
        // 读取输出数据
        for (int j = 0; j < 4; j++) {
            ((uint32_t*)(output + i))[j] = aes_data[j];
        }
    }
}
```

#### 真随机数生成器（TRNG）

```c
// 真随机数生成器接口
#define TRNG_BASE 0x40001000
#define TRNG_DATA (TRNG_BASE + 0x00)
#define TRNG_CTRL (TRNG_BASE + 0x04)
#define TRNG_STATUS (TRNG_BASE + 0x08)

void trng_init() {
    volatile uint32_t *trng_ctrl = (uint32_t*)TRNG_CTRL;
    *trng_ctrl = 0x1;  // 启用 TRNG
}

uint32_t trng_get_random() {
    volatile uint32_t *trng_data = (uint32_t*)TRNG_DATA;
    volatile uint32_t *trng_status = (uint32_t*)TRNG_STATUS;
    
    // 等待随机数就绪
    while ((*trng_status & 0x1) == 0);
    
    return *trng_data;
}

// 生成随机密钥
void generate_random_key(uint8_t *key, uint32_t length) {
    for (uint32_t i = 0; i < length; i += 4) {
        uint32_t random = trng_get_random();
        for (int j = 0; j < 4 && (i + j) < length; j++) {
            key[i + j] = (random >> (j * 8)) & 0xFF;
        }
    }
}
```

#### 安全密钥存储

```c
// 密钥存储在 OTP（一次性可编程存储器）
#define OTP_KEY_BASE 0x50000000

typedef struct {
    uint8_t aes_key[16];      // AES-128 密钥
    uint8_t hmac_key[32];     // HMAC-SHA256 密钥
    uint8_t device_id[16];    // 设备唯一 ID
} secure_keys_t;

// 读取安全密钥（只能在安全模式下）
const secure_keys_t* get_secure_keys() {
    // 检查是否在安全模式
    uint32_t mstatus;
    asm volatile ("csrr %0, mstatus" : "=r"(mstatus));
    if ((mstatus & 0x3) != 0x3) {  // 不在 M 模式
        return NULL;
    }
    
    return (secure_keys_t*)OTP_KEY_BASE;
}

// 使用安全密钥加密数据
void encrypt_with_device_key(const uint8_t *input, uint8_t *output, 
                             uint32_t length) {
    const secure_keys_t *keys = get_secure_keys();
    if (keys == NULL) {
        // 无法访问密钥
        return;
    }
    
    uint8_t iv[16] = {0};  // 初始化向量
    aes_encrypt(keys->aes_key, iv, input, output, length, AES_MODE_CBC);
}
```

### 15.4.4 侧信道攻击防护

#### 时间恒定算法

```c
// 时间恒定的内存比较（防止时序攻击）
int secure_memcmp(const void *a, const void *b, size_t n) {
    const uint8_t *pa = (const uint8_t*)a;
    const uint8_t *pb = (const uint8_t*)b;
    uint8_t diff = 0;
    
    // 始终比较所有字节，不提前返回
    for (size_t i = 0; i < n; i++) {
        diff |= pa[i] ^ pb[i];
    }
    
    return diff;
}

// 时间恒定的密码验证
int verify_password(const char *input, const char *stored_hash) {
    uint8_t input_hash[32];
    sha256((uint8_t*)input, strlen(input), input_hash);
    
    // 使用时间恒定比较
    return secure_memcmp(input_hash, stored_hash, 32) == 0;
}
```

#### 随机化

```c
// 地址空间布局随机化（ASLR）
void* allocate_with_aslr(size_t size) {
    // 获取随机偏移
    uint32_t random_offset = trng_get_random() & 0xFFF;
    
    // 在基地址上加上随机偏移
    void *base = (void*)0x20000000;
    return (void*)((uint32_t)base + random_offset);
}

// 指令执行随机化（防止功耗分析）
void random_delay() {
    uint32_t delay = trng_get_random() & 0xF;
    for (uint32_t i = 0; i < delay; i++) {
        asm volatile ("nop");
    }
}
```

## 15.5 未来扩展

CoralNPU 的架构设计考虑了未来的扩展性，本节介绍可能的改进方向。

### 15.5.1 多核支持

#### 多核架构

```
+--------+  +--------+  +--------+  +--------+
| Core 0 |  | Core 1 |  | Core 2 |  | Core 3 |
+--------+  +--------+  +--------+  +--------+
    |           |           |           |
    +-----+-----+-----+-----+-----+-----+
          |                       |
    +-----v-----+           +-----v-----+
    | L2 Cache  |           | L2 Cache  |
    +-----------+           +-----------+
          |                       |
          +----------+------------+
                     |
              +------v------+
              | Interconnect|
              +-------------+
                     |
              +------v------+
              |   Memory    |
              +-------------+
```

**多核配置参数：**

```scala
// Parameters.scala
class MultiCoreParameters extends Parameters {
  val numCores = 4
  val l2CacheSize = 256 * 1024  // 256KB
  val coherenceProtocol = "MESI"  // 缓存一致性协议
}
```

#### 核间通信

```c
// 核间中断（IPI）
#define IPI_BASE 0x60000000

void send_ipi(int target_core, uint32_t message) {
    volatile uint32_t *ipi_reg = (uint32_t*)(IPI_BASE + target_core * 4);
    *ipi_reg = message;
}

void ipi_handler() {
    volatile uint32_t *ipi_reg = (uint32_t*)(IPI_BASE + get_core_id() * 4);
    uint32_t message = *ipi_reg;
    
    // 处理消息
    process_ipi_message(message);
    
    // 清除中断
    *ipi_reg = 0;
}

// 共享内存通信
typedef struct {
    volatile uint32_t lock;
    volatile uint32_t data[256];
} shared_buffer_t;

shared_buffer_t *shared_buf = (shared_buffer_t*)0x70000000;

void acquire_lock(volatile uint32_t *lock) {
    while (1) {
        uint32_t expected = 0;
        // 使用原子操作获取锁
        asm volatile (
            "amoswap.w.aq %0, %1, (%2)"
            : "=r"(expected)
            : "r"(1), "r"(lock)
            : "memory"
        );
        if (expected == 0) break;
    }
}

void release_lock(volatile uint32_t *lock) {
    asm volatile (
        "amoswap.w.rl zero, zero, (%0)"
        :: "r"(lock)
        : "memory"
    );
}
```

### 15.5.2 更大的向量宽度

#### 扩展到 256 位向量

```scala
// 扩展向量宽度
class ExtendedVectorParameters extends Parameters {
  override val rvvVlen = 256  // 从 128 位扩展到 256 位
  
  // 相应调整其他参数
  override val lsuDataBits = 512  // 加倍访存带宽
}
```

**性能提升：**

```c
// 256 位向量可以同时处理 8 个 32 位整数
void add_arrays_vec256(int32_t *a, int32_t *b, int32_t *c, int n) {
    size_t vl;
    for (size_t i = 0; i < n; i += vl) {
        vl = vsetvl_e32m1(n - i);  // 最多 8 个元素
        vint32m1_t va = vle32_v_i32m1(a + i, vl);
        vint32m1_t vb = vle32_v_i32m1(b + i, vl);
        vint32m1_t vc = vadd_vv_i32m1(va, vb, vl);
        vse32_v_i32m1(c + i, vc, vl);
    }
}
```

### 15.5.3 新的指令扩展

#### 位操作扩展（Zbs）

```c
// 单比特操作
static inline uint32_t bset(uint32_t rs1, uint32_t rs2) {
    // 设置第 rs2 位
    return rs1 | (1U << (rs2 & 0x1F));
}

static inline uint32_t bclr(uint32_t rs1, uint32_t rs2) {
    // 清除第 rs2 位
    return rs1 & ~(1U << (rs2 & 0x1F));
}

static inline uint32_t binv(uint32_t rs1, uint32_t rs2) {
    // 翻转第 rs2 位
    return rs1 ^ (1U << (rs2 & 0x1F));
}

static inline uint32_t bext(uint32_t rs1, uint32_t rs2) {
    // 提取第 rs2 位
    return (rs1 >> (rs2 & 0x1F)) & 1;
}
```

#### 加密扩展（Zkn）

```c
// AES 加密指令
static inline uint32_t aes32esi(uint32_t rs1, uint32_t rs2, int bs) {
    // AES 加密中间轮
    uint32_t result;
    asm volatile (
        ".insn r 0x33, 0x0, 0x16, %0, %1, %2"
        : "=r"(result)
        : "r"(rs1), "r"(rs2 | (bs << 30))
    );
    return result;
}

// SHA-256 指令
static inline uint32_t sha256sum0(uint32_t rs1) {
    uint32_t result;
    asm volatile (
        ".insn i 0x13, 0x1, %0, %1, 0x100"
        : "=r"(result)
        : "r"(rs1)
    );
    return result;
}
```

### 15.5.4 其他可能的改进

#### 1. 乱序执行

```
当前：顺序发射，顺序执行，顺序提交
未来：顺序发射，乱序执行，顺序提交

优势：
- 更好地隐藏访存延迟
- 提高指令级并行度
- 更高的 IPC（每周期指令数）

挑战：
- 更复杂的调度逻辑
- 需要重排序缓冲（ROB）
- 功耗和面积增加
```

#### 2. 分支预测

```c
// 当前：静态预测（向后taken，向前not-taken）
// 未来：动态分支预测

// 两级自适应预测器
typedef struct {
    uint8_t history;      // 分支历史
    uint8_t prediction;   // 预测状态
} branch_predictor_entry_t;

branch_predictor_entry_t bht[256];  // 分支历史表

int predict_branch(uint32_t pc) {
    int index = (pc >> 2) & 0xFF;
    return bht[index].prediction >= 2;  // 2-bit 饱和计数器
}

void update_predictor(uint32_t pc, int taken) {
    int index = (pc >> 2) & 0xFF;
    if (taken && bht[index].prediction < 3) {
        bht[index].prediction++;
    } else if (!taken && bht[index].prediction > 0) {
        bht[index].prediction--;
    }
}
```

#### 3. 硬件预取

```scala
// 硬件预取器
class HardwarePrefetcher extends Module {
  val io = IO(new Bundle {
    val access = Input(Valid(UInt(32.W)))  // 访存地址
    val prefetch = Output(Valid(UInt(32.W)))  // 预取地址
  })
  
  // 步长预取器
  val lastAddr = RegInit(0.U(32.W))
  val stride = RegInit(0.U(32.W))
  
  when (io.access.valid) {
    stride := io.access.bits - lastAddr
    lastAddr := io.access.bits
  }
  
  // 预测下一个访问地址
  io.prefetch.valid := io.access.valid
  io.prefetch.bits := io.access.bits + stride
}
```

#### 4. 神经网络加速

```c
// 矩阵乘法加速指令（假设）
void matmul_accelerated(float *A, float *B, float *C, int M, int N, int K) {
    // 使用专用的矩阵乘法加速器
    volatile uint32_t *accel_ctrl = (uint32_t*)0x80000000;
    volatile uint32_t *accel_a = (uint32_t*)0x80000004;
    volatile uint32_t *accel_b = (uint32_t*)0x80000008;
    volatile uint32_t *accel_c = (uint32_t*)0x8000000C;
    
    *accel_a = (uint32_t)A;
    *accel_b = (uint32_t)B;
    *accel_c = (uint32_t)C;
    *accel_ctrl = (M << 16) | (N << 8) | K;
    
    // 等待完成
    while (*accel_ctrl & 0x1);
}
```

#### 5. 可重配置计算

```c
// FPGA 风格的可重配置逻辑
typedef struct {
    uint32_t lut[16];     // 查找表
    uint32_t routing;     // 路由配置
} configurable_logic_t;

void configure_logic(configurable_logic_t *logic, 
                    uint32_t function) {
    // 配置可重配置逻辑实现特定功能
    for (int i = 0; i < 16; i++) {
        logic->lut[i] = (function >> i) & 1;
    }
}
```

## 15.6 总结

本章介绍了 CoralNPU 的高级特性：

1. **自定义指令**：如何扩展指令集以满足特定需求
2. **性能调优**：充分利用硬件特性提升性能的技巧
3. **低功耗技术**：时钟门控、电源门控和 DVFS 等节能方法
4. **安全特性**：内存保护、安全启动和加密支持
5. **未来扩展**：多核、更大向量宽度等发展方向

这些高级特性使 CoralNPU 能够适应各种应用场景，从低功耗嵌入式系统到高性能计算平台。

### 参考资源

- RISC-V 指令集手册：https://riscv.org/specifications/
- RISC-V 向量扩展规范：https://github.com/riscv/riscv-v-spec
- CoralNPU 源代码：/home/curry/code/coralnpu/
- 低功耗设计技术：IEEE 相关论文
- 硬件安全：NIST 安全标准

### 下一步

- 阅读 CoralNPU 源代码，深入理解实现细节
- 尝试添加自己的自定义指令
- 使用性能分析工具优化应用程序
- 参与 CoralNPU 社区，贡献代码和想法

---

**注意事项：**

1. 本章内容涉及硬件修改，需要对 Chisel 和数字电路设计有一定了解
2. 自定义指令需要重新生成 RTL 并综合
3. 安全特性的实现需要配合硬件支持（如 OTP、TRNG）
4. 性能优化效果取决于具体应用和数据特征
5. 未来扩展部分是设计方向，并非当前实现

