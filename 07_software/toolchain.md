# 7.1 工具链

## 7.1.1 工具链概述

### 什么是工具链

工具链（Toolchain）是一组用于开发软件的工具集合。对于嵌入式系统开发来说，工具链通常包括：

- **编译器（Compiler）**：将高级语言（如 C/C++）代码翻译成机器码
- **汇编器（Assembler）**：将汇编语言代码转换成机器码
- **链接器（Linker）**：将多个目标文件和库文件链接成可执行文件
- **调试器（Debugger）**：用于调试程序，查找和修复错误
- **其他工具**：如 objdump（反汇编）、objcopy（格式转换）、ar（创建静态库）等

简单来说，工具链就是把你写的代码变成芯片能执行的程序的一整套工具。

### CoralNPU 使用的工具链

CoralNPU 是基于 RISC-V 架构的处理器，因此使用 RISC-V 工具链。项目支持两种编译器：

1. **RISC-V GCC**：GNU 编译器集合，开源且成熟
2. **RISC-V Clang/LLVM**：现代化的编译器，优化能力强

CoralNPU 项目使用的工具链版本信息：
- 工具链名称：`toolchain_coralnpu_v2`
- 目标架构：`riscv32-unknown-elf`
- 工具链版本：GCC 15.1.0
- 下载地址：https://storage.googleapis.com/shodan-public-artifacts/toolchain_kelvin_tar_files/

工具链的配置文件位于 `/home/curry/code/coralnpu/toolchain/` 目录，包括：
- `cc_toolchain_config.bzl`：Bazel 工具链配置
- `coralnpu_tcm.ld.tpl`：链接脚本模板
- `wrappers/`：工具链包装脚本
- `crt/`：C 运行时启动代码

## 7.1.2 编译器（GCC/Clang）

### RISC-V GCC 基本使用

RISC-V GCC 的命令格式为：

```bash
riscv32-unknown-elf-gcc [选项] 源文件.c -o 输出文件
```

在 CoralNPU 项目中，工具链通过包装脚本调用，实际命令会自动添加必要的参数。

**编译一个简单的 C 程序示例：**

```bash
# 编译单个 C 文件
riscv32-unknown-elf-gcc -march=rv32imf_zve32x_zicsr_zifencei_zbb -mabi=ilp32 \
    -mcmodel=medany hello.c -o hello.elf

# 编译 C++ 文件
riscv32-unknown-elf-g++ -march=rv32imf_zve32x_zicsr_zifencei_zbb -mabi=ilp32 \
    -mcmodel=medany hello.cc -o hello.elf
```

### RISC-V Clang 基本使用

Clang 的使用方式与 GCC 类似：

```bash
clang --target=riscv32-unknown-elf -march=rv32imf_zve32x_zicsr_zifencei_zbb \
    -mabi=ilp32 hello.c -o hello.elf
```

### 编译选项详解

#### 架构选项（-march）

`-march=rv32imf_zve32x_zicsr_zifencei_zbb` 指定了 CoralNPU 支持的指令集扩展：

- **rv32**：32 位 RISC-V 架构
- **i**：基本整数指令集（这是必须的）
- **m**：整数乘法和除法指令
- **f**：单精度浮点指令
- **zve32x**：向量扩展（32 位元素宽度）
- **zicsr**：控制和状态寄存器指令
- **zifencei**：指令同步指令
- **zbb**：位操作基本扩展

**解释超出基本语法的概念：**
- **指令集扩展**：就像给 CPU 增加新功能。基本的 RISC-V 只能做简单的加减法，加上 `m` 扩展后就能做乘除法，加上 `f` 扩展后就能做浮点运算。
- **向量扩展（zve32x）**：允许一条指令同时处理多个数据，比如一次性给 8 个数字都加 1，这样可以大大加快计算速度。

#### ABI 选项（-mabi）

`-mabi=ilp32` 指定应用程序二进制接口（ABI）：

- **ilp32**：整数（int）、长整数（long）、指针（pointer）都是 32 位
- 这个选项决定了函数调用时如何传递参数、如何使用寄存器

**解释 ABI：**
ABI 就像是一套"规矩"，规定了程序之间如何互相调用。比如，当你调用一个函数时，参数放在哪个寄存器里，返回值放在哪里，这些都是 ABI 规定的。

#### 代码模型（-mcmodel）

`-mcmodel=medany` 指定代码模型：

- **medany**：中等代码模型，代码和数据可以放在内存的任意位置
- 适合嵌入式系统，因为内存地址可能不是从 0 开始

### 优化级别

编译器提供不同的优化级别，影响生成代码的速度和大小：

#### -O0（无优化）
```bash
riscv32-unknown-elf-gcc -O0 -g3 hello.c -o hello.elf
```
- 不进行任何优化
- 编译速度最快
- 生成的代码最容易调试
- 代码体积最大，运行速度最慢
- **适用场景**：开发和调试阶段

#### -O1（基本优化）
```bash
riscv32-unknown-elf-gcc -O1 hello.c -o hello.elf
```
- 进行基本的优化
- 编译时间适中
- 代码体积和速度有所改善
- **适用场景**：快速构建，需要一定性能

