# 编程指南

本章介绍如何为 CoralNPU 编写高效的程序，包括 C 编程、汇编编程、向量编程以及各种优化技巧。

## 1. C 编程

### 1.1 基本的 C 程序结构

CoralNPU 程序的典型结构包括：

1. **输入缓冲区**：存储计算所需的输入数据
2. **输出缓冲区**：存储计算结果
3. **计算逻辑**：在 `main` 函数中实现

下面是一个简单的示例：

```c
#include <stdint.h>

// 定义输入和输出缓冲区
uint32_t input1_buffer[8] __attribute__((section(".data")));
uint32_t input2_buffer[8] __attribute__((section(".data")));
uint32_t output_buffer[8] __attribute__((section(".data")));

int main(int argc, char** argv) {
    // 执行逐元素加法
    for (int i = 0; i < 8; i++) {
        output_buffer[i] = input1_buffer[i] + input2_buffer[i];
    }
    return 0;
}
```

**代码解释**：

- `__attribute__((section(".data")))`：这是一个编译器属性（attribute），告诉编译器把这个变量放到 `.data` 段（section）中。`.data` 段通常存放已初始化的全局变量和静态变量。在 CoralNPU 中，链接脚本会将 `.data` 段分配到 DTCM（数据紧耦合内存）中。
- 核心会在从 `main` 函数返回时停止运行。

### 1.2 如何访问硬件寄存器

在嵌入式编程中，经常需要直接访问硬件寄存器来控制外设。CoralNPU 使用内存映射 I/O（Memory-Mapped I/O），即硬件寄存器被映射到特定的内存地址。

#### 定义寄存器地址

首先定义外设的基地址和各个寄存器的偏移：

```c
#include <stdint.h>

// GPIO 外设基地址
#define GPIO_BASE 0x40030000

// GPIO 寄存器地址
#define GPIO_DATA_IN  (GPIO_BASE + 0x00)  // 输入数据寄存器
#define GPIO_DATA_OUT (GPIO_BASE + 0x04)  // 输出数据寄存器
#define GPIO_OUT_EN   (GPIO_BASE + 0x08)  // 输出使能寄存器
```

**解释**：
- `#define` 是 C 语言的宏定义，用于定义常量。
- `0x40030000` 是十六进制数，表示 GPIO 外设的起始地址。
- 每个寄存器都有一个偏移量（如 `0x00`、`0x04`），加上基地址就得到寄存器的完整地址。

#### 定义寄存器访问宏

为了方便访问寄存器，定义一个宏：

```c
#define REG32(addr) (*(volatile uint32_t*)(addr))
```

**详细解释**（这里涉及 C 语言的指针和类型转换）：

1. `(addr)`：传入的地址参数
2. `(volatile uint32_t*)`：将地址转换为"指向 volatile uint32_t 类型的指针"
   - `uint32_t`：32 位无符号整数类型
   - `*`：指针符号，表示这是一个指针
   - `volatile`：稍后详细解释
3. `*(...)` ：解引用（dereference）操作符，表示访问指针指向的内存位置

简单来说，`REG32(addr)` 就是把地址 `addr` 当作一个 32 位寄存器来读写。

#### 使用示例

```c
int main() {
    // 1. 配置所有引脚为输出模式
    REG32(GPIO_OUT_EN) = 0xFF;
    
    // 2. 向输出寄存器写入 0xAA
    REG32(GPIO_DATA_OUT) = 0xAA;
    
    // 3. 从输出寄存器读回数据
    volatile uint32_t val = REG32(GPIO_DATA_OUT);
    
    // 4. 检查读回的值是否正确
    if ((val & 0xFF) != 0xAA) {
        return 1;  // 测试失败
    }
    
    return 0;  // 测试成功
}
```

### 1.3 volatile 关键字的使用

`volatile` 是 C 语言中一个非常重要的关键字，在访问硬件寄存器时必须使用。

#### 为什么需要 volatile？

编译器在优化代码时，可能会做一些"聪明"的事情，比如：

- **消除冗余读取**：如果你连续两次读取同一个变量，编译器可能认为第二次读取是多余的，直接使用第一次读取的值。
- **重排指令**：编译器可能会调整指令的执行顺序以提高性能。

但是，硬件寄存器的值可能会被硬件自动改变（比如接收到新数据），或者写入操作本身就有副作用（比如触发某个硬件动作）。如果编译器优化掉了这些访问，程序就会出错。

#### volatile 的作用

`volatile` 告诉编译器：

- 每次访问这个变量时，都必须真正地从内存中读取，不能使用缓存的值。
- 不要优化掉任何对这个变量的读写操作。
- 不要重排涉及这个变量的指令。

#### 示例对比

**不使用 volatile（错误）**：

```c
uint32_t* gpio_reg = (uint32_t*)GPIO_DATA_IN;

// 连续读取两次
uint32_t val1 = *gpio_reg;
uint32_t val2 = *gpio_reg;  // 编译器可能优化为 val2 = val1
```

**使用 volatile（正确）**：

```c
volatile uint32_t* gpio_reg = (volatile uint32_t*)GPIO_DATA_IN;

// 连续读取两次
uint32_t val1 = *gpio_reg;  // 真正从硬件读取
uint32_t val2 = *gpio_reg;  // 再次从硬件读取
```

#### 何时使用 volatile

- 访问硬件寄存器
- 在中断处理函数和主程序之间共享的变量
- 多线程程序中共享的变量（但通常还需要其他同步机制）

### 1.4 内存对齐

内存对齐（Memory Alignment）是指数据在内存中的起始地址必须是某个值的整数倍。

#### 为什么需要内存对齐？

