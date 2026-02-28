# 7.2 运行时

本节介绍 CoralNPU 的 C 运行时（CRT）和程序启动过程。运行时是连接硬件和应用程序的桥梁，负责在程序开始执行前完成必要的初始化工作。

## 7.2.1 C 运行时（CRT）

### 什么是 C 运行时

C 运行时（C Runtime，简称 CRT）是一组在程序启动时执行的代码，它的主要任务是：

1. 初始化处理器状态
2. 设置栈指针和全局指针
3. 清零 BSS 段（未初始化的全局变量）
4. 初始化 DATA 段（已初始化的全局变量）
5. 调用 C++ 构造函数（如果有）
6. 调用 main 函数
7. 处理 main 函数返回后的清理工作

在嵌入式系统中，CRT 通常由汇编语言编写，因为此时 C 语言运行环境还未建立。

### CoralNPU 的 CRT 实现

CoralNPU 的 CRT 由以下文件组成：

- `coralnpu_start.S` - 主启动代码，包含 `_start` 入口点
- `crt.S` - 工具函数，用于清零和复制内存段
- `coralnpu_exceptions.cc` - 默认的异常处理函数

## 7.2.2 启动代码详解

### 复位向量

当处理器复位或上电时，会从 `_start` 符号开始执行。这个符号在链接脚本中被指定为程序的入口点：

```asm
.section ._init
.balign 4
.global _start
.type _start, @function
_start:
    # 启动代码从这里开始
```

**解释**：
- `.section ._init`：这是一个汇编指令（directive），告诉汇编器把后面的代码放到名为 `._init` 的段（section）中
- `.balign 4`：对齐到 4 字节边界（RISC-V 指令必须 4 字节对齐）
- `.global _start`：声明 `_start` 是全局符号，可以被链接器看到
- `.type _start, @function`：告诉汇编器 `_start` 是一个函数

### 初始化性能计数器

```asm
csrw minstret, 0
csrw minstreth, 0
csrr a0, minstret
csrr a1, minstreth
```

**解释**：
- `csrw`：CSR Write，写入控制状态寄存器（Control and Status Register）
- `csrr`：CSR Read，读取控制状态寄存器
- `minstret` 和 `minstreth`：指令退休计数器（低 32 位和高 32 位）
- 这段代码将计数器清零，用于性能测量

### 初始化寄存器

```asm
.option norelax
la   sp, __stack_end__
la   gp, _global_pointer
.option relax

mv   tp, zero
mv   t1, zero
# ... 更多寄存器初始化
```

**解释**：
- `la`：Load Address，加载地址到寄存器（这是一个伪指令）
- `sp`：栈指针（Stack Pointer），指向栈顶
- `gp`：全局指针（Global Pointer），用于访问全局变量
- `tp`：线程指针（Thread Pointer）
- `.option norelax` 和 `.option relax`：控制链接器优化行为
- `mv reg, zero`：将寄存器清零（`mv` 是 move 的缩写）

### 清零 BSS 段

```asm
la   a0, __bss_start__
la   a1, __bss_end__
call crt_section_clear
```

**BSS 段**（Block Started by Symbol）存放未初始化的全局变量和静态变量。C 语言标准要求这些变量在程序启动时被初始化为 0。

`crt_section_clear` 函数的实现（在 `crt.S` 中）：

```asm
crt_section_clear:
    bgeu a0, a1, .L_clear_nothing  # 如果 start >= end，跳转
    
    # 检查地址是否 4 字节对齐
    or   t0, a0, a1
    andi t0, t0, 0x3
    bnez t0, .L_clear_error
    
.L_clear_loop:
    sw   zero, 0(a0)    # 写入 0
    addi a0, a0, 4      # 地址加 4
    bltu a0, a1, .L_clear_loop  # 如果 a0 < a1，继续循环
    ret
```

**解释**：
- `bgeu`：Branch if Greater or Equal Unsigned（无符号大于等于则跳转）
- `or` 和 `andi`：用于检查地址对齐
- `sw zero, 0(a0)`：将 0 写入 a0 指向的内存地址
- `addi`：Add Immediate（立即数加法）
- `bltu`：Branch if Less Than Unsigned（无符号小于则跳转）