#### -O2（标准优化）
```bash
riscv32-unknown-elf-gcc -O2 hello.c -o hello.elf
```
- 进行大部分优化，但不增加代码体积
- 这是推荐的优化级别
- **适用场景**：发布版本

#### -O3（激进优化）
```bash
riscv32-unknown-elf-gcc -O3 hello.c -o hello.elf
```
- 进行所有优化，包括可能增加代码体积的优化
- 运行速度最快，但代码体积可能增大
- **适用场景**：性能关键的代码

#### -Os（优化代码大小）
```bash
riscv32-unknown-elf-gcc -Os hello.c -o hello.elf
```
- 优化代码大小，减少内存占用
- 对于内存受限的嵌入式系统很重要
- **适用场景**：内存紧张的场景（如 CoralNPU 的 8KB ITCM）

**CoralNPU 项目的优化配置：**
- **opt 模式**：使用 `-O3`（虽然注释说理想情况应该用 `-Os`）
- **dbg 模式**：使用 `-Og -g3`（优化调试体验）
- **fastbuild 模式**：使用 `-O1 -g3`（快速构建）

### 其他重要编译选项

#### 调试信息
```bash
-g3    # 生成最详细的调试信息
```

#### 警告选项
```bash
-Wall    # 启用所有常见警告
-Werror  # 将警告视为错误
```

#### 函数和数据分段
```bash
-ffunction-sections  # 每个函数放在单独的段中
-fdata-sections      # 每个数据放在单独的段中
```
这两个选项配合链接器的 `--gc-sections` 可以删除未使用的代码，减小程序体积。

#### C++ 特定选项
```bash
-fno-rtti        # 禁用运行时类型信息（减小代码体积）
-fno-exceptions  # 禁用异常处理（减小代码体积）
```

**解释 RTTI 和异常：**
- **RTTI（运行时类型信息）**：C++ 的一个特性，允许程序在运行时判断对象的类型。但这需要额外的内存，嵌入式系统通常不需要。
- **异常处理**：C++ 的 try-catch 机制。异常处理需要额外的代码和栈空间，嵌入式系统通常用返回错误码的方式代替。


## 7.1.3 汇编器

### RISC-V 汇编语法基础

RISC-V 汇编语言是一种低级编程语言，直接对应 CPU 指令。汇编语法的基本格式：

```assembly
指令 目标寄存器, 源寄存器1, 源寄存器2
```

**RISC-V 寄存器：**
- `x0` - `x31`：32 个通用寄存器
  - `x0`：永远是 0（硬件保证）
  - `x1` (ra)：返回地址
  - `x2` (sp)：栈指针
  - `x3` (gp)：全局指针
  - `x8` (s0/fp)：帧指针
  - `x10-x11` (a0-a1)：函数参数和返回值
  - `x10-x17` (a0-a7)：函数参数
- `f0` - `f31`：32 个浮点寄存器
- `v0` - `v31`：32 个向量寄存器

**基本指令示例：**

```assembly
# 整数运算
addi x10, x0, 5      # x10 = 0 + 5 (将 5 加载到 x10)
add  x11, x10, x10   # x11 = x10 + x10 (x11 = 10)
sub  x12, x11, x10   # x12 = x11 - x10 (x12 = 5)
mul  x13, x10, x11   # x13 = x10 * x11 (x13 = 50)

# 内存访问
lw   x10, 0(x2)      # 从地址 x2+0 加载一个字（word）到 x10
sw   x10, 4(x2)      # 将 x10 存储到地址 x2+4

# 浮点运算
fadd.s f10, f11, f12 # f10 = f11 + f12 (单精度浮点加法)
fmul.s f10, f11, f12 # f10 = f11 * f12 (单精度浮点乘法)

# 分支跳转
beq  x10, x11, label # 如果 x10 == x11，跳转到 label
bne  x10, x11, label # 如果 x10 != x11，跳转到 label
j    label           # 无条件跳转到 label
```

**解释汇编基本概念：**
- **寄存器**：CPU 内部的存储单元，速度非常快，但数量有限。就像你的工作台，只能放几样工具。
- **内存**：主存储器，容量大但速度慢。就像你的工具箱，可以放很多东西，但拿取需要时间。
- **立即数**：直接写在指令中的数字，如 `addi x10, x0, 5` 中的 5。
- **标签（label）**：代码中的位置标记，用于跳转。

### 编写汇编代码

#### 独立汇编文件

创建一个 `.S` 文件（注意大写 S）：

```assembly
# add_numbers.S
.section .text
.global add_two_numbers

# 函数：int add_two_numbers(int a, int b)
# 参数：a 在 a0 (x10), b 在 a1 (x11)
# 返回值：在 a0 (x10)
add_two_numbers:
    add  a0, a0, a1    # a0 = a0 + a1
    ret                # 返回（跳转到 ra 寄存器中的地址）
```

在 C 代码中调用：