1. **性能**：许多处理器访问对齐的数据更快。
2. **硬件要求**：某些处理器或指令（如向量指令）要求数据必须对齐，否则会出错。

#### 对齐要求

- `uint8_t`（1 字节）：可以放在任何地址
- `uint16_t`（2 字节）：地址必须是 2 的倍数
- `uint32_t`（4 字节）：地址必须是 4 的倍数
- `uint64_t`（8 字节）：地址必须是 8 的倍数

#### 使用 __attribute__((aligned(N)))

```c
// 对齐到 16 字节边界
uint8_t buffer[512] __attribute__((aligned(16)));
```

**解释**：
- `aligned(16)` 表示这个数组的起始地址必须是 16 的倍数。
- 这对于使用向量指令处理数据非常重要。

#### 完整示例

```c
#include <stdint.h>

// 输入缓冲区，对齐到 16 字节
uint8_t in_buf[512] __attribute__((section(".data"))) 
                    __attribute__((aligned(16)));

// 输出缓冲区，对齐到 16 字节
uint8_t out_buf[512] __attribute__((section(".data"))) 
                     __attribute__((aligned(16)));

int main() {
    // 使用这些缓冲区进行向量操作
    // ...
    return 0;
}
```

## 2. 汇编编程

### 2.1 如何编写汇编函数

汇编语言直接对应处理器指令，可以实现最精细的控制。CoralNPU 使用 RISC-V 汇编语法。

#### 基本汇编文件结构

创建一个 `.S` 文件（注意是大写的 S）：

```asm
    .section .text          # 代码段
    .balign 4               # 4 字节对齐
    .global _start          # 声明全局符号
    .type _start, @function # 声明这是一个函数
_start:
    li x1, (1 << 1)         # 加载立即数到寄存器 x1
    li x2, (1 << 2)         # 加载立即数到寄存器 x2
    add x3, x1, x2          # x3 = x1 + x2
    wfi                     # 等待中断（停止执行）
```

**汇编指令解释**：

- `.section .text`：声明这是代码段（section）。
- `.balign 4`：确保后面的代码按 4 字节对齐。
- `.global _start`：声明 `_start` 是一个全局符号，可以被其他文件引用。
- `.type _start, @function`：告诉汇编器这是一个函数。
- `li`（Load Immediate）：加载立即数指令。
- `add`：加法指令。
- `wfi`（Wait For Interrupt）：等待中断指令。

#### RISC-V 寄存器

RISC-V 有 32 个通用寄存器，编号为 x0 到 x31：

- `x0`：永远是 0，写入无效
- `x1`（ra）：返回地址
- `x2`（sp）：栈指针
- `x3-x31`：通用寄存器

还有 32 个浮点寄存器 f0 到 f31。

### 2.2 汇编与 C 的混合编程

可以在 C 程序中调用汇编函数，或在汇编中调用 C 函数。

#### 在 C 中声明汇编函数

```c
// 声明一个汇编函数
extern int asm_add(int a, int b);

int main() {
    int result = asm_add(10, 20);
    return result;
}
```

#### 实现汇编函数

```asm
    .section .text
    .global asm_add
    .type asm_add, @function
asm_add:
    # 参数 a 在 a0 寄存器，参数 b 在 a1 寄存器
    add a0, a0, a1    # a0 = a0 + a1
    ret               # 返回，结果在 a0 中
```

**调用约定解释**：

- RISC-V 的调用约定规定：
  - 前 8 个整数参数通过寄存器 a0-a7 传递
  - 返回值通过 a0（和 a1，如果需要）返回
  - `ret` 指令等价于 `jalr x0, 0(ra)`，即跳转到返回地址

### 2.3 内联汇编

内联汇编允许在 C 代码中直接嵌入汇编指令，非常方便。

#### 基本语法

```c
asm volatile (
    "汇编指令"
    : 输出操作数列表
    : 输入操作数列表
    : 破坏列表
);
```

**语法解释**：

- `asm` 或 `__asm__`：内联汇编关键字。
- `volatile`：告诉编译器不要优化这段汇编代码。
- 三个冒号分隔四个部分：汇编指令、输出、输入、破坏列表。

#### 简单示例：NOP 指令

```c
int main(void) {
    for (int i = 0; i <= 100; i++) {
        asm volatile("nop");  // 空操作指令
    }
    return 0;
}
```

`nop`（No Operation）指令不做任何事情，常用于延时或占位。

#### 读取 CSR 寄存器

CSR（Control and Status Registers）是 RISC-V 的控制和状态寄存器。

```c
#include <stdint.h>

uint64_t mcycle_read(void) {
    uint32_t cycle_low = 0;
    uint32_t cycle_high = 0;
    uint32_t cycle_high_2 = 0;
    
    asm volatile(
        "1:"                      // 标签 1
        "  csrr %0, mcycleh;"     // 读取 mcycleh 到 cycle_high
        "  csrr %1, mcycle;"      // 读取 mcycle 到 cycle_low
        "  csrr %2, mcycleh;"     // 再次读取 mcycleh 到 cycle_high_2
        "  bne  %0, %2, 1b;"      // 如果两次读取不同，跳回标签 1
        : "=r"(cycle_high), "=r"(cycle_low), "=r"(cycle_high_2)
        :
    );
    
    return ((uint64_t)cycle_high << 32) | cycle_low;
}
```

**详细解释**：

- `csrr`（CSR Read）：读取 CSR 寄存器的指令。
- `mcycle` 和 `mcycleh`：64 位周期计数器的低 32 位和高 32 位。
- `%0`、`%1`、`%2`：占位符，对应输出操作数列表中的变量。
- `"=r"`：表示这是一个输出操作数，`r` 表示使用任意通用寄存器。
- 为什么要读两次 `mcycleh`？因为在读取过程中，`mcycle` 可能溢出导致 `mcycleh` 增加，所以需要检查前后两次读取是否一致。