### 运行构造函数

```asm
la   s0, __init_array_start__
la   s1, __init_array_end__
bgeu s0, s1, init_array_loop_end
init_array_loop:
    lw   t0, 0(s0)      # 加载函数指针
    jalr t0             # 调用函数
    addi s0, s0, 0x4    # 移动到下一个函数指针
    bltu s0, s1, init_array_loop
init_array_loop_end:
```

**解释**：
- `.init_array` 段存放 C++ 全局对象的构造函数指针
- `lw`：Load Word（加载 32 位字）
- `jalr`：Jump And Link Register（跳转到寄存器指定的地址，并保存返回地址）
- 这段代码遍历所有构造函数并依次调用

### 设置异常处理

```asm
la t0, coralnpu_exception_handler
csrw mtvec, t0
```

**解释**：
- `mtvec`：Machine Trap Vector，机器模式陷阱向量寄存器
- 当发生异常或中断时，处理器会跳转到 `mtvec` 指定的地址
- `coralnpu_exception_handler` 是默认的异常处理函数

### 设置浮点和向量状态

```asm
li      t0, 0x6600
csrrs   zero, mstatus, t0
```

**解释**：
- `li`：Load Immediate（加载立即数）
- `csrrs`：CSR Read and Set（读取 CSR 并设置指定的位）
- `mstatus`：机器状态寄存器
- `0x6600` 设置 FS（浮点状态）和 VS（向量状态）为 Dirty，表示这些单元已启用

### 调用 main 函数

```asm
li   a0, 0  # argv
li   a1, 0  # argc
la   ra, main
jalr ra, ra
mv   s2, a0  # 保存返回值
```

**解释**：
- `a0` 和 `a1` 是函数参数寄存器，分别传递 `argc` 和 `argv`
- `ra`：Return Address（返回地址寄存器）
- `jalr ra, ra`：跳转到 main 函数
- `mv s2, a0`：保存 main 的返回值到 s2 寄存器

### 运行析构函数和清理

```asm
call __cxa_finalize

la   s0, __fini_array_start__
la   s1, __fini_array_end__
beq  s0, s1, fini_array_loop_end
fini_array_loop:
    addi s1, s1, -0x4   # 从后往前遍历
    lw   t0, 0(s1)
    jalr t0
    bne  s0, s1, fini_array_loop
fini_array_loop_end:
```

**解释**：
- `__cxa_finalize`：C++ 运行时函数，用于清理
- `.fini_array` 段存放析构函数指针
- 析构函数按照与构造函数相反的顺序调用

### 程序结束

```asm
mv   a0, s2
la   t0, _ret
sw   a0, 0(t0)
beqz a0, success
failure:
    ebreak
    j    loop
success:
    csrr a0, minstret
    csrr a1, minstreth
    .word 0x08000073  # mpause
loop:
    j    loop
```

**解释**：
- 将 main 的返回值存储到 `_ret` 内存位置
- `beqz`：Branch if Equal to Zero（等于零则跳转）
- `ebreak`：触发断点异常
- `j loop`：无限循环（嵌入式系统通常不会"退出"）

## 7.2.3 内存布局

CoralNPU 使用 TCM（Tightly Coupled Memory）架构，将内存分为指令 TCM（ITCM）和数据 TCM（DTCM）。

### 内存区域

```
MEMORY {
    ITCM(rx): ORIGIN = 0x00000000, LENGTH = 256K
    DTCM(rw): ORIGIN = 0x10000000, LENGTH = 256K
    EXTMEM(rw): ORIGIN = 0x20000000, LENGTH = 4096K
}
```

**解释**：
- `ITCM`：指令紧耦合内存，只读可执行（rx = read + execute）
- `DTCM`：数据紧耦合内存，可读写（rw = read + write）
- `EXTMEM`：外部内存，用于大数据存储

### 内存布局图