```c
// main.c
extern int add_two_numbers(int a, int b);

int main() {
    int result = add_two_numbers(3, 5);  // result = 8
    return result;
}
```

编译和链接：

```bash
# 编译汇编文件
riscv32-unknown-elf-gcc -c add_numbers.S -o add_numbers.o

# 编译 C 文件
riscv32-unknown-elf-gcc -c main.c -o main.o

# 链接
riscv32-unknown-elf-gcc add_numbers.o main.o -o program.elf
```

#### 汇编伪指令

汇编器提供一些伪指令（directive），用于控制汇编过程：

```assembly
.section .text        # 指定代码段
.section .data        # 指定数据段
.section .bss         # 指定未初始化数据段

.global symbol_name   # 声明全局符号（可被其他文件访问）
.local symbol_name    # 声明局部符号（仅本文件可见）

.align 4              # 对齐到 4 字节边界
.word 0x12345678      # 定义一个 32 位数据
.byte 0x12            # 定义一个 8 位数据

.ascii "hello"        # 定义 ASCII 字符串（不含结尾 0）
.asciz "hello"        # 定义 ASCII 字符串（含结尾 0）
```

### 内联汇编

内联汇编允许在 C/C++ 代码中直接嵌入汇编指令。

#### 基本语法

```c
asm volatile (
    "汇编指令"
    : 输出操作数
    : 输入操作数
    : 破坏描述符
);
```

**解释内联汇编的各部分：**
- **汇编指令**：要执行的汇编代码
- **输出操作数**：汇编代码的输出，会写入 C 变量
- **输入操作数**：汇编代码的输入，从 C 变量读取
- **破坏描述符**：告诉编译器哪些寄存器或内存被修改了

#### 简单示例

```c
// 读取 RISC-V 的 mhartid 寄存器（硬件线程 ID）
static inline unsigned int read_mhartid() {
    unsigned int hartid;
    asm volatile (
        "csrr %0, mhartid"  // 读取 CSR 寄存器
        : "=r" (hartid)     // 输出：hartid 变量，使用任意寄存器
        :                   // 无输入
        :                   // 无破坏
    );
    return hartid;
}

// 两个数相加（演示输入输出）
static inline int add_asm(int a, int b) {
    int result;
    asm volatile (
        "add %0, %1, %2"    // result = a + b
        : "=r" (result)     // 输出：result
        : "r" (a), "r" (b)  // 输入：a 和 b
        :                   // 无破坏
    );
    return result;
}
```

**操作数约束符：**
- `"=r"`：输出，使用任意通用寄存器
- `"r"`：输入，使用任意通用寄存器
- `"f"`：使用浮点寄存器
- `"m"`：使用内存地址
- `"i"`：立即数（常量）

#### 内存屏障

在多核或有缓存的系统中，需要使用内存屏障确保内存操作的顺序：

```c
// 内存屏障
static inline void memory_barrier() {
    asm volatile ("fence" ::: "memory");
}

// 指令同步屏障
static inline void instruction_barrier() {
    asm volatile ("fence.i" ::: "memory");
}
```

**解释内存屏障：**
现代 CPU 为了提高性能，可能会打乱指令的执行顺序。内存屏障就像一道"栅栏"，确保屏障前的内存操作都完成后，才能执行屏障后的操作。

#### CoralNPU 启动代码示例

CoralNPU 的启动代码 `coralnpu_start.S` 使用汇编编写，负责初始化 CPU：

```assembly
.section ._init, "ax"
.global _start

_start:
    # 设置栈指针
    la sp, __stack_end__
    
    # 清零 BSS 段
    la a0, __bss_start__
    la a1, __bss_end__
    call memset_zero
    
    # 调用 C++ 全局构造函数
    call __libc_init_array
    
    # 调用 main 函数
    call main
    
    # main 返回后，进入无限循环
    j .
```

## 7.1.4 链接器

### 链接过程

链接器（Linker）的作用是将多个目标文件（.o）和库文件（.a）组合成一个可执行文件。链接过程包括：

1. **符号解析**：找到所有函数和变量的定义
2. **重定位**：确定每个符号的最终地址
3. **段合并**：将相同类型的段合并在一起
4. **生成可执行文件**：输出最终的 ELF 文件

**简单理解链接过程：**
假设你写了两个 C 文件，`main.c` 调用了 `utils.c` 中的函数。编译后，`main.o` 只知道"我要调用一个叫 `add` 的函数"，但不知道它在哪里。链接器的工作就是找到 `add` 函数的定义（在 `utils.o` 中），然后把地址填进去。

### 链接脚本（Linker Script）

链接脚本告诉链接器如何组织内存布局。CoralNPU 使用的链接脚本模板是 `coralnpu_tcm.ld.tpl`。

#### 内存区域定义

```ld
MEMORY {
    ITCM(rx): ORIGIN = 0x00000000, LENGTH = 8K
    DTCM(rw): ORIGIN = 0x10000000, LENGTH = 32K
    EXTMEM(rw): ORIGIN = 0x20000000, LENGTH = 4096K
}
```