#### 写入 CSR 寄存器

```c
void cycle_counter_reset(void) {
    asm volatile(
        "csrwi mcycleh, 1;"       // 写立即数 1 到 mcycleh
        "li a0, 0xfffffff0;"      // 加载立即数到 a0
        "csrrw a0, mcycle, a0;"   // 交换 a0 和 mcycle 的值
        :                         // 无输出
        :                         // 无输入
        : "a0"                    // a0 寄存器被修改
    );
}
```

**指令解释**：

- `csrwi`（CSR Write Immediate）：将立即数写入 CSR。
- `csrrw`（CSR Read-Write）：原子地读取 CSR 到寄存器，并将寄存器的值写入 CSR。
- 破坏列表 `"a0"` 告诉编译器 a0 寄存器的值被改变了。

### 2.4 调用约定

调用约定（Calling Convention）规定了函数调用时如何传递参数、返回值以及哪些寄存器需要保存。

#### RISC-V 调用约定

| 寄存器 | ABI 名称 | 用途 | 调用者/被调用者保存 |
|--------|----------|------|---------------------|
| x0 | zero | 常量 0 | - |
| x1 | ra | 返回地址 | 调用者保存 |
| x2 | sp | 栈指针 | 被调用者保存 |
| x3 | gp | 全局指针 | - |
| x4 | tp | 线程指针 | - |
| x5-x7 | t0-t2 | 临时寄存器 | 调用者保存 |
| x8 | s0/fp | 保存寄存器/帧指针 | 被调用者保存 |
| x9 | s1 | 保存寄存器 | 被调用者保存 |
| x10-x11 | a0-a1 | 参数/返回值 | 调用者保存 |
| x12-x17 | a2-a7 | 参数 | 调用者保存 |
| x18-x27 | s2-s11 | 保存寄存器 | 被调用者保存 |
| x28-x31 | t3-t6 | 临时寄存器 | 调用者保存 |

**解释**：

- **调用者保存**：如果调用者需要在函数调用后继续使用这些寄存器的值，必须在调用前保存它们。
- **被调用者保存**：被调用的函数如果要使用这些寄存器，必须先保存它们的值，并在返回前恢复。

#### 示例：保存和恢复寄存器

```asm
my_function:
    # 保存需要使用的被调用者保存寄存器
    addi sp, sp, -16      # 分配栈空间
    sw s0, 0(sp)          # 保存 s0
    sw s1, 4(sp)          # 保存 s1
    
    # 函数体
    # ... 使用 s0 和 s1 ...
    
    # 恢复寄存器
    lw s0, 0(sp)          # 恢复 s0
    lw s1, 4(sp)          # 恢复 s1
    addi sp, sp, 16       # 释放栈空间
    
    ret                   # 返回
```

**指令解释**：

- `addi`（Add Immediate）：立即数加法，这里用于调整栈指针。
- `sw`（Store Word）：将寄存器的值存储到内存。
- `lw`（Load Word）：从内存加载值到寄存器。
- 栈是向下增长的，所以分配空间时减小 sp，释放空间时增加 sp。


## 3. 向量编程

CoralNPU 支持 RISC-V 向量扩展（RVV），可以同时处理多个数据，大幅提升性能。

### 3.1 如何使用向量指令

#### 什么是向量指令？

传统的标量指令一次只能处理一个数据：

```c
// 标量代码：一次处理一个元素
for (int i = 0; i < 1024; i++) {
    output[i] = input1[i] + input2[i];
}
```

向量指令可以一次处理多个数据：

```c
// 向量代码：一次处理 32 个元素
for (int i = 0; i < 1024; i += 32) {
    // 一条指令处理 32 个元素
    vector_add(&output[i], &input1[i], &input2[i], 32);
}
```

#### 向量寄存器

RISC-V 向量扩展提供了 32 个向量寄存器（v0-v31），每个寄存器可以存储多个元素。向量寄存器的长度是可配置的，CoralNPU 支持最大 256 位（32 字节）。

#### 向量长度（VL）和元素宽度（SEW）

- **SEW**（Selected Element Width）：每个元素的位宽，可以是 8、16、32 或 64 位。
- **VL**（Vector Length）：本次操作处理的元素个数。
- **LMUL**（Length Multiplier）：向量寄存器组的大小，可以是 1/8、1/4、1/2、1、2、4、8。

例如：
- SEW=8，LMUL=1：一个向量寄存器可以存储 32 个 8 位元素
- SEW=16，LMUL=2：两个向量寄存器可以存储 32 个 16 位元素

### 3.2 向量内置函数（intrinsics）

RISC-V 提供了一套 C 语言内置函数（intrinsics），可以在 C 代码中使用向量指令，而不需要写汇编。

#### 包含头文件

```c
#include <riscv_vector.h>
```

#### 基本向量操作示例

```c
#include <riscv_vector.h>
#include <stdint.h>

int8_t input1[1024];
int8_t input2[1024];
int16_t output[1024];

int main() {
    const int8_t* input1_ptr = &input1[0];
    const int8_t* input2_ptr = &input2[0];
    int16_t* output_ptr = &output[0];
    
    // 每次处理 32 个元素
    for (int idx = 0; (idx + 31) < 1024; idx += 32) {
        // 从内存加载 32 个 int8_t 到向量寄存器
        vint8m4_t input_v1 = __riscv_vle8_v_i8m4(input1_ptr + idx, 32);
        vint8m4_t input_v2 = __riscv_vle8_v_i8m4(input2_ptr + idx, 32);
        
        // 向量加法，结果扩展为 int16_t
        vint16m8_t temp_sum = __riscv_vwadd_vv_i16m8(input_v1, input_v2, 32);
        
        // 将结果存储回内存
        __riscv_vse16_v_i16m8(output_ptr + idx, temp_sum, 32);
    }
    
    return 0;
}
```