```
ITCM (0x00000000 - 0x0003FFFF)
+---------------------------+
|      ._init (启动代码)     |  <- _start 入口点
+---------------------------+
|      .text (代码段)        |  <- 程序代码
|                           |
+---------------------------+
|   .init_array (构造函数)   |
+---------------------------+
|   .fini_array (析构函数)   |
+---------------------------+
|    .rodata (只读数据)      |  <- 字符串常量等
+---------------------------+

DTCM (0x10000000 - 0x1003FFFF)
+---------------------------+
|    .tdata (线程局部数据)   |
+---------------------------+
|    .tbss (线程局部BSS)     |
+---------------------------+
|    .data (数据段)          |  <- 已初始化全局变量
|                           |
|    _global_pointer (gp)   |  <- 全局指针位置
|                           |
|    _ret (返回值)           |
+---------------------------+
|    .bss (BSS段)           |  <- 未初始化全局变量
|                           |
+---------------------------+
|    .heap (堆)             |  <- 动态内存分配
|         |                 |
|         v                 |
|                           |
|         ^                 |
|         |                 |
|    .stack (栈)            |  <- 函数调用栈
+---------------------------+  <- __stack_end__ (sp 初始值)

EXTMEM (0x20000000 - 0x203FFFFF)
+---------------------------+
|    .extdata (外部数据)     |
+---------------------------+
|    .extbss (外部BSS)       |
+---------------------------+
```

### 各段的作用

#### .text 段（代码段）

存放程序的机器指令。这个段是只读的，防止程序意外修改自己的代码。

```c
int add(int a, int b) {
    return a + b;  // 这个函数的机器码存放在 .text 段
}
```

#### .rodata 段（只读数据段）

存放只读数据，如字符串常量。

```c
const char* message = "Hello, CoralNPU!";  // 字符串存放在 .rodata
const int max_value = 100;                 // 常量也在 .rodata
```

#### .data 段（数据段）

存放已初始化的全局变量和静态变量。

```c
int global_counter = 42;           // 存放在 .data 段
static int module_state = 1;       // 也在 .data 段
```

**注意**：这些变量的初始值存储在 ITCM 中（因为 ITCM 是非易失性的），程序启动时需要将它们复制到 DTCM。

#### .bss 段（未初始化数据段）

存放未初始化的全局变量和静态变量。

```c
int buffer[1024];                  // 存放在 .bss 段
static char temp_data[256];        // 也在 .bss 段
```

**注意**：BSS 段不占用程序文件空间，只在内存中分配。启动时会被清零。

#### 栈（Stack）

栈用于函数调用和局部变量。栈从高地址向低地址增长。

```c
void function() {
    int local_var = 10;    // 存放在栈上
    char buffer[100];      // 也在栈上
}
```

**栈的特点**：
- 自动管理：函数返回时自动释放
- 速度快：访问速度快
- 大小有限：由链接脚本中的 `STACK_SIZE` 定义

#### 堆（Heap）

堆用于动态内存分配（malloc/free）。堆从低地址向高地址增长。

```c
int* ptr = (int*)malloc(100 * sizeof(int));  // 从堆分配内存
free(ptr);  // 释放内存
```

### 全局指针优化

```c
_global_pointer = . + 0x800;
__global_pointer$ = . + 0x800;
```

**解释**：
- RISC-V 使用 `gp` 寄存器存储全局指针
- 链接器可以使用 `gp` 相对寻址访问 `gp ± 2048` 范围内的全局变量
- 这比使用绝对地址更高效（节省一条指令）

例如：
```asm
# 不使用 gp 优化（需要 2 条指令）
lui  a0, %hi(global_var)
lw   a0, %lo(global_var)(a0)

# 使用 gp 优化（只需 1 条指令）
lw   a0, offset(gp)
```

## 7.2.4 链接脚本

### 链接脚本的作用

链接脚本（Linker Script）告诉链接器如何组织程序的各个部分。它的主要作用是：

1. 定义内存布局
2. 指定各段的位置
3. 定义符号（如 `__bss_start__`）
4. 控制段的对齐

### CoralNPU 链接脚本示例

链接脚本使用特殊的语法，下面逐部分解释。

#### 定义内存区域

```ld
MEMORY {
    ITCM(rx): ORIGIN = 0x00000000, LENGTH = 256K
    DTCM(rw): ORIGIN = 0x10000000, LENGTH = 256K
    EXTMEM(rw): ORIGIN = 0x20000000, LENGTH = 4096K
}
```