**解释内存区域：**
- **ITCM（Instruction TCM）**：指令紧耦合内存，用于存放代码，速度快但容量小（8KB）
  - `rx` 表示可读（r）可执行（x）
  - 起始地址 `0x00000000`
- **DTCM（Data TCM）**：数据紧耦合内存，用于存放数据，速度快（32KB）
  - `rw` 表示可读（r）可写（w）
  - 起始地址 `0x10000000`
- **EXTMEM**：外部内存，容量大但速度慢（4MB）
  - 起始地址 `0x20000000`

**TCM 是什么？**
TCM（Tightly Coupled Memory）是紧耦合内存，直接连接到 CPU 核心，访问速度非常快（通常 1 个时钟周期），但容量很小。就像 CPU 的"私人仓库"，放最常用的东西。

#### 段（Section）定义

```ld
SECTIONS {
    .text : {
        *(.text)      # 所有的代码
        *(.text.*)    # 所有以 .text. 开头的段
    } > ITCM          # 放到 ITCM 中
    
    .data : {
        *(.data)      # 已初始化的全局变量
        *(.data.*)
    } > DTCM          # 放到 DTCM 中
    
    .bss : {
        *(.bss)       # 未初始化的全局变量
        *(.bss.*)
    } > DTCM
}
```

**常见的段：**
- `.text`：代码段，存放程序指令
- `.rodata`：只读数据段，存放常量字符串等
- `.data`：已初始化数据段，存放有初值的全局变量
- `.bss`：未初始化数据段，存放没有初值的全局变量（会被初始化为 0）
- `.heap`：堆，用于动态内存分配（malloc）
- `.stack`：栈，用于函数调用和局部变量

**为什么要分段？**
分段可以更好地管理内存。比如，代码段是只读的，可以防止程序意外修改自己的代码；BSS 段不需要在文件中存储，可以减小程序文件的大小。


#### 全局指针（Global Pointer）

CoralNPU 的链接脚本中有一个重要的符号 `__global_pointer$`：

```ld
.data : {
    __global_pointer$ = . + 0x800;
    *(.sdata)
    *(.data)
} > DTCM
```

**全局指针的作用：**
RISC-V 有一个专门的寄存器 `gp`（x3），用于存放全局指针。编译器会把小的全局变量放在 `gp` 附近（±2048 字节范围内），这样访问这些变量只需要一条指令：

```assembly
# 不使用全局指针（需要两条指令）
lui  a0, %hi(variable)      # 加载高位地址
lw   a1, %lo(variable)(a0)  # 加载数据

# 使用全局指针（只需一条指令）
lw   a1, offset(gp)         # 直接从 gp+offset 加载
```

这可以减小代码体积，提高执行速度。

#### 栈和堆的配置

```ld
STACK_SIZE = DEFINED(__stack_size__) ? __stack_size__ : 2048;

.heap : {
    __heap_start__ = .;
    . = ORIGIN(DTCM) + LENGTH(DTCM) - STACK_SIZE;
    __heap_end__ = .;
} > DTCM

.stack : {
    __stack_start__ = .;
    . += STACK_SIZE;
    __stack_end__ = .;
} > DTCM
```

这段脚本配置了栈的大小（默认 2KB），堆从数据段结束后开始，一直到栈的起始位置。

**栈和堆的区别：**
- **栈（Stack）**：自动管理，用于函数调用和局部变量。函数返回时自动释放。栈从高地址向低地址增长。
- **堆（Heap）**：手动管理，用于动态内存分配（malloc/new）。需要手动释放（free/delete）。堆从低地址向高地址增长。

### 指定内存布局

#### 使用 section 属性

在 C/C++ 代码中，可以使用 `__attribute__((section("段名")))` 将变量或函数放到指定的段：

```c
// 将数组放到外部内存
float large_buffer[10000] __attribute__((section(".extdata")));

// 将函数放到 RAM 中执行（用于自修改代码或性能关键代码）
void critical_function() __attribute__((section(".data"))) {
    // 函数代码
}

// 将变量放到特定地址
volatile uint32_t *uart_base __attribute__((section(".peripheral"))) = 
    (uint32_t*)0x40000000;
```

#### 使用链接器选项

```bash
# 指定链接脚本
riscv32-unknown-elf-gcc -T coralnpu_tcm.ld main.o -o program.elf

# 生成 map 文件（显示所有符号的地址）
riscv32-unknown-elf-gcc -Wl,-Map=output.map main.o -o program.elf

# 启用垃圾回收（删除未使用的函数和数据）
riscv32-unknown-elf-gcc -Wl,--gc-sections main.o -o program.elf
```

**Map 文件的作用：**
Map 文件记录了程序中每个函数、变量的地址和大小，可以用来：
- 分析程序的内存占用
- 查找某个符号的地址
- 调试链接问题

#### CoralNPU 的链接脚本生成

CoralNPU 使用 Bazel 规则动态生成链接脚本，可以配置不同的 TCM 大小：