**代码解释**：

- `vint8m4_t`：向量类型，表示元素类型为 int8，LMUL=4。
  - `v`：向量
  - `int8`：8 位有符号整数
  - `m4`：LMUL=4，使用 4 个向量寄存器组
- `__riscv_vle8_v_i8m4(ptr, vl)`：向量加载指令
  - `vle8`：加载 8 位元素
  - `v`：从内存加载
  - `i8m4`：目标类型
  - `ptr`：内存地址
  - `vl`：向量长度（要加载的元素个数）
- `__riscv_vwadd_vv_i16m8(v1, v2, vl)`：向量加宽加法
  - `vwadd`：加宽加法（widening add），结果位宽是输入的两倍
  - `vv`：两个向量操作数
  - `i16m8`：结果类型
- `__riscv_vse16_v_i16m8(ptr, v, vl)`：向量存储指令
  - `vse16`：存储 16 位元素
  - `v`：存储到内存

#### 常用向量内置函数

**加载和存储**：

```c
// 单位步长加载（连续内存）
vint32m1_t v = __riscv_vle32_v_i32m1(ptr, vl);

// 单位步长存储
__riscv_vse32_v_i32m1(ptr, v, vl);

// 步长加载（非连续内存）
vint32m1_t v = __riscv_vlse32_v_i32m1(ptr, stride, vl);

// 索引加载
vint32m1_t v = __riscv_vluxei32_v_i32m1(base, index, vl);
```

**算术运算**：

```c
// 向量加法
vint32m1_t result = __riscv_vadd_vv_i32m1(v1, v2, vl);

// 向量减法
vint32m1_t result = __riscv_vsub_vv_i32m1(v1, v2, vl);

// 向量乘法
vint32m1_t result = __riscv_vmul_vv_i32m1(v1, v2, vl);

// 向量与标量运算
vint32m1_t result = __riscv_vadd_vx_i32m1(v, scalar, vl);
```

**类型转换**：

```c
// 加宽转换（8位 -> 16位）
vint16m2_t result = __riscv_vwadd_vv_i16m2(v1, v2, vl);

// 窄化转换（32位 -> 16位）
vint16m1_t result = __riscv_vnsra_wx_i16m1(v, shift, vl);
```

### 3.3 向量化循环

#### 标量版本

```c
void add_arrays(int32_t* out, const int32_t* in1, const int32_t* in2, size_t n) {
    for (size_t i = 0; i < n; i++) {
        out[i] = in1[i] + in2[i];
    }
}
```

#### 向量化版本

```c
#include <riscv_vector.h>

void add_arrays_vector(int32_t* out, const int32_t* in1, const int32_t* in2, size_t n) {
    size_t vl;
    
    for (size_t i = 0; i < n; i += vl) {
        // 设置向量长度（自动处理剩余元素）
        vl = __riscv_vsetvl_e32m1(n - i);
        
        // 加载
        vint32m1_t v1 = __riscv_vle32_v_i32m1(in1 + i, vl);
        vint32m1_t v2 = __riscv_vle32_v_i32m1(in2 + i, vl);
        
        // 计算
        vint32m1_t vsum = __riscv_vadd_vv_i32m1(v1, v2, vl);
        
        // 存储
        __riscv_vse32_v_i32m1(out + i, vsum, vl);
    }
}
```

**关键点**：

- `__riscv_vsetvl_e32m1(avl)`：设置向量长度
  - `avl`（Application Vector Length）：应用程序请求的向量长度
  - 返回实际的向量长度（可能小于请求值）
  - 自动处理不足一个向量长度的剩余元素

#### 向量化 memcpy

```c
#include <riscv_vector.h>

void* vector_memcpy(void* dst, const void* src, size_t n) {
    const uint8_t* s = (const uint8_t*)src;
    uint8_t* d = (uint8_t*)dst;
    size_t vl = 0;
    
    while (n > 0) {
        // 设置向量长度，使用 LMUL=8 以最大化吞吐量
        vl = __riscv_vsetvl_e8m8(n);
        
        // 加载
        vuint8m8_t vdata = __riscv_vle8_v_u8m8(s, vl);
        
        // 存储
        __riscv_vse8_v_u8m8(d, vdata, vl);
        
        // 更新指针和计数
        s += vl;
        d += vl;
        n -= vl;
    }
    
    return dst;
}
```

### 3.4 性能优化技巧

#### 1. 选择合适的 LMUL

LMUL 越大，一次可以处理的元素越多，但占用的寄存器也越多。

```c
// LMUL=1：每次处理较少元素，但寄存器压力小
vl = __riscv_vsetvl_e32m1(n);

// LMUL=8：每次处理更多元素，但需要 8 个向量寄存器
vl = __riscv_vsetvl_e32m8(n);
```

**建议**：
- 简单操作（如 memcpy）：使用 LMUL=8
- 复杂操作（需要多个中间变量）：使用 LMUL=1 或 2

#### 2. 减少内存访问

```c
// 不好：多次加载同一数据
for (int i = 0; i < n; i += vl) {
    vl = __riscv_vsetvl_e32m1(n - i);
    vint32m1_t v = __riscv_vle32_v_i32m1(data + i, vl);
    vint32m1_t r1 = __riscv_vadd_vx_i32m1(v, 1, vl);
    vint32m1_t r2 = __riscv_vmul_vx_i32m1(v, 2, vl);  // 重复加载
    // ...
}

// 好：加载一次，多次使用
for (int i = 0; i < n; i += vl) {
    vl = __riscv_vsetvl_e32m1(n - i);
    vint32m1_t v = __riscv_vle32_v_i32m1(data + i, vl);
    vint32m1_t r1 = __riscv_vadd_vx_i32m1(v, 1, vl);
    vint32m1_t r2 = __riscv_vmul_vx_i32m1(r1, 2, vl);  // 使用 r1
    // ...
}
```