**解释**：
- `MEMORY` 命令定义可用的内存区域
- `ORIGIN`：内存区域的起始地址
- `LENGTH`：内存区域的大小（K = 1024 字节）
- `(rx)` 和 `(rw)`：访问权限标志

#### 定义栈大小

```ld
STACK_SIZE = DEFINED(__stack_size__) ? __stack_size__ : 4K;
```

**解释**：
- 如果用户定义了 `__stack_size__`，使用用户的值
- 否则使用默认值 4K
- 这是一个三元运算符：`条件 ? 真值 : 假值`

#### 指定入口点

```ld
ENTRY(_start)
```

**解释**：
- 告诉链接器程序的入口点是 `_start` 符号
- 调试器和加载器会使用这个信息

#### 定义段布局

```ld
SECTIONS {
    . = ORIGIN(ITCM);
    .text : ALIGN(16) {
        *(._init)
        *(.text)
        *(.text.*)
        . = ALIGN(16);
    } > ITCM
}
```

**解释**：
- `SECTIONS` 命令定义输出段的布局
- `. = ORIGIN(ITCM)`：设置当前位置为 ITCM 起始地址
- `.text : ALIGN(16)`：定义 .text 段，16 字节对齐
- `*(._init)`：包含所有输入文件的 ._init 段
- `*(.text)`：包含所有输入文件的 .text 段
- `*(.text.*)`：包含所有 .text.* 段（如 .text.startup）
- `> ITCM`：将这个段放到 ITCM 内存区域

#### 定义符号

```ld
.bss : ALIGN(16) {
    __bss_start__ = .;
    *(.sbss)
    *(.sbss.*)
    *(.bss)
    *(.bss.*)
    __bss_end__ = .;
} > DTCM
```

**解释**：
- `__bss_start__ = .`：定义符号 `__bss_start__`，值为当前位置
- `.` 是一个特殊符号，表示当前地址
- 启动代码使用这些符号来清零 BSS 段

#### 堆和栈的定义

```ld
.heap : ALIGN(16) {
    __heap_start__ = .;
    . = ORIGIN(DTCM) + LENGTH(DTCM) - STACK_SIZE;
    __heap_end__ = .;
} > DTCM

.stack : ALIGN(16) {
    __stack_start__ = .;
    . += STACK_SIZE;
    __stack_end__ = .;
} > DTCM
```

**解释**：
- 堆从 BSS 段结束后开始，一直到栈开始前
- `. = ORIGIN(DTCM) + LENGTH(DTCM) - STACK_SIZE`：计算堆的结束位置
- `. += STACK_SIZE`：为栈分配空间
- 栈在 DTCM 的最高地址处

### 如何使用链接脚本

编译时使用 `-T` 选项指定链接脚本：

```bash
riscv32-unknown-elf-gcc \
    -T coralnpu.ld \
    -o program.elf \
    main.c startup.S
```

### 自定义段

你可以将特定的变量或函数放到自定义段：

```c
// 将变量放到外部内存
int large_buffer[10000] __attribute__((section(".extdata")));

// 将函数放到特定段
void critical_function() __attribute__((section(".text.critical"))) {
    // 关键代码
}
```

链接脚本中处理：

```ld
.text : {
    *(.text.critical)  # 关键代码放在前面
    *(.text)
    *(.text.*)
} > ITCM
```

## 7.2.5 异常和中断处理

### 异常向量表

RISC-V 使用 `mtvec` 寄存器指向异常处理入口。CoralNPU 在启动时设置：

```asm
la t0, coralnpu_exception_handler
csrw mtvec, t0
```

### 默认异常处理

默认的异常处理函数非常简单：

```c
extern "C" {
void __attribute__((weak)) coralnpu_exception_handler() {
    asm volatile("ebreak");
    while (1) {}
}
}
```

**解释**：
- `__attribute__((weak))`：弱符号，可以被用户定义的同名函数覆盖
- `asm volatile("ebreak")`：触发断点，方便调试
- `while (1) {}`：无限循环，防止处理器继续执行