```python
# BUILD.bazel
generate_linker_script(
    name = "coralnpu_tcm_ld",
    src = "coralnpu_tcm.ld.tpl",
    out = "coralnpu_tcm.ld",
    dtcm_size_kbytes = 32,   # DTCM 大小 32KB
    itcm_size_kbytes = 8,    # ITCM 大小 8KB
)

generate_linker_script(
    name = "coralnpu_tcm_highmem_ld",
    src = "coralnpu_tcm.ld.tpl",
    out = "coralnpu_tcm_highmem.ld",
    dtcm_size_kbytes = 1024,  # 高内存配置：1MB DTCM
    itcm_size_kbytes = 1024,  # 高内存配置：1MB ITCM
)
```

## 7.1.5 调试器（GDB）

### RISC-V GDB 基本使用

GDB（GNU Debugger）是一个强大的调试工具，可以：
- 设置断点，单步执行程序
- 查看变量的值
- 查看寄存器和内存
- 分析程序崩溃的原因

#### 启动 GDB

```bash
# 调试本地程序
riscv32-unknown-elf-gdb program.elf

# 连接到远程目标（如 QEMU 或硬件）
riscv32-unknown-elf-gdb program.elf -ex "target remote localhost:1234"
```

### 常用调试命令

#### 运行控制

```gdb
(gdb) run                    # 运行程序
(gdb) continue               # 继续执行（简写：c）
(gdb) step                   # 单步执行，进入函数（简写：s）
(gdb) next                   # 单步执行，不进入函数（简写：n）
(gdb) finish                 # 执行到当前函数返回
(gdb) until                  # 执行到当前循环结束
(gdb) kill                   # 终止程序
(gdb) quit                   # 退出 GDB（简写：q）
```

**step 和 next 的区别：**
- `step`：如果遇到函数调用，会进入函数内部
- `next`：如果遇到函数调用，会直接执行完函数，不进入内部

#### 断点管理

```gdb
(gdb) break main             # 在 main 函数设置断点（简写：b）
(gdb) break file.c:10        # 在 file.c 的第 10 行设置断点
(gdb) break *0x1000          # 在地址 0x1000 设置断点
(gdb) info breakpoints       # 查看所有断点（简写：i b）
(gdb) delete 1               # 删除断点 1（简写：d）
(gdb) disable 1              # 禁用断点 1
(gdb) enable 1               # 启用断点 1
(gdb) clear main             # 删除 main 函数的断点
```

#### 查看变量

```gdb
(gdb) print variable         # 打印变量的值（简写：p）
(gdb) print/x variable       # 以十六进制打印
(gdb) print/t variable       # 以二进制打印
(gdb) print/d variable       # 以十进制打印
(gdb) print array[0]@10      # 打印数组的前 10 个元素
(gdb) display variable       # 每次停下来都自动打印变量
(gdb) undisplay 1            # 取消自动打印
(gdb) info locals            # 查看所有局部变量
(gdb) info args              # 查看函数参数
```

#### 查看寄存器

```gdb
(gdb) info registers         # 查看所有通用寄存器（简写：i r）
(gdb) info all-registers     # 查看所有寄存器（包括浮点、向量）
(gdb) print $a0              # 打印 a0 寄存器的值
(gdb) print $pc              # 打印程序计数器（当前指令地址）
(gdb) print $sp              # 打印栈指针
(gdb) set $a0 = 10           # 修改寄存器的值
```

**RISC-V 寄存器在 GDB 中的名字：**
- 可以用 `$x0` - `$x31` 或 `$zero`, `$ra`, `$sp`, `$gp`, `$a0` 等别名
- 浮点寄存器：`$f0` - `$f31` 或 `$fa0`, `$fs0` 等
- 特殊寄存器：`$pc`（程序计数器）

#### 查看内存

```gdb
(gdb) x/10x 0x10000000       # 以十六进制查看地址 0x10000000 的 10 个字
(gdb) x/10i $pc              # 反汇编当前位置的 10 条指令
(gdb) x/s 0x10000000         # 以字符串形式查看内存
(gdb) x/10b 0x10000000       # 以字节形式查看 10 个字节
```

**x 命令的格式：**
```
x/[数量][格式][单位] 地址
```
- 数量：要显示多少个单位
- 格式：x（十六进制）、d（十进制）、i（指令）、s（字符串）、t（二进制）
- 单位：b（字节）、h（半字，2 字节）、w（字，4 字节）、g（巨字，8 字节）

#### 查看调用栈

```gdb
(gdb) backtrace              # 查看调用栈（简写：bt）
(gdb) frame 1                # 切换到栈帧 1（简写：f）
(gdb) up                     # 向上移动一个栈帧
(gdb) down                   # 向下移动一个栈帧
(gdb) info frame             # 查看当前栈帧的详细信息
```

**调用栈是什么？**
调用栈记录了函数的调用关系。比如 `main` 调用了 `foo`，`foo` 调用了 `bar`，那么调用栈就是：
```
#0  bar()
#1  foo()
#2  main()
```
这对于追踪程序的执行流程和查找 bug 非常有用。

#### 反汇编