#### 3. 使用合适的加载/存储指令

```c
// 连续内存：使用单位步长加载
vint32m1_t v = __riscv_vle32_v_i32m1(ptr, vl);

// 固定步长：使用步长加载
vint32m1_t v = __riscv_vlse32_v_i32m1(ptr, stride * sizeof(int32_t), vl);

// 不规则访问：使用索引加载
vuint32m1_t indices = ...;
vint32m1_t v = __riscv_vluxei32_v_i32m1(base, indices, vl);
```

#### 4. 循环展开

```c
// 未展开
for (int i = 0; i < n; i += vl) {
    vl = __riscv_vsetvl_e32m1(n - i);
    vint32m1_t v = __riscv_vle32_v_i32m1(data + i, vl);
    vint32m1_t r = __riscv_vadd_vx_i32m1(v, 1, vl);
    __riscv_vse32_v_i32m1(out + i, r, vl);
}

// 展开 2 次（如果寄存器足够）
for (int i = 0; i < n - vl; i += vl * 2) {
    vl = __riscv_vsetvl_e32m1(n - i);
    
    vint32m1_t v1 = __riscv_vle32_v_i32m1(data + i, vl);
    vint32m1_t v2 = __riscv_vle32_v_i32m1(data + i + vl, vl);
    
    vint32m1_t r1 = __riscv_vadd_vx_i32m1(v1, 1, vl);
    vint32m1_t r2 = __riscv_vadd_vx_i32m1(v2, 1, vl);
    
    __riscv_vse32_v_i32m1(out + i, r1, vl);
    __riscv_vse32_v_i32m1(out + i + vl, r2, vl);
}
// 处理剩余元素
```

## 4. 优化技巧

### 4.1 编译器优化选项

#### 基本优化级别

```bash
# 无优化（调试用）
riscv32-unknown-elf-gcc -O0 -g program.c -o program.elf

# 基本优化
riscv32-unknown-elf-gcc -O1 program.c -o program.elf

# 推荐的优化级别
riscv32-unknown-elf-gcc -O2 program.c -o program.elf

# 激进优化（可能增加代码大小）
riscv32-unknown-elf-gcc -O3 program.c -o program.elf

# 优化代码大小
riscv32-unknown-elf-gcc -Os program.c -o program.elf
```

**优化级别说明**：

- `-O0`：不优化，编译快，便于调试
- `-O1`：基本优化，不会显著增加编译时间
- `-O2`：推荐级别，包含大部分优化，不会过度增加代码大小
- `-O3`：激进优化，可能增加代码大小，不一定更快
- `-Os`：优化代码大小，适合内存受限的场景

#### 特定优化选项

```bash
# 启用向量化
riscv32-unknown-elf-gcc -O2 -ftree-vectorize program.c

# 启用循环展开
riscv32-unknown-elf-gcc -O2 -funroll-loops program.c

# 启用函数内联
riscv32-unknown-elf-gcc -O2 -finline-functions program.c

# 启用链接时优化（LTO）
riscv32-unknown-elf-gcc -O2 -flto program.c -o program.elf
```

#### 目标架构选项

```bash
# 指定目标架构（包含向量扩展）
riscv32-unknown-elf-gcc -march=rv32imfv -mabi=ilp32f program.c

# 指定 CPU 型号（如果支持）
riscv32-unknown-elf-gcc -mcpu=coralnpu program.c
```

**参数解释**：

- `-march=rv32imfv`：
  - `rv32`：32 位 RISC-V
  - `i`：基本整数指令集
  - `m`：乘法和除法扩展
  - `f`：单精度浮点扩展
  - `v`：向量扩展
- `-mabi=ilp32f`：
  - `ilp32`：int、long、pointer 都是 32 位
  - `f`：使用浮点寄存器传递浮点参数

### 4.2 循环优化

#### 循环不变量外提

```c
// 优化前：每次循环都计算 n * 2
for (int i = 0; i < n; i++) {
    output[i] = input[i] * (n * 2);
}

// 优化后：只计算一次
int factor = n * 2;
for (int i = 0; i < n; i++) {
    output[i] = input[i] * factor;
}
```

#### 循环展开

```c
// 优化前
for (int i = 0; i < n; i++) {
    output[i] = input[i] * 2;
}

// 优化后：展开 4 次
int i;
for (i = 0; i < n - 3; i += 4) {
    output[i]     = input[i]     * 2;
    output[i + 1] = input[i + 1] * 2;
    output[i + 2] = input[i + 2] * 2;
    output[i + 3] = input[i + 3] * 2;
}
// 处理剩余元素
for (; i < n; i++) {
    output[i] = input[i] * 2;
}
```

**好处**：
- 减少循环控制开销（比较、跳转）
- 增加指令级并行性
- 更好地利用流水线

#### 循环融合

```c
// 优化前：两个独立的循环
for (int i = 0; i < n; i++) {
    temp[i] = input[i] * 2;
}
for (int i = 0; i < n; i++) {
    output[i] = temp[i] + 1;
}

// 优化后：融合为一个循环
for (int i = 0; i < n; i++) {
    temp[i] = input[i] * 2;
    output[i] = temp[i] + 1;
}

// 更好：消除临时数组
for (int i = 0; i < n; i++) {
    output[i] = input[i] * 2 + 1;
}
```