### 自定义异常处理

用户可以定义自己的异常处理函数：

```c
extern "C" {
void coralnpu_exception_handler() {
    // 读取异常原因
    uint32_t mcause;
    asm volatile("csrr %0, mcause" : "=r"(mcause));
    
    // 读取异常地址
    uint32_t mepc;
    asm volatile("csrr %0, mepc" : "=r"(mepc));
    
    // 根据异常类型处理
    if (mcause & 0x80000000) {
        // 中断
        uint32_t interrupt_id = mcause & 0x7FFFFFFF;
        handle_interrupt(interrupt_id);
    } else {
        // 异常
        handle_exception(mcause, mepc);
    }
}
}
```

**解释**：
- `mcause`：Machine Cause 寄存器，存储异常/中断原因
- `mepc`：Machine Exception PC，存储发生异常的指令地址
- 最高位为 1 表示中断，为 0 表示异常

### 中断处理示例

```c
// 中断处理函数表
typedef void (*interrupt_handler_t)(void);
interrupt_handler_t interrupt_handlers[32];

// 注册中断处理函数
void register_interrupt_handler(uint32_t interrupt_id, 
                                interrupt_handler_t handler) {
    if (interrupt_id < 32) {
        interrupt_handlers[interrupt_id] = handler;
    }
}

// 中断分发
void handle_interrupt(uint32_t interrupt_id) {
    if (interrupt_id < 32 && interrupt_handlers[interrupt_id]) {
        interrupt_handlers[interrupt_id]();
    }
}

// 使用示例
void timer_interrupt_handler() {
    // 处理定时器中断
    // ...
}

int main() {
    // 注册定时器中断处理函数
    register_interrupt_handler(7, timer_interrupt_handler);
    
    // 启用中断
    asm volatile("csrsi mstatus, 0x8");  // 设置 MIE 位
    asm volatile("csrsi mie, 0x80");     // 启用定时器中断
    
    // 主循环
    while (1) {
        // ...
    }
}
```

**解释**：
- `interrupt_handlers` 数组存储中断处理函数指针
- `register_interrupt_handler` 用于注册处理函数
- `handle_interrupt` 根据中断 ID 调用相应的处理函数
- `csrsi`：CSR Set Immediate（设置 CSR 的指定位）
- `mstatus` 的 MIE 位（bit 3）控制全局中断使能
- `mie`：Machine Interrupt Enable，控制各个中断源的使能

### 常见异常类型

| mcause 值 | 异常类型 | 说明 |
|-----------|---------|------|
| 0 | Instruction address misaligned | 指令地址未对齐 |
| 1 | Instruction access fault | 指令访问错误 |
| 2 | Illegal instruction | 非法指令 |
| 3 | Breakpoint | 断点（ebreak） |
| 4 | Load address misaligned | 加载地址未对齐 |
| 5 | Load access fault | 加载访问错误 |
| 6 | Store address misaligned | 存储地址未对齐 |
| 7 | Store access fault | 存储访问错误 |
| 11 | Environment call from M-mode | 系统调用（ecall） |

### 中断类型

| mcause 值 | 中断类型 | 说明 |
|-----------|---------|------|
| 0x80000003 | Machine software interrupt | 软件中断 |
| 0x80000007 | Machine timer interrupt | 定时器中断 |
| 0x8000000B | Machine external interrupt | 外部中断 |

## 7.2.6 总结

本节介绍了 CoralNPU 的运行时系统：

1. **C 运行时**：负责程序启动前的初始化工作
2. **启动代码**：从 `_start` 开始，初始化硬件和软件环境
3. **内存布局**：使用 TCM 架构，分为 ITCM、DTCM 和 EXTMEM
4. **链接脚本**：定义内存布局和段的组织方式
5. **异常处理**：通过 `mtvec` 寄存器和异常处理函数处理异常和中断

理解运行时系统对于嵌入式开发非常重要，它是连接硬件和应用程序的桥梁。

## 7.2.7 参考资料

- RISC-V 特权架构规范
- GCC 链接脚本文档
- CoralNPU 源代码：`toolchain/crt/` 目录