```gdb
(gdb) disassemble main       # 反汇编 main 函数（简写：disas）
(gdb) disassemble $pc        # 反汇编当前位置
(gdb) disassemble 0x1000,0x1100  # 反汇编地址范围
```

### 远程调试

远程调试允许在一台机器上运行 GDB，调试另一台机器（或模拟器）上的程序。

#### 使用 QEMU 进行远程调试

1. 启动 QEMU，开启 GDB 服务器：
```bash
qemu-system-riscv32 -machine virt -nographic -bios none \
    -kernel program.elf -s -S
```
- `-s`：在 1234 端口开启 GDB 服务器
- `-S`：启动时暂停，等待 GDB 连接

2. 在另一个终端启动 GDB：
```bash
riscv32-unknown-elf-gdb program.elf
(gdb) target remote localhost:1234
(gdb) break main
(gdb) continue
```

#### 使用 OpenOCD 调试硬件

OpenOCD（Open On-Chip Debugger）可以通过 JTAG/SWD 接口调试真实硬件：

1. 启动 OpenOCD：
```bash
openocd -f interface/ftdi/olimex-arm-usb-tiny-h.cfg \
        -f target/riscv32.cfg
```

2. 连接 GDB：
```bash
riscv32-unknown-elf-gdb program.elf
(gdb) target remote localhost:3333
(gdb) monitor reset halt
(gdb) load
(gdb) break main
(gdb) continue
```

**JTAG 是什么？**
JTAG 是一种硬件调试接口，通过几根线连接调试器和芯片，可以：
- 暂停和恢复 CPU
- 读写寄存器和内存
- 设置断点
- 烧写程序到 Flash


### GDB 调试技巧

#### 条件断点

```gdb
# 只有当 i == 10 时才触发断点
(gdb) break loop.c:15 if i == 10

# 修改已有断点的条件
(gdb) condition 1 i == 10
```

#### 观察点（Watchpoint）

观察点可以监视变量的变化，当变量被修改时自动暂停：

```gdb
(gdb) watch variable         # 当 variable 被写入时暂停
(gdb) rwatch variable        # 当 variable 被读取时暂停
(gdb) awatch variable        # 当 variable 被读或写时暂停
```

#### 保存和恢复断点

```gdb
(gdb) save breakpoints bp.txt   # 保存断点到文件
(gdb) source bp.txt              # 从文件加载断点
```

#### 调试优化后的代码

优化后的代码可能难以调试（变量被优化掉、代码顺序改变等）。可以使用：

```bash
# 编译时使用 -Og（优化调试体验）
riscv32-unknown-elf-gcc -Og -g3 main.c -o program.elf
```

#### 使用 GDB 脚本

可以编写 GDB 脚本自动化调试任务：

```gdb
# debug.gdb
target remote localhost:1234
break main
commands
  silent
  printf "Entering main, a0=%d\n", $a0
  continue
end
continue
```

使用脚本：
```bash
riscv32-unknown-elf-gdb program.elf -x debug.gdb
```

## 7.1.6 构建脚本

CoralNPU 项目使用 Bazel 作为构建系统。Bazel 是 Google 开发的构建工具，支持大规模项目的快速增量构建。

### Bazel 基础概念

**什么是 Bazel？**
Bazel 是一个构建工具，类似于 Make、CMake。它的特点是：
- 快速：只重新编译修改过的文件
- 可重现：相同的输入总是产生相同的输出
- 支持多语言：C/C++、Python、Java、Scala 等

**基本概念：**
- **WORKSPACE**：项目根目录的标识文件，定义外部依赖
- **BUILD.bazel**：定义构建规则的文件，每个目录可以有一个
- **Target**：构建目标，如一个可执行文件或库
- **Rule**：构建规则，定义如何构建目标

### CoralNPU 的 Bazel 构建规则

#### coralnpu_v2_binary 规则

这是 CoralNPU 项目定义的自定义规则，用于构建 RISC-V 可执行文件：

```python
# examples/BUILD.bazel
load("//rules:coralnpu_v2.bzl", "coralnpu_v2_binary")

coralnpu_v2_binary(
    name = "coralnpu_v2_hello_world_add_floats",
    srcs = ["hello_world_add_floats.cc"],
)
```

这个规则会自动：
- 使用正确的工具链（riscv32-unknown-elf-gcc）
- 添加必要的编译选项（-march、-mabi 等）
- 链接必要的库（newlib、libgcc 等）
- 使用正确的链接脚本

#### 编译选项配置

CoralNPU 的工具链配置在 `toolchain/cc_toolchain_config.bzl` 中，定义了：

**架构选项：**
```python
architecture_flag_set = flag_set(
    actions = all_compile_actions,
    flag_groups = [
        flag_group(
            flags = [
                "-march=rv32imf_zve32x_zicsr_zifencei_zbb",
                "-mabi=ilp32",
                "-mcmodel=medany",
                "-nostdlib",
            ],
        ),
    ],
)
```