### 4.3 内存访问优化

#### 数据对齐

```c
// 确保数据对齐到缓存行（通常是 64 字节）
uint8_t buffer[1024] __attribute__((aligned(64)));
```

#### 顺序访问

```c
// 好：顺序访问，利用缓存
for (int i = 0; i < rows; i++) {
    for (int j = 0; j < cols; j++) {
        sum += matrix[i][cols + j];  // 行优先访问
    }
}

// 不好：跳跃访问，缓存命中率低
for (int j = 0; j < cols; j++) {
    for (int i = 0; i < rows; i++) {
        sum += matrix[i][cols + j];  // 列优先访问
    }
}
```

#### 减少指针解引用

```c
// 优化前：每次都解引用指针
for (int i = 0; i < n; i++) {
    (*ptr)[i] = (*ptr)[i] * 2;
}

// 优化后：使用局部变量
int* p = *ptr;
for (int i = 0; i < n; i++) {
    p[i] = p[i] * 2;
}
```

### 4.4 使用 TCM 提升性能

TCM（Tightly Coupled Memory，紧耦合内存）是直接连接到 CPU 的高速内存，访问延迟极低。

#### ITCM 和 DTCM

- **ITCM**（Instruction TCM）：存放代码
- **DTCM**（Data TCM）：存放数据

#### 将数据放入 DTCM

```c
// 使用 section 属性
uint32_t fast_buffer[1024] __attribute__((section(".data")));

// 或者在链接脚本中指定
```

#### 将代码放入 ITCM

```c
// 使用 section 属性
__attribute__((section(".text")))
void fast_function(void) {
    // 这个函数会被放入 ITCM
}
```

#### 性能对比

```c
// 测试代码
#include "sw/utils/utils.h"

uint32_t data_dtcm[1024] __attribute__((section(".data")));
uint32_t data_sram[1024];  // 在普通 SRAM 中

int main(void) {
    uint64_t start, end;
    
    // 测试 DTCM 访问
    start = mcycle_read();
    for (int i = 0; i < 1024; i++) {
        data_dtcm[i] = i;
    }
    end = mcycle_read();
    uint64_t dtcm_cycles = end - start;
    
    // 测试 SRAM 访问
    start = mcycle_read();
    for (int i = 0; i < 1024; i++) {
        data_sram[i] = i;
    }
    end = mcycle_read();
    uint64_t sram_cycles = end - start;
    
    // dtcm_cycles 应该明显小于 sram_cycles
    
    return 0;
}
```


## 5. 最佳实践

### 5.1 代码风格

良好的代码风格可以提高代码的可读性和可维护性。

#### 命名规范

```c
// 常量：全大写，下划线分隔
#define MAX_BUFFER_SIZE 1024
#define GPIO_BASE_ADDR 0x40030000

// 变量：小写，下划线分隔
int buffer_size;
uint32_t* data_ptr;

// 函数：小写，下划线分隔，动词开头
void init_hardware(void);
int read_sensor_data(uint8_t* buffer, size_t len);

// 类型：首字母大写，驼峰命名（如果使用 typedef）
typedef struct {
    uint32_t address;
    uint32_t size;
} MemoryRegion;
```

#### 注释规范

```c
/**
 * @brief 初始化 UART 外设
 * 
 * @param clock_freq_mhz 时钟频率（MHz）
 * @return 0 表示成功，负数表示错误码
 */
int uart_init(uint32_t clock_freq_mhz) {
    // 配置波特率
    uint32_t divisor = clock_freq_mhz * 1000000 / (16 * 115200);
    REG32(UART_DIVISOR) = divisor;
    
    // 使能发送和接收
    REG32(UART_CONTROL) = UART_TX_EN | UART_RX_EN;
    
    return 0;
}
```

#### 代码格式

```c
// 缩进：使用 2 或 4 个空格（不要用 Tab）
void example_function(void) {
    if (condition) {
        // 代码块
        do_something();
    } else {
        // 另一个代码块
        do_something_else();
    }
}

// 大括号：K&R 风格或 Allman 风格，保持一致
// K&R 风格
if (condition) {
    // ...
}

// Allman 风格
if (condition)
{
    // ...
}

// 行长度：不超过 80 或 100 字符
// 如果太长，适当换行
int result = very_long_function_name(
    parameter1,
    parameter2,
    parameter3
);
```

#### 文件组织

```c
// header.h
#ifndef HEADER_H_
#define HEADER_H_

// 1. 包含的头文件
#include <stdint.h>

// 2. 宏定义
#define BUFFER_SIZE 256

// 3. 类型定义
typedef struct {
    uint32_t field1;
    uint32_t field2;
} MyStruct;

// 4. 函数声明
void init_module(void);
int process_data(uint8_t* data, size_t len);

#endif  // HEADER_H_
```

```c
// source.c
#include "header.h"

// 1. 静态（私有）函数声明
static void internal_helper(void);

// 2. 全局变量定义
uint32_t global_counter = 0;

// 3. 函数实现
void init_module(void) {
    // ...
}

int process_data(uint8_t* data, size_t len) {
    // ...
}

static void internal_helper(void) {
    // ...
}
```

### 5.2 错误处理

#### 返回错误码

```c
// 定义错误码
#define SUCCESS 0
#define ERR_INVALID_PARAM -1
#define ERR_TIMEOUT -2
#define ERR_HARDWARE -3

int read_sensor(uint8_t* buffer, size_t len) {
    // 参数检查
    if (buffer == NULL || len == 0) {
        return ERR_INVALID_PARAM;
    }
    
    // 等待数据就绪
    int timeout = 1000;
    while (!(REG32(SENSOR_STATUS) & DATA_READY)) {
        if (--timeout == 0) {
            return ERR_TIMEOUT;
        }
    }
    
    // 读取数据
    for (size_t i = 0; i < len; i++) {
        buffer[i] = REG32(SENSOR_DATA);
    }
    
    return SUCCESS;
}

// 使用示例
int main(void) {
    uint8_t buffer[16];
    int result = read_sensor(buffer, sizeof(buffer));
    
    if (result != SUCCESS) {
        // 处理错误
        return result;
    }
    
    // 继续处理数据
    return 0;
}
```

#### 使用断言

```c
#include <assert.h>

void process_buffer(uint8_t* buffer, size_t len) {
    // 调试时检查参数
    assert(buffer != NULL);
    assert(len > 0);
    assert(len <= MAX_BUFFER_SIZE);
    
    // 处理数据
    for (size_t i = 0; i < len; i++) {
        buffer[i] = buffer[i] * 2;
    }
}
```

**注意**：`assert` 在发布版本中通常会被禁用（定义 `NDEBUG`），所以不要用它来处理运行时错误。

#### 防御性编程

```c
// 检查指针
if (ptr != NULL) {
    *ptr = value;
}

// 检查数组边界
if (index < array_size) {
    array[index] = value;
}

// 检查除数
if (divisor != 0) {
    result = dividend / divisor;
}

// 初始化变量
int counter = 0;  // 好
int counter;      // 不好，未初始化
```

### 5.3 调试技巧

#### 使用性能计数器

```c
#include "sw/utils/utils.h"

void benchmark_function(void) {
    // 重置计数器
    cycle_counter_reset();
    instrut_counter_reset();
    
    // 开始测量
    uint64_t cycle_start = mcycle_read();
    uint64_t inst_start = minstret_read();
    
    // 执行要测试的代码
    for (int i = 0; i < 1000; i++) {
        // 一些计算
    }
    
    // 结束测量
    uint64_t cycle_end = mcycle_read();
    uint64_t inst_end = minstret_read();
    
    // 计算结果
    uint64_t cycles = cycle_end - cycle_start;
    uint64_t instructions = inst_end - inst_start;
    
    // cycles 和 instructions 可以用于性能分析
}
```

#### 使用 UART 输出调试信息

```c
#include "fpga/sw/uart.h"

int main(void) {
    uart_init(CLOCK_FREQUENCY_MHZ);
    
    uart_puts("Program started\n");
    
    uint32_t value = 0x12345678;
    uart_puts("Value: 0x");
    uart_puthex32(value);
    uart_puts("\n");
    
    // 执行程序逻辑
    int result = do_something();
    
    if (result == 0) {
        uart_puts("Test PASS\n");
    } else {
        uart_puts("Test FAIL\n");
    }
    
    return result;
}
```

#### 使用内联汇编插入断点

```c
void debug_checkpoint(void) {
    // ebreak 指令会触发调试器断点
    asm volatile("ebreak");
}

int main(void) {
    // 初始化
    init_hardware();
    
    // 在关键位置插入断点
    debug_checkpoint();
    
    // 继续执行
    process_data();
    
    return 0;
}
```

#### 使用 volatile 防止优化

```c
// 调试时，防止编译器优化掉某些变量
volatile uint32_t debug_var = 0;

void function(void) {
    debug_var = 1;  // 在调试器中可以看到这个赋值
    
    // 一些代码
    
    debug_var = 2;  // 检查是否执行到这里
}
```

#### 内存转储

```c
void dump_memory(const uint8_t* addr, size_t len) {
    uart_puts("Memory dump:\n");
    
    for (size_t i = 0; i < len; i++) {
        if (i % 16 == 0) {
            uart_puts("\n");
            uart_puthex32((uint32_t)(addr + i));
            uart_puts(": ");
        }
        uart_puthex8(addr[i]);
        uart_puts(" ");
    }
    
    uart_puts("\n");
}

int main(void) {
    uart_init(CLOCK_FREQUENCY_MHZ);
    
    uint8_t buffer[64];
    // 填充数据
    for (int i = 0; i < 64; i++) {
        buffer[i] = i;
    }
    
    // 转储内存内容
    dump_memory(buffer, sizeof(buffer));
    
    return 0;
}
```

### 5.4 性能分析

#### 测量函数执行时间

```c
#include "sw/utils/utils.h"

typedef struct {
    const char* name;
    uint64_t cycles;
    uint64_t instructions;
} BenchmarkResult;

BenchmarkResult benchmark(const char* name, void (*func)(void)) {
    BenchmarkResult result;
    result.name = name;
    
    // 预热（避免缓存冷启动）
    func();
    
    // 测量
    uint64_t start_cycle = mcycle_read();
    uint64_t start_inst = minstret_read();
    
    func();
    
    uint64_t end_cycle = mcycle_read();
    uint64_t end_inst = minstret_read();
    
    result.cycles = end_cycle - start_cycle;
    result.instructions = end_inst - start_inst;
    
    return result;
}

// 使用示例
void scalar_add(void) {
    for (int i = 0; i < 1024; i++) {
        output[i] = input1[i] + input2[i];
    }
}

void vector_add(void) {
    // 向量化版本
    // ...
}

int main(void) {
    BenchmarkResult r1 = benchmark("Scalar Add", scalar_add);
    BenchmarkResult r2 = benchmark("Vector Add", vector_add);
    
    // 比较结果
    // r1.cycles vs r2.cycles
    // r1.instructions vs r2.instructions
    
    return 0;
}
```

#### 计算 IPC（每周期指令数）