**优化选项：**
```python
# opt 模式（-c opt）
opt_compile_flag_set = flag_set(
    flags = [
        "-O3",
        "-ffunction-sections",
        "-fdata-sections",
    ],
)

# dbg 模式（-c dbg）
dbg_flag_set = flag_set(
    flags = [
        "-g3",
        "-Og",
    ],
)

# fastbuild 模式（-c fastbuild，默认）
fastbuild_flag_set = flag_set(
    flags = [
        "-g3",
        "-O1",
    ],
)
```

**警告选项：**
```python
warnings_feature = feature(
    name = "warnings",
    enabled = True,
    flag_sets = [
        flag_set(
            flags = [
                "-Wall",    # 启用所有警告
                "-Werror",  # 将警告视为错误
            ],
        ),
    ],
)
```

### 如何编译程序

#### 编译单个目标

```bash
# 编译 hello world 示例
bazel build //examples:coralnpu_v2_hello_world_add_floats

# 编译优化版本
bazel build -c opt //examples:coralnpu_v2_hello_world_add_floats

# 编译调试版本
bazel build -c dbg //examples:coralnpu_v2_hello_world_add_floats
```

**Bazel 目标的格式：**
```
//路径:目标名
```
- `//` 表示从 WORKSPACE 根目录开始
- `examples` 是目录路径
- `:coralnpu_v2_hello_world_add_floats` 是目标名（在 BUILD.bazel 中定义）

#### 查看编译输出

编译后的文件在 `bazel-bin` 目录下：

```bash
# 查看生成的 ELF 文件
ls -lh bazel-bin/examples/coralnpu_v2_hello_world_add_floats

# 查看文件信息
file bazel-bin/examples/coralnpu_v2_hello_world_add_floats

# 查看段信息
riscv32-unknown-elf-size bazel-bin/examples/coralnpu_v2_hello_world_add_floats

# 反汇编
riscv32-unknown-elf-objdump -d bazel-bin/examples/coralnpu_v2_hello_world_add_floats
```

#### 清理构建输出

```bash
# 清理所有构建输出
bazel clean

# 清理所有输出，包括下载的依赖
bazel clean --expunge
```

#### 查看构建依赖

```bash
# 查看目标的依赖关系
bazel query --output=graph //examples:coralnpu_v2_hello_world_add_floats

# 查看目标的构建规则
bazel query --output=build //examples:coralnpu_v2_hello_world_add_floats
```

### 添加自定义编译选项

#### 在 BUILD.bazel 中添加选项

```python
coralnpu_v2_binary(
    name = "my_program",
    srcs = ["main.cc"],
    copts = [
        "-O3",           # 额外的编译选项
        "-DDEBUG=1",     # 定义宏
    ],
    linkopts = [
        "-Wl,--print-memory-usage",  # 链接选项
    ],
)
```

#### 使用不同的链接脚本

```python
coralnpu_v2_binary(
    name = "my_program_highmem",
    srcs = ["main.cc"],
    linker_script = "//toolchain:coralnpu_tcm_highmem_ld",
)
```

### 构建多个目标

```bash
# 构建整个目录下的所有目标
bazel build //examples/...

# 构建所有目标
bazel build //...

# 并行构建（使用 8 个线程）
bazel build --jobs=8 //examples/...
```

### Semihosting 支持

CoralNPU 支持 semihosting，允许程序通过调试器进行 I/O 操作（如 printf 输出到主机终端）：

```python
coralnpu_v2_binary(
    name = "my_program_semihosting",
    srcs = ["main.cc"],
    target_compatible_with = [
        "//platforms/cpu:coralnpu_v2",
        "//platforms/os:semihosting",
    ],
)
```

**Semihosting 是什么？**
Semihosting 是一种机制，允许嵌入式程序通过调试器访问主机的资源。比如：
- `printf` 可以输出到主机的终端
- 可以读写主机的文件
- 可以获取主机的时间

这对于调试非常有用，但在实际硬件上运行时通常不可用。

### 工具链包装脚本

CoralNPU 使用包装脚本（`toolchain/wrappers/driver.sh`）来调用实际的工具链：

```bash
#!/bin/bash
PROG=$(basename "$0")
TOOLCHAIN="toolchain_coralnpu_v2"
PREFIX="riscv32-unknown-elf"

exec "external/${TOOLCHAIN}/bin/${PREFIX}-${PROG}" "$@"
```

这个脚本的作用是：
1. 获取被调用的程序名（如 gcc、ld）
2. 找到实际的工具链路径
3. 调用对应的工具（如 riscv32-unknown-elf-gcc）

**为什么需要包装脚本？**
包装脚本可以：
- 统一工具链的路径管理
- 在调用工具前添加额外的参数
- 支持不同的构建环境（如 Docker）

## 7.1.7 工具链的安装和配置

### 下载预编译工具链

CoralNPU 项目使用预编译的工具链，在 WORKSPACE 文件中自动下载：