```c
double calculate_ipc(uint64_t instructions, uint64_t cycles) {
    if (cycles == 0) return 0.0;
    return (double)instructions / (double)cycles;
}

int main(void) {
    uint64_t start_cycle = mcycle_read();
    uint64_t start_inst = minstret_read();
    
    // 执行代码
    do_work();
    
    uint64_t end_cycle = mcycle_read();
    uint64_t end_inst = minstret_read();
    
    uint64_t cycles = end_cycle - start_cycle;
    uint64_t instructions = end_inst - start_inst;
    
    double ipc = calculate_ipc(instructions, cycles);
    
    // IPC 接近 1.0 表示流水线利用率高
    // IPC 远小于 1.0 可能表示有大量停顿（如内存访问）
    
    return 0;
}
```

#### 识别性能瓶颈

```c
void analyze_performance(void) {
    uint64_t total_cycles = 0;
    uint64_t section_cycles[5];
    
    // 测量各个部分
    uint64_t start = mcycle_read();
    
    // 第 1 部分：数据加载
    load_data();
    section_cycles[0] = mcycle_read() - start;
    start = mcycle_read();
    
    // 第 2 部分：预处理
    preprocess();
    section_cycles[1] = mcycle_read() - start;
    start = mcycle_read();
    
    // 第 3 部分：主计算
    compute();
    section_cycles[2] = mcycle_read() - start;
    start = mcycle_read();
    
    // 第 4 部分：后处理
    postprocess();
    section_cycles[3] = mcycle_read() - start;
    start = mcycle_read();
    
    // 第 5 部分：数据存储
    store_data();
    section_cycles[4] = mcycle_read() - start;
    
    // 计算总周期数
    for (int i = 0; i < 5; i++) {
        total_cycles += section_cycles[i];
    }
    
    // 分析：哪个部分占用时间最多？
    // 优化重点应该放在耗时最多的部分
}
```

#### 优化前后对比

```c
// 优化前的版本
void original_version(void) {
    for (int i = 0; i < 1024; i++) {
        output[i] = input[i] * 2 + 1;
    }
}

// 优化后的版本（使用向量指令）
void optimized_version(void) {
    size_t vl;
    for (size_t i = 0; i < 1024; i += vl) {
        vl = __riscv_vsetvl_e32m1(1024 - i);
        vint32m1_t v = __riscv_vle32_v_i32m1(input + i, vl);
        vint32m1_t r = __riscv_vadd_vx_i32m1(
            __riscv_vmul_vx_i32m1(v, 2, vl), 1, vl);
        __riscv_vse32_v_i32m1(output + i, r, vl);
    }
}

int main(void) {
    // 测量原始版本
    uint64_t start = mcycle_read();
    original_version();
    uint64_t original_cycles = mcycle_read() - start;
    
    // 测量优化版本
    start = mcycle_read();
    optimized_version();
    uint64_t optimized_cycles = mcycle_read() - start;
    
    // 计算加速比
    double speedup = (double)original_cycles / (double)optimized_cycles;
    
    // speedup > 1.0 表示优化有效
    // 例如 speedup = 4.0 表示快了 4 倍
    
    return 0;
}
```

## 6. 常见问题和解决方案

### 6.1 程序无法运行

**问题**：程序编译成功，但在 CoralNPU 上无法运行。

**可能原因和解决方案**：

1. **链接脚本配置错误**
   - 检查代码和数据是否正确放置在 ITCM 和 DTCM 中
   - 确认栈指针初始化正确

2. **未初始化的变量**
   - 确保全局变量正确初始化
   - 检查 `.bss` 段是否被清零

3. **栈溢出**
   - 增加栈大小
   - 减少局部变量的使用
   - 避免深度递归

### 6.2 性能不如预期

**问题**：向量化后性能提升不明显。

**可能原因和解决方案**：

1. **数据未对齐**
   - 使用 `__attribute__((aligned(16)))` 确保数据对齐

2. **向量长度设置不当**
   - 尝试不同的 LMUL 值
   - 确保充分利用向量寄存器

3. **内存访问模式不佳**
   - 优先使用顺序访问
   - 避免频繁的非对齐访问

4. **循环开销过大**
   - 考虑循环展开
   - 减少循环内的分支

### 6.3 编译错误

**问题**：使用向量内置函数时编译失败。

**解决方案**：

```bash
# 确保包含向量扩展
riscv32-unknown-elf-gcc -march=rv32imfv -mabi=ilp32f program.c

# 包含正确的头文件
#include <riscv_vector.h>
```

### 6.4 调试困难

**问题**：程序行为异常，难以定位问题。

**解决方案**：

1. **使用 UART 输出**
   - 在关键位置输出调试信息
   - 输出变量的值

2. **使用断点**
   - 插入 `ebreak` 指令
   - 使用调试器单步执行

3. **检查寄存器状态**
   - 使用内联汇编读取 CSR
   - 检查异常原因寄存器

4. **内存转储**
   - 输出关键数据结构的内容
   - 检查是否有内存越界

## 7. 总结

本章介绍了 CoralNPU 的编程方法，包括：

- **C 编程**：基本结构、硬件寄存器访问、volatile 关键字、内存对齐
- **汇编编程**：汇编函数、混合编程、内联汇编、调用约定
- **向量编程**：向量指令、内置函数、向量化循环、性能优化
- **优化技巧**：编译器选项、循环优化、内存访问优化、TCM 使用
- **最佳实践**：代码风格、错误处理、调试技巧、性能分析

掌握这些技术，你就可以为 CoralNPU 编写高效、可靠的程序了。

## 8. 参考资源

- RISC-V 指令集手册：https://riscv.org/technical/specifications/
- RISC-V 向量扩展规范：https://github.com/riscv/riscv-v-spec
- CoralNPU 示例程序：`examples/` 目录
- CoralNPU 测试程序：`tests/cocotb/` 目录