```python
http_archive(
    name = "toolchain_coralnpu_v2",
    sha256 = "c9c85f8361e9d02d64474c51e3b3730ba09807cf4610d6d002c49a270458b49c",
    strip_prefix = "toolchain_kelvin_v2",
    urls = [
        "https://storage.googleapis.com/shodan-public-artifacts/toolchain_kelvin_tar_files/toolchain_kelvin_v2-2025-09-11.tar.gz",
    ],
)
```

第一次运行 `bazel build` 时，Bazel 会自动下载工具链。

### 手动安装工具链

如果需要手动安装 RISC-V 工具链：

#### 从包管理器安装（Ubuntu/Debian）

```bash
sudo apt-get install gcc-riscv64-unknown-elf
```

注意：这个包是 64 位的，CoralNPU 需要 32 位工具链。

#### 从源码编译

```bash
# 安装依赖
sudo apt-get install autoconf automake autotools-dev curl python3 \
    libmpc-dev libmpfr-dev libgmp-dev gawk build-essential bison \
    flex texinfo gperf libtool patchutils bc zlib1g-dev libexpat-dev

# 下载 RISC-V GNU 工具链
git clone https://github.com/riscv/riscv-gnu-toolchain
cd riscv-gnu-toolchain

# 配置（32 位，newlib）
./configure --prefix=/opt/riscv32 --with-arch=rv32imf --with-abi=ilp32

# 编译（需要很长时间）
make -j$(nproc)

# 添加到 PATH
export PATH=/opt/riscv32/bin:$PATH
```

### 验证工具链安装

```bash
# 检查 GCC 版本
riscv32-unknown-elf-gcc --version

# 检查支持的架构
riscv32-unknown-elf-gcc -march=help

# 编译一个简单的程序
echo 'int main() { return 0; }' > test.c
riscv32-unknown-elf-gcc -march=rv32imf test.c -o test.elf
file test.elf
```

## 7.1.8 常见问题和解决方法

### 编译错误

#### 错误：undefined reference to `__global_pointer$`

**原因：** 链接脚本中没有定义全局指针。

**解决：** 确保使用正确的链接脚本，或在链接脚本中添加：
```ld
__global_pointer$ = . + 0x800;
```

#### 错误：region `ITCM' overflowed

**原因：** 代码太大，超过了 ITCM 的容量（8KB）。

**解决方法：**
1. 使用 `-Os` 优化代码大小
2. 将部分代码放到外部内存：
   ```c
   void large_function() __attribute__((section(".extdata"))) {
       // 函数代码
   }
   ```
3. 使用高内存配置的链接脚本（1MB ITCM）

#### 错误：unrecognized opcode

**原因：** 使用了工具链不支持的指令。

**解决：** 检查 `-march` 选项是否包含所需的扩展。

### 链接错误

#### 错误：undefined reference to `memcpy`

**原因：** 缺少 C 标准库。

**解决：** 确保链接了 newlib：
```bash
riscv32-unknown-elf-gcc main.o -lc -lgcc -o program.elf
```

#### 错误：multiple definition of `main`

**原因：** 多个文件中定义了 `main` 函数。

**解决：** 确保只有一个文件包含 `main` 函数。

### 调试问题

#### GDB 无法连接到远程目标

**检查：**
1. 目标是否在运行？
2. 端口是否正确？（默认 1234 或 3333）
3. 防火墙是否阻止了连接？

```bash
# 测试端口是否开放
telnet localhost 1234
```

#### GDB 显示 "No symbol table is loaded"

**原因：** 程序没有调试信息。

**解决：** 使用 `-g` 或 `-g3` 编译：
```bash
riscv32-unknown-elf-gcc -g3 main.c -o program.elf
```

### 性能问题

#### 程序运行很慢

**检查：**
1. 是否使用了优化选项？（-O2 或 -O3）
2. 是否启用了指令缓存？
3. 热点代码是否在 TCM 中？

**分析性能：**
```bash
# 使用 gprof 进行性能分析
riscv32-unknown-elf-gcc -pg main.c -o program.elf
# 运行程序后会生成 gmon.out
riscv32-unknown-elf-gprof program.elf gmon.out
```

## 7.1.9 总结

本章介绍了 CoralNPU 的工具链，包括：

1. **编译器**：RISC-V GCC/Clang，支持 rv32imf_zve32x_zicsr_zifencei_zbb 指令集
2. **汇编器**：支持 RISC-V 汇编语法和内联汇编
3. **链接器**：使用链接脚本管理内存布局（ITCM、DTCM、EXTMEM）
4. **调试器**：RISC-V GDB，支持远程调试
5. **构建系统**：Bazel，支持快速增量构建

工具链是嵌入式开发的基础，熟练掌握工具链的使用可以大大提高开发效率。

## 7.1.10 参考资源

- RISC-V 指令集手册：https://riscv.org/technical/specifications/
- RISC-V GCC 文档：https://gcc.gnu.org/onlinedocs/
- GDB 用户手册：https://sourceware.org/gdb/documentation/
- Bazel 文档：https://bazel.build/
- Newlib 文档：https://sourceware.org/newlib/
- CoralNPU 工具链源码：`/home/curry/code/coralnpu/toolchain/`
