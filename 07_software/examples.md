# 7.5 示例程序

本节提供了一系列从简单到复杂的 CoralNPU 示例程序，帮助你快速上手 CoralNPU 编程。每个示例都包含完整的源代码、编译方法、运行步骤和详细的代码解释。

## 7.5.1 Hello World - 整数加法

这是最简单的 CoralNPU 程序，演示了如何定义输入输出缓冲区，并执行基本的整数加法运算。

### 源代码

```c
#include <stdint.h>

// 定义输入缓冲区1：8个32位无符号整数
uint32_t input1_buffer[8] __attribute__((section(".data")));

// 定义输入缓冲区2：8个32位无符号整数
uint32_t input2_buffer[8] __attribute__((section(".data")));

// 定义输出缓冲区：8个32位无符号整数
uint32_t output_buffer[8] __attribute__((section(".data")));

int main(int argc, char** argv) {
  // 逐元素相加：output[i] = input1[i] + input2[i]
  for (int i = 0; i < 8; i++) {
    output_buffer[i] = input1_buffer[i] + input2_buffer[i];
  }
  
  // 程序返回时，CoralNPU 核心会停止运行
  return 0;
}
```

### 编译命令

```bash
bazel build //examples:coralnpu_v2_hello_world_add_ints
```

编译成功后会生成 `coralnpu_v2_hello_world_add_ints.elf` 文件。

### 运行方法

CoralNPU 程序需要通过测试平台运行。你需要：

1. 编写测试脚本加载 ELF 文件
2. 向输入缓冲区写入测试数据
3. 启动 CoralNPU 执行程序
4. 等待程序完成
5. 读取输出缓冲区的结果

### 预期输出

假设输入数据为：
- input1_buffer: [0, 1, 2, 3, 4, 5, 6, 7]
- input2_buffer: [8994, 8994, 8994, 8994, 8994, 8994, 8994, 8994]

输出结果为：
```
[8994, 8995, 8996, 8997, 8998, 8999, 9000, 9001]
```

### 代码详解

#### 1. 缓冲区定义

```c
uint32_t input1_buffer[8] __attribute__((section(".data")));
```

这行代码定义了一个包含 8 个元素的数组。让我们逐部分解释：

- `uint32_t`：这是一个数据类型，表示"32位无符号整数"
  - `uint` 表示 unsigned（无符号），即只能存储非负数（0和正数）
  - `32` 表示占用32位（4字节）内存
  - `_t` 是类型后缀，表示这是一个标准定义的类型
  
- `input1_buffer[8]`：定义一个名为 input1_buffer 的数组，包含 8 个元素

- `__attribute__((section(".data")))`：这是一个编译器指令（attribute）
  - `section(".data")` 告诉编译器把这个变量放在 `.data` 段
  - `.data` 段会被加载到 CoralNPU 的 DTCM（数据紧耦合内存）中
  - 这样 CoralNPU 可以快速访问这些数据

#### 2. 主函数

```c
int main(int argc, char** argv) {
```

这是程序的入口点。CoralNPU 从这里开始执行：

- `int main`：函数返回一个整数值
- `argc` 和 `argv`：虽然定义了这两个参数，但在 CoralNPU 上通常不使用它们

#### 3. 循环计算

```c
for (int i = 0; i < 8; i++) {
  output_buffer[i] = input1_buffer[i] + input2_buffer[i];
}
```

这是一个标准的 for 循环：
- `int i = 0`：初始化循环变量 i 为 0
- `i < 8`：循环条件，当 i 小于 8 时继续循环
- `i++`：每次循环后 i 增加 1
- 循环体执行 8 次（i = 0, 1, 2, ..., 7）
- 每次将两个输入数组对应位置的元素相加，结果存入输出数组

#### 4. 程序返回

```c
return 0;
```

当 main 函数返回时，CoralNPU 核心会进入停止（halt）状态。返回值 0 通常表示程序正常结束。

### 关键概念

1. **内存布局**：CoralNPU 程序的数据存储在 DTCM 中，代码存储在 ITCM 中
2. **缓冲区通信**：主处理器通过写入 DTCM 向 CoralNPU 传递数据，通过读取 DTCM 获取结果
3. **程序生命周期**：加载 → 写入输入 → 执行 → 停止 → 读取输出

## 7.5.2 浮点数加法

这个示例演示了如何处理浮点数运算。

### 源代码

```c
#include <string.h>

// 定义浮点数输入缓冲区1：8个单精度浮点数
float input1[8] __attribute__((section(".data")));

// 定义浮点数输入缓冲区2：8个单精度浮点数
float input2[8] __attribute__((section(".data")));

// 定义浮点数输出缓冲区：8个单精度浮点数
float output[8] __attribute__((section(".data")));

int main() {
  // 逐元素相加浮点数
  for (int i = 0; i < 8; i++) {
    output[i] = input1[i] + input2[i];
  }
  return 0;
}
```

### 编译命令

```bash
bazel build //examples:coralnpu_v2_hello_world_add_floats
```

### 预期输出

假设输入数据为：
- input1: [1.0, 2.5, 3.7, 4.2, 5.9, 6.1, 7.3, 8.8]
- input2: [0.5, 0.5, 0.3, 0.8, 0.1, 0.9, 0.7, 0.2]

输出结果为：
```
[1.5, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0, 9.0]
```

### 代码详解

#### 浮点数类型

```c
float input1[8] __attribute__((section(".data")));
```

- `float`：单精度浮点数类型
  - 占用 32 位（4 字节）内存
  - 可以表示小数，如 3.14、-0.5 等
  - 精度约为 6-7 位有效数字
  - 范围约为 ±3.4 × 10^38

与整数类型 `uint32_t` 的区别：
- 整数只能表示整数值（如 1, 2, 3）
- 浮点数可以表示小数值（如 1.5, 2.7, 3.14）

#### 头文件

```c
#include <string.h>
```

虽然这个示例没有使用 `string.h` 中的函数，但在更复杂的程序中，你可能需要使用：
- `memset()`：初始化内存
- `memcpy()`：复制内存
- `memcmp()`：比较内存

### 整数 vs 浮点数

选择使用整数还是浮点数取决于你的应用：

| 特性 | 整数 (int/uint32_t) | 浮点数 (float) |
|------|---------------------|----------------|
| 表示范围 | 有限的整数 | 很大范围的实数 |
| 精度 | 精确 | 有舍入误差 |
| 运算速度 | 快 | 相对较慢 |
| 内存占用 | 4字节 | 4字节 |
| 适用场景 | 计数、索引 | 科学计算、ML |


## 7.5.3 向量操作 - RISC-V 向量扩展

CoralNPU 支持 RISC-V 向量扩展（RVV），可以一次处理多个数据，大幅提升性能。

### 源代码

```c
#include <riscv_vector.h>
#include <string.h>

// 定义输入缓冲区1：1024个8位有符号整数
int8_t input_1[1024];

// 定义输入缓冲区2：1024个8位有符号整数
int8_t input_2[1024];

// 定义输出缓冲区：1024个16位有符号整数
int16_t output[1024];

int main() {
  // 初始化输入数据
  memset(input_1, 1, 1024);  // input_1 所有元素设为 1
  memset(input_2, 6, 1024);  // input_2 所有元素设为 6
  
  // 获取数组指针
  const int8_t* input1_ptr = &input_1[0];
  const int8_t* input2_ptr = &input_2[0];
  int16_t* output_ptr = &output[0];

  // 使用向量指令处理数据，每次处理32个元素
  for (int idx = 0; (idx + 31) < 1024; idx += 32) {
    // 从内存加载32个8位整数到向量寄存器
    vint8m4_t input_v2 = __riscv_vle8_v_i8m4(input2_ptr + idx, 32);
    vint8m4_t input_v1 = __riscv_vle8_v_i8m4(input1_ptr + idx, 32);

    // 向量加法并扩展：将两个8位向量相加，结果扩展为16位
    vint16m8_t temp_sum = __riscv_vwadd_vv_i16m8(input_v1, input_v2, 32);
    
    // 将结果向量存回内存
    __riscv_vse16_v_i16m8(output_ptr + idx, temp_sum, 32);
  }

  return 0;
}
```

### 编译命令

```bash
bazel build //examples:coralnpu_v2_rvv_add_intrinsic
```

### 预期输出

- input_1 所有元素为 1
- input_2 所有元素为 6
- output 所有元素为 7（1 + 6 = 7）

前 10 个输出元素：
```
[7, 7, 7, 7, 7, 7, 7, 7, 7, 7]
```

### 代码详解

#### 1. RISC-V 向量头文件

```c
#include <riscv_vector.h>
```

这个头文件提供了 RISC-V 向量扩展的内建函数（intrinsics）。内建函数是编译器提供的特殊函数，可以直接生成向量指令。

#### 2. 数据类型

```c
int8_t input_1[1024];
int16_t output[1024];
```

- `int8_t`：8位有符号整数
  - 范围：-128 到 127
  - 占用 1 字节内存
  
- `int16_t`：16位有符号整数
  - 范围：-32768 到 32767
  - 占用 2 字节内存

为什么输出使用 `int16_t`？因为两个 `int8_t` 相加可能超出 `int8_t` 的范围。例如：127 + 127 = 254，超出了 127 的最大值。

#### 3. 内存初始化

```c
memset(input_1, 1, 1024);
```

`memset()` 函数用于初始化内存：
- 第一个参数：要初始化的内存地址（数组名）
- 第二个参数：要设置的值（1）
- 第三个参数：要设置的字节数（1024）

这行代码将 input_1 数组的所有 1024 个字节都设置为 1。

#### 4. 指针

```c
const int8_t* input1_ptr = &input_1[0];
```

这行代码创建了一个指针：
- `int8_t*`：指向 int8_t 类型的指针
- `const`：表示不能通过这个指针修改数据（只读）
- `&input_1[0]`：获取数组第一个元素的地址

指针是一个变量，存储的是内存地址。在 C 语言中：
- `&` 运算符：获取变量的地址
- `*` 运算符：访问指针指向的值

#### 5. 向量循环

```c
for (int idx = 0; (idx + 31) < 1024; idx += 32) {
```

这个循环每次处理 32 个元素：
- `idx` 从 0 开始
- 每次增加 32（`idx += 32`）
- 循环条件 `(idx + 31) < 1024` 确保不会越界
- 总共循环 1024 / 32 = 32 次

#### 6. 向量加载指令

```c
vint8m4_t input_v2 = __riscv_vle8_v_i8m4(input2_ptr + idx, 32);
```

这是一个向量加载内建函数，让我们分解理解：

- `__riscv_vle8_v_i8m4`：函数名，遵循 RISC-V 向量命名规则
  - `vle8`：向量加载（Vector Load），元素宽度 8 位
  - `v`：向量-向量操作
  - `i8m4`：8位整数，LMUL=4（寄存器组倍数）
  
- `input2_ptr + idx`：从这个地址开始加载数据
  - 指针加法：`input2_ptr + idx` 表示从数组的第 idx 个元素开始
  
- `32`：加载 32 个元素（向量长度 VL）

- `vint8m4_t`：向量类型
  - `v`：向量
  - `int8`：8位整数
  - `m4`：LMUL=4，使用 4 个向量寄存器

#### 7. 向量加法并扩展

```c
vint16m8_t temp_sum = __riscv_vwadd_vv_i16m8(input_v1, input_v2, 32);
```

这是一个向量加宽加法指令：

- `__riscv_vwadd_vv_i16m8`：函数名
  - `vwadd`：向量加宽加法（Vector Widening Add）
  - `vv`：两个向量相加
  - `i16m8`：结果是 16 位整数，LMUL=8
  
- 功能：将两个 8 位向量相加，结果扩展为 16 位
  - 输入：两个 `vint8m4_t` 向量（8位）
  - 输出：一个 `vint16m8_t` 向量（16位）
  
- 为什么 LMUL 从 4 变成 8？
  - 元素宽度翻倍（8位 → 16位）
  - 为了保持相同数量的元素，需要翻倍的寄存器

#### 8. 向量存储指令

```c
__riscv_vse16_v_i16m8(output_ptr + idx, temp_sum, 32);
```

这是一个向量存储内建函数：

- `__riscv_vse16_v_i16m8`：函数名
  - `vse16`：向量存储（Vector Store），元素宽度 16 位
  - `v`：向量操作
  - `i16m8`：16位整数，LMUL=8
  
- `output_ptr + idx`：存储到这个地址
- `temp_sum`：要存储的向量数据
- `32`：存储 32 个元素

### 向量操作的优势

标量版本（逐个处理）：
```c
for (int i = 0; i < 1024; i++) {
  output[i] = input_1[i] + input_2[i];
}
```
- 需要执行 1024 次循环
- 每次处理 1 个元素

向量版本（批量处理）：
```c
for (int idx = 0; idx < 1024; idx += 32) {
  // 向量操作，一次处理 32 个元素
}
```
- 只需执行 32 次循环
- 每次处理 32 个元素
- 理论加速比：32 倍

### RISC-V 向量扩展关键概念

1. **向量寄存器**：可以存储多个数据元素的特殊寄存器
2. **向量长度（VL）**：一次操作处理的元素数量
3. **LMUL**：寄存器组倍数，控制使用多少个向量寄存器
4. **加宽操作**：输出元素宽度是输入的两倍，避免溢出


## 7.5.4 矩阵乘法

矩阵乘法是机器学习和科学计算中的核心操作。本节展示如何在 CoralNPU 上实现矩阵乘法，并对比标量版本和向量优化版本的性能。

### 矩阵乘法原理

矩阵乘法 C = A × B 的计算规则：

```
C[i][j] = Σ(A[i][k] × B[k][j])  其中 k 从 0 到 K-1
```

例如，计算 2×3 矩阵乘以 3×2 矩阵：

```
A (2×3)        B (3×2)        C (2×2)
[1 2 3]        [7  8]         [58  64]
[4 5 6]    ×   [9  10]    =   [139 154]
               [11 12]
```

计算过程：
- C[0][0] = 1×7 + 2×9 + 3×11 = 7 + 18 + 33 = 58
- C[0][1] = 1×8 + 2×10 + 3×12 = 8 + 20 + 36 = 64
- C[1][0] = 4×7 + 5×9 + 6×11 = 28 + 45 + 66 = 139
- C[1][1] = 4×8 + 5×10 + 6×12 = 32 + 50 + 72 = 154

### 标量版本

这是最直接的实现方式，使用三层嵌套循环。

#### 源代码

```c
#include <stdint.h>

// 矩阵维度定义
#define M 16  // A 矩阵的行数
#define K 32  // A 矩阵的列数 / B 矩阵的行数
#define N 16  // B 矩阵的列数

// 定义矩阵 A (M×K)
int32_t matrix_a[M][K] __attribute__((section(".data")));

// 定义矩阵 B (K×N)
int32_t matrix_b[K][N] __attribute__((section(".data")));

// 定义结果矩阵 C (M×N)
int32_t matrix_c[M][N] __attribute__((section(".data")));

int main() {
  // 三层嵌套循环实现矩阵乘法
  for (int i = 0; i < M; i++) {           // 遍历 A 的每一行
    for (int j = 0; j < N; j++) {         // 遍历 B 的每一列
      int32_t sum = 0;                    // 累加器
      for (int k = 0; k < K; k++) {       // 遍历 A 的列 / B 的行
        sum += matrix_a[i][k] * matrix_b[k][j];  // 累加乘积
      }
      matrix_c[i][j] = sum;               // 存储结果
    }
  }
  return 0;
}
```

#### 代码详解

##### 1. 宏定义

```c
#define M 16
```

`#define` 是预处理器指令，用于定义常量：
- 在编译前，所有的 `M` 都会被替换为 `16`
- 使用宏定义的好处：
  - 代码更易读（M 比 16 更有意义）
  - 修改方便（只需改一处）
  - 编译时确定，没有运行时开销

##### 2. 二维数组

```c
int32_t matrix_a[M][K] __attribute__((section(".data")));
```

这定义了一个二维数组：
- `matrix_a[M][K]`：M 行 K 列的矩阵
- 展开后是 `matrix_a[16][32]`
- 总共包含 16 × 32 = 512 个元素
- 内存占用：512 × 4 字节 = 2048 字节

二维数组在内存中是按行存储的（行优先）：
```
matrix_a[0][0], matrix_a[0][1], ..., matrix_a[0][31],
matrix_a[1][0], matrix_a[1][1], ..., matrix_a[1][31],
...
```

##### 3. 三层循环

```c
for (int i = 0; i < M; i++) {           // 外层循环
  for (int j = 0; j < N; j++) {         // 中层循环
    int32_t sum = 0;
    for (int k = 0; k < K; k++) {       // 内层循环
      sum += matrix_a[i][k] * matrix_b[k][j];
    }
    matrix_c[i][j] = sum;
  }
}
```

循环结构分析：
- 外层循环（i）：遍历结果矩阵的每一行，执行 M 次
- 中层循环（j）：遍历结果矩阵的每一列，执行 N 次
- 内层循环（k）：计算点积，执行 K 次

总操作次数：
- 乘法次数：M × N × K = 16 × 16 × 32 = 8192 次
- 加法次数：M × N × K = 8192 次

##### 4. 累加器

```c
int32_t sum = 0;
for (int k = 0; k < K; k++) {
  sum += matrix_a[i][k] * matrix_b[k][j];
}
matrix_c[i][j] = sum;
```

累加器模式：
1. 初始化 sum 为 0
2. 循环中不断累加：`sum = sum + (a × b)`
3. 循环结束后，sum 包含最终结果
4. 将 sum 存入结果矩阵

`+=` 运算符是复合赋值运算符：
- `sum += x` 等价于 `sum = sum + x`
- 先计算右边的表达式
- 然后加到 sum 上
- 最后将结果赋值给 sum

### 向量优化版本

使用 RISC-V 向量扩展可以显著加速矩阵乘法。

#### 源代码

```c
#include <riscv_vector.h>
#include <stdint.h>

#define M 16
#define K 32
#define N 16

int32_t matrix_a[M][K] __attribute__((section(".data")));
int32_t matrix_b[K][N] __attribute__((section(".data")));
int32_t matrix_c[M][N] __attribute__((section(".data")));

int main() {
  // 遍历 A 的每一行
  for (int i = 0; i < M; i++) {
    // 遍历 B 的每一列，每次处理 4 列（向量化）
    for (int j = 0; j < N; j += 4) {
      // 初始化累加向量为 0
      vint32m1_t sum_vec = __riscv_vmv_v_x_i32m1(0, 4);
      
      // 遍历 A 的列 / B 的行
      for (int k = 0; k < K; k++) {
        // 广播 A[i][k] 到向量的所有元素
        vint32m1_t a_vec = __riscv_vmv_v_x_i32m1(matrix_a[i][k], 4);
        
        // 加载 B 的一行中的 4 个元素
        vint32m1_t b_vec = __riscv_vle32_v_i32m1(&matrix_b[k][j], 4);
        
        // 向量乘法并累加：sum_vec += a_vec * b_vec
        sum_vec = __riscv_vmacc_vv_i32m1(sum_vec, a_vec, b_vec, 4);
      }
      
      // 将结果向量存回矩阵 C
      __riscv_vse32_v_i32m1(&matrix_c[i][j], sum_vec, 4);
    }
  }
  return 0;
}
```

#### 代码详解

##### 1. 向量化策略

标量版本每次计算一个元素 C[i][j]，向量版本每次计算 4 个元素 C[i][j:j+3]。

循环结构变化：
```c
// 标量版本
for (int j = 0; j < N; j++) {  // 每次处理 1 列
  // 计算 C[i][j]
}

// 向量版本
for (int j = 0; j < N; j += 4) {  // 每次处理 4 列
  // 计算 C[i][j], C[i][j+1], C[i][j+2], C[i][j+3]
}
```

##### 2. 向量初始化

```c
vint32m1_t sum_vec = __riscv_vmv_v_x_i32m1(0, 4);
```

`__riscv_vmv_v_x_i32m1` 是向量移动指令：
- `vmv_v_x`：将标量值移动到向量的所有元素
- `i32m1`：32位整数，LMUL=1
- 第一个参数 `0`：要广播的标量值
- 第二个参数 `4`：向量长度

结果：sum_vec = [0, 0, 0, 0]

##### 3. 标量广播

```c
vint32m1_t a_vec = __riscv_vmv_v_x_i32m1(matrix_a[i][k], 4);
```

将标量 matrix_a[i][k] 广播到向量的所有元素。

例如，如果 matrix_a[i][k] = 5，则：
- a_vec = [5, 5, 5, 5]

为什么需要广播？因为我们要计算：
```
C[i][j]   = A[i][k] × B[k][j]
C[i][j+1] = A[i][k] × B[k][j+1]
C[i][j+2] = A[i][k] × B[k][j+2]
C[i][j+3] = A[i][k] × B[k][j+3]
```

注意 A[i][k] 是相同的，所以广播后可以一次计算 4 个乘法。

##### 4. 向量加载

```c
vint32m1_t b_vec = __riscv_vle32_v_i32m1(&matrix_b[k][j], 4);
```

从内存加载 4 个连续的 32 位整数：
- `&matrix_b[k][j]`：起始地址
- 加载 matrix_b[k][j], matrix_b[k][j+1], matrix_b[k][j+2], matrix_b[k][j+3]

##### 5. 向量乘加指令

```c
sum_vec = __riscv_vmacc_vv_i32m1(sum_vec, a_vec, b_vec, 4);
```

`vmacc` 是向量乘加（Multiply-Accumulate）指令：
- 功能：`sum_vec = sum_vec + (a_vec * b_vec)`
- 这是一个融合操作，一条指令完成乘法和加法
- 比分开执行乘法和加法更高效

逐元素展开：
```
sum_vec[0] = sum_vec[0] + (a_vec[0] * b_vec[0])
sum_vec[1] = sum_vec[1] + (a_vec[1] * b_vec[1])
sum_vec[2] = sum_vec[2] + (a_vec[2] * b_vec[2])
sum_vec[3] = sum_vec[3] + (a_vec[3] * b_vec[3])
```

##### 6. 向量存储

```c
__riscv_vse32_v_i32m1(&matrix_c[i][j], sum_vec, 4);
```

将向量 sum_vec 的 4 个元素存储到内存：
- 存储到 matrix_c[i][j], matrix_c[i][j+1], matrix_c[i][j+2], matrix_c[i][j+3]

### 性能对比

假设 CoralNPU 的时钟频率为 100 MHz，我们来估算两个版本的执行时间。

#### 标量版本

操作分析：
- 总循环次数：M × N × K = 16 × 16 × 32 = 8192 次（内层循环）
- 每次循环：1 次乘法 + 1 次加法 + 循环开销
- 假设每次循环需要 5 个时钟周期

估算时间：
- 总时钟周期：8192 × 5 = 40,960 周期
- 执行时间：40,960 / 100,000,000 = 0.41 毫秒

#### 向量版本

操作分析：
- 外层循环：M 次 = 16 次
- 中层循环：N / 4 次 = 16 / 4 = 4 次
- 内层循环：K 次 = 32 次
- 总循环次数：16 × 4 × 32 = 2048 次
- 每次循环处理 4 个元素

估算时间：
- 总时钟周期：2048 × 5 = 10,240 周期
- 执行时间：10,240 / 100,000,000 = 0.10 毫秒

#### 性能提升

- 加速比：40,960 / 10,240 = 4 倍
- 向量版本快 4 倍，因为每次处理 4 个元素

实际性能可能更好，因为：
1. 向量指令可能比标量指令更高效
2. 减少了循环开销
3. 更好的内存访问模式

### 优化技巧

1. **循环展开**：减少循环控制开销
2. **数据重用**：利用缓存，减少内存访问
3. **向量化**：使用 SIMD 指令并行处理
4. **分块（Tiling）**：将大矩阵分成小块，提高缓存命中率


## 7.5.5 机器学习推理

CoralNPU 可以运行机器学习模型进行推理。本节展示如何使用 TensorFlow Lite Micro 在 CoralNPU 上运行神经网络模型。

### 什么是机器学习推理

机器学习推理是指使用已训练好的模型对新数据进行预测的过程：

1. **训练阶段**（通常在服务器上完成）：
   - 使用大量数据训练模型
   - 调整模型参数（权重和偏置）
   - 得到训练好的模型文件

2. **推理阶段**（在 CoralNPU 上执行）：
   - 加载训练好的模型
   - 输入新数据
   - 模型计算并输出预测结果

### TensorFlow Lite Micro 简介

TensorFlow Lite Micro (TFLM) 是 TensorFlow 的轻量级版本，专为嵌入式设备设计：

- **小内存占用**：适合资源受限的设备
- **无操作系统依赖**：可以在裸机上运行
- **支持量化模型**：使用 8 位整数代替浮点数，加速计算
- **优化的算子**：针对嵌入式处理器优化

### 基础推理示例

#### 源代码

```c
#include "tensorflow/lite/micro/micro_interpreter.h"
#include "tensorflow/lite/micro/micro_mutable_op_resolver.h"
#include "tensorflow/lite/schema/schema_generated.h"

// 模型数据（通常从 .tflite 文件转换而来）
extern const unsigned char model_data[];
extern const int model_data_len;

// 定义 tensor arena（张量工作区）
constexpr int kTensorArenaSize = 10 * 1024;  // 10 KB
uint8_t tensor_arena[kTensorArenaSize] __attribute__((section(".data")));

int main() {
  // 1. 加载模型
  const tflite::Model* model = tflite::GetModel(model_data);
  
  // 检查模型版本
  if (model->version() != TFLITE_SCHEMA_VERSION) {
    // 版本不匹配，返回错误
    return -1;
  }

  // 2. 设置算子解析器（注册模型使用的操作）
  tflite::MicroMutableOpResolver<5> resolver;
  resolver.AddFullyConnected();  // 全连接层
  resolver.AddSoftmax();         // Softmax 激活函数
  resolver.AddQuantize();        // 量化操作
  resolver.AddDequantize();      // 反量化操作
  resolver.AddReshape();         // 重塑操作

  // 3. 创建解释器
  tflite::MicroInterpreter interpreter(
      model, resolver, tensor_arena, kTensorArenaSize);

  // 4. 分配张量内存
  if (interpreter.AllocateTensors() != kTfLiteOk) {
    // 内存分配失败
    return -2;
  }

  // 5. 获取输入张量
  TfLiteTensor* input = interpreter.input(0);
  
  // 6. 填充输入数据
  // 假设输入是 28×28 的图像，展平为 784 个元素
  for (int i = 0; i < 784; i++) {
    input->data.int8[i] = /* 你的输入数据 */;
  }

  // 7. 运行推理
  if (interpreter.Invoke() != kTfLiteOk) {
    // 推理失败
    return -3;
  }

  // 8. 获取输出张量
  TfLiteTensor* output = interpreter.output(0);
  
  // 9. 读取结果
  // 假设输出是 10 个类别的概率（如 MNIST 数字识别）
  int8_t max_score = output->data.int8[0];
  int max_index = 0;
  for (int i = 1; i < 10; i++) {
    if (output->data.int8[i] > max_score) {
      max_score = output->data.int8[i];
      max_index = i;
    }
  }
  
  // max_index 就是预测的类别
  return max_index;
}
```

#### 代码详解

##### 1. 模型数据

```c
extern const unsigned char model_data[];
extern const int model_data_len;
```

`extern` 关键字表示这些变量在其他文件中定义：
- `model_data[]`：模型的二进制数据（.tflite 文件内容）
- `model_data_len`：模型数据的长度（字节数）

通常使用工具将 .tflite 文件转换为 C 数组：
```bash
xxd -i model.tflite > model_data.cc
```

##### 2. Tensor Arena（张量工作区）

```c
constexpr int kTensorArenaSize = 10 * 1024;
uint8_t tensor_arena[kTensorArenaSize] __attribute__((section(".data")));
```

Tensor Arena 是一块内存区域，用于存储：
- 输入张量
- 输出张量
- 中间层的激活值
- 临时缓冲区

`constexpr` 是 C++ 关键字，表示编译时常量：
- 值在编译时确定
- 可以用于数组大小等需要常量的地方
- 比 `#define` 更类型安全

大小选择：
- 太小：内存分配失败
- 太大：浪费内存
- 需要根据模型大小调整

##### 3. 加载模型

```c
const tflite::Model* model = tflite::GetModel(model_data);
```

`tflite::GetModel()` 解析模型数据：
- 输入：模型的二进制数据
- 输出：指向 Model 对象的指针
- Model 对象包含模型的结构、权重等信息

`::` 是作用域解析运算符：
- `tflite::GetModel` 表示 tflite 命名空间中的 GetModel 函数
- 命名空间用于避免名称冲突

##### 4. 版本检查

```c
if (model->version() != TFLITE_SCHEMA_VERSION) {
  return -1;
}
```

检查模型版本是否与 TFLM 库版本兼容：
- `model->version()`：模型的 schema 版本
- `TFLITE_SCHEMA_VERSION`：库支持的 schema 版本
- 不匹配可能导致解析错误或运行时错误

`->` 是指针成员访问运算符：
- `model->version()` 等价于 `(*model).version()`
- 用于通过指针访问对象的成员

##### 5. 算子解析器

```c
tflite::MicroMutableOpResolver<5> resolver;
resolver.AddFullyConnected();
resolver.AddSoftmax();
```

算子解析器（Op Resolver）负责注册模型使用的操作：
- `MicroMutableOpResolver<5>`：可以注册最多 5 个操作
- 每个 `Add...()` 调用注册一个操作

为什么需要注册？
- TFLM 不会自动包含所有操作（减小代码体积）
- 只注册模型实际使用的操作
- 如果模型使用了未注册的操作，推理会失败

常见操作：
- `AddFullyConnected()`：全连接层（密集层）
- `AddConv2D()`：2D 卷积层
- `AddDepthwiseConv2D()`：深度可分离卷积
- `AddMaxPool2D()`：最大池化层
- `AddSoftmax()`：Softmax 激活函数
- `AddReshape()`：重塑张量形状

##### 6. 创建解释器

```c
tflite::MicroInterpreter interpreter(
    model, resolver, tensor_arena, kTensorArenaSize);
```

解释器（Interpreter）是推理的核心：
- 管理模型的执行
- 分配和管理内存
- 调用算子执行计算

构造函数参数：
1. `model`：要执行的模型
2. `resolver`：算子解析器
3. `tensor_arena`：工作内存
4. `kTensorArenaSize`：工作内存大小

##### 7. 分配张量

```c
if (interpreter.AllocateTensors() != kTfLiteOk) {
  return -2;
}
```

`AllocateTensors()` 在 tensor arena 中分配内存：
- 为所有张量分配空间
- 返回 `kTfLiteOk` 表示成功
- 失败通常是因为 arena 太小

##### 8. 获取输入张量

```c
TfLiteTensor* input = interpreter.input(0);
```

获取模型的输入张量：
- `input(0)`：获取第一个输入（索引从 0 开始）
- 如果模型有多个输入，可以用 `input(1)`, `input(2)` 等
- 返回指向 `TfLiteTensor` 结构的指针

`TfLiteTensor` 结构包含：
- `data`：张量数据的指针
- `dims`：张量的维度
- `type`：数据类型（如 int8, float32）

##### 9. 填充输入数据

```c
for (int i = 0; i < 784; i++) {
  input->data.int8[i] = /* 你的输入数据 */;
}
```

将输入数据写入张量：
- `input->data.int8`：访问 int8 类型的数据数组
- 对于浮点模型，使用 `input->data.f`
- 对于 uint8 模型，使用 `input->data.uint8`

数据格式：
- 图像通常需要预处理（归一化、量化）
- 多维张量按行优先顺序展平

##### 10. 运行推理

```c
if (interpreter.Invoke() != kTfLiteOk) {
  return -3;
}
```

`Invoke()` 执行模型推理：
- 按顺序执行模型的所有层
- 从输入张量读取数据
- 将结果写入输出张量
- 返回 `kTfLiteOk` 表示成功

##### 11. 获取输出张量

```c
TfLiteTensor* output = interpreter.output(0);
```

获取模型的输出张量：
- `output(0)`：获取第一个输出
- 多输出模型可以用 `output(1)`, `output(2)` 等

##### 12. 解析结果

```c
int8_t max_score = output->data.int8[0];
int max_index = 0;
for (int i = 1; i < 10; i++) {
  if (output->data.int8[i] > max_score) {
    max_score = output->data.int8[i];
    max_index = i;
  }
}
```

这段代码找出最大值的索引：
- 遍历输出数组的所有元素
- 记录最大值和对应的索引
- 对于分类任务，索引就是预测的类别

例如，MNIST 数字识别：
- 输出有 10 个元素（对应数字 0-9）
- 每个元素表示该数字的概率（分数）
- 最大值对应的索引就是预测的数字

### 完整的 MobileNet 推理示例

MobileNet 是一个轻量级的图像分类模型，适合在嵌入式设备上运行。

#### 源代码

```c
#include "tensorflow/lite/micro/micro_interpreter.h"
#include "tensorflow/lite/micro/micro_mutable_op_resolver.h"
#include "tensorflow/lite/schema/schema_generated.h"

// 包含模型数据
#include "mobilenet_v1_model_data.h"

// Tensor arena 大小（根据模型调整）
constexpr int kTensorArenaSize = 100 * 1024;  // 100 KB
uint8_t tensor_arena[kTensorArenaSize] __attribute__((section(".data")));

// 输入图像数据（224×224×3 = 150528 像素）
uint8_t input_image[150528] __attribute__((section(".data")));

// 输出类别（1000 个 ImageNet 类别）
int8_t output_scores[1000] __attribute__((section(".data")));

int main() {
  // 1. 加载模型
  const tflite::Model* model = tflite::GetModel(mobilenet_v1_model_data);
  
  if (model->version() != TFLITE_SCHEMA_VERSION) {
    return -1;
  }

  // 2. 注册 MobileNet 使用的算子
  tflite::MicroMutableOpResolver<10> resolver;
  resolver.AddConv2D();              // 标准卷积
  resolver.AddDepthwiseConv2D();     // 深度可分离卷积
  resolver.AddFullyConnected();      // 全连接层
  resolver.AddSoftmax();             // Softmax
  resolver.AddReshape();             // 重塑
  resolver.AddQuantize();            // 量化
  resolver.AddDequantize();          // 反量化
  resolver.AddAveragePool2D();       // 平均池化
  resolver.AddRelu();                // ReLU 激活
  resolver.AddMean();                // 均值操作

  // 3. 创建解释器
  tflite::MicroInterpreter interpreter(
      model, resolver, tensor_arena, kTensorArenaSize);

  // 4. 分配张量
  if (interpreter.AllocateTensors() != kTfLiteOk) {
    return -2;
  }

  // 5. 获取输入张量并填充数据
  TfLiteTensor* input = interpreter.input(0);
  
  // 将图像数据复制到输入张量
  // 注意：实际应用中需要对图像进行预处理
  for (int i = 0; i < 150528; i++) {
    input->data.uint8[i] = input_image[i];
  }

  // 6. 运行推理
  if (interpreter.Invoke() != kTfLiteOk) {
    return -3;
  }

  // 7. 获取输出并找到最可能的类别
  TfLiteTensor* output = interpreter.output(0);
  
  // 复制输出分数
  for (int i = 0; i < 1000; i++) {
    output_scores[i] = output->data.int8[i];
  }
  
  // 找到最高分数的类别
  int8_t max_score = output_scores[0];
  int predicted_class = 0;
  
  for (int i = 1; i < 1000; i++) {
    if (output_scores[i] > max_score) {
      max_score = output_scores[i];
      predicted_class = i;
    }
  }

  // predicted_class 是预测的 ImageNet 类别 ID
  return predicted_class;
}
```

#### 编译命令

```bash
bazel build //tests/cocotb/tutorial/tfmicro:run_mobilenet
```

#### 预期输出

假设输入一张猫的图片：
- predicted_class 可能是 281（对应 ImageNet 中的 "tabby cat"）
- max_score 是该类别的置信度分数

#### 关键概念

##### 1. 量化模型

量化是将浮点数转换为整数的过程：

**浮点模型**：
- 权重和激活值使用 float32（32位浮点数）
- 精度高，但计算慢，内存占用大
- 模型大小：~17 MB（MobileNet V1）

**量化模型**：
- 权重和激活值使用 int8（8位整数）
- 精度略降，但计算快，内存占用小
- 模型大小：~4 MB（MobileNet V1）

量化公式：
```
量化值 = round((浮点值 - zero_point) / scale)
反量化值 = 量化值 * scale + zero_point
```

##### 2. 深度可分离卷积

MobileNet 使用深度可分离卷积减少计算量：

**标准卷积**：
- 同时处理空间和通道维度
- 计算量：H × W × C_in × C_out × K × K

**深度可分离卷积**：
- 分为两步：深度卷积 + 逐点卷积
- 计算量：H × W × C_in × K × K + H × W × C_in × C_out
- 计算量减少约 8-9 倍

##### 3. 图像预处理

输入图像通常需要预处理：

1. **调整大小**：缩放到模型要求的尺寸（如 224×224）
2. **归一化**：将像素值从 [0, 255] 映射到 [-1, 1] 或 [0, 1]
3. **量化**：转换为 int8 或 uint8
4. **通道顺序**：确保 RGB 或 BGR 顺序正确

示例预处理代码：
```c
// 假设原始像素值在 [0, 255] 范围
for (int i = 0; i < 150528; i++) {
  // 归一化到 [-1, 1]
  float normalized = (raw_pixel[i] / 127.5) - 1.0;
  
  // 量化到 int8
  input->data.int8[i] = (int8_t)(normalized * 128);
}
```


### 性能优化技巧

#### 1. 选择合适的模型

不同模型的计算量和精度差异很大：

| 模型 | 参数量 | 计算量 | Top-1 精度 | 适用场景 |
|------|--------|--------|-----------|----------|
| MobileNet V1 | 4.2M | 569M | 70.6% | 通用分类 |
| MobileNet V2 | 3.4M | 300M | 71.8% | 通用分类 |
| EfficientNet-Lite | 4.7M | 407M | 75.1% | 高精度需求 |
| SqueezeNet | 1.2M | 352M | 57.5% | 极小模型 |

选择建议：
- 精度要求高：EfficientNet-Lite
- 速度要求高：MobileNet V2
- 内存受限：SqueezeNet

#### 2. 使用量化模型

量化可以显著提升性能：

- **训练后量化**：训练完成后转换为 int8
  - 简单快速
  - 精度损失 1-2%
  
- **量化感知训练**：训练时模拟量化
  - 精度损失更小（<1%）
  - 需要重新训练

转换命令（使用 TensorFlow）：
```python
import tensorflow as tf

# 加载浮点模型
model = tf.keras.models.load_model('model.h5')

# 转换为 TFLite 量化模型
converter = tf.lite.TFLiteConverter.from_keras_model(model)
converter.optimizations = [tf.lite.Optimize.DEFAULT]
tflite_model = converter.convert()

# 保存模型
with open('model_quantized.tflite', 'wb') as f:
  f.write(tflite_model)
```

#### 3. 优化 Tensor Arena 大小

Tensor Arena 太大浪费内存，太小导致分配失败。

确定最小 Arena 大小：
```c
// 分配张量后检查实际使用量
interpreter.AllocateTensors();
size_t used_bytes = interpreter.arena_used_bytes();
printf("Arena used: %zu bytes\n", used_bytes);
```

建议：
- 初始设置为估计值的 1.5 倍
- 运行测试，记录实际使用量
- 调整为实际使用量 + 10% 余量

#### 4. 使用向量指令加速算子

为关键算子编写向量优化版本：

```c
// 标量版本的 ReLU
void relu_scalar(float* data, int size) {
  for (int i = 0; i < size; i++) {
    if (data[i] < 0) {
      data[i] = 0;
    }
  }
}

// 向量版本的 ReLU（使用 RVV）
void relu_vector(float* data, int size) {
  for (int i = 0; i < size; i += 8) {
    vfloat32m2_t v = __riscv_vle32_v_f32m2(data + i, 8);
    vfloat32m2_t zero = __riscv_vfmv_v_f_f32m2(0.0f, 8);
    vfloat32m2_t result = __riscv_vfmax_vv_f32m2(v, zero, 8);
    __riscv_vse32_v_f32m2(data + i, result, 8);
  }
}
```

### 调试技巧

#### 1. 检查张量形状

```c
TfLiteTensor* input = interpreter.input(0);
printf("Input shape: ");
for (int i = 0; i < input->dims->size; i++) {
  printf("%d ", input->dims->data[i]);
}
printf("\n");
```

#### 2. 打印中间层输出

```c
// 获取中间层张量（需要知道张量索引）
TfLiteTensor* intermediate = interpreter.tensor(5);
printf("Intermediate output: ");
for (int i = 0; i < 10; i++) {
  printf("%d ", intermediate->data.int8[i]);
}
printf("\n");
```

#### 3. 验证输入数据

```c
// 检查输入数据范围
int8_t min_val = input->data.int8[0];
int8_t max_val = input->data.int8[0];
for (int i = 1; i < input_size; i++) {
  if (input->data.int8[i] < min_val) min_val = input->data.int8[i];
  if (input->data.int8[i] > max_val) max_val = input->data.int8[i];
}
printf("Input range: [%d, %d]\n", min_val, max_val);
```

## 7.5.6 编译和运行流程

### 使用 Bazel 编译

CoralNPU 项目使用 Bazel 构建系统。

#### 基本编译命令

```bash
# 编译单个目标
bazel build //examples:coralnpu_v2_hello_world_add_floats

# 编译所有示例
bazel build //examples:all

# 编译并显示详细信息
bazel build //examples:coralnpu_v2_hello_world_add_floats --verbose_failures

# 清理构建缓存
bazel clean
```

#### BUILD 文件示例

```python
# examples/BUILD.bazel

load("//rules:coralnpu_v2.bzl", "coralnpu_v2_binary")

# 定义一个 CoralNPU 程序
coralnpu_v2_binary(
    name = "coralnpu_v2_hello_world",
    srcs = ["hello_world.cc"],
    deps = [
        # 依赖的库
    ],
)

# 定义一个使用 TensorFlow Lite Micro 的程序
coralnpu_v2_binary(
    name = "coralnpu_v2_mobilenet",
    srcs = [
        "mobilenet.cc",
        "model_data.cc",
    ],
    deps = [
        "@tflite_micro//:micro_framework",
    ],
)
```

#### 编译选项

```bash
# 优化级别
bazel build //examples:target -c opt          # 优化编译（推荐）
bazel build //examples:target -c dbg          # 调试编译

# 指定工具链
bazel build //examples:target --config=coralnpu_v2

# 查看生成的汇编代码
bazel build //examples:target --save_temps
```

### 运行测试

#### 使用 Cocotb 测试框架

```bash
# 运行单个测试
bazel run //tests/cocotb/tutorial:tutorial

# 运行所有测试
bazel test //tests/cocotb/...

# 运行测试并显示输出
bazel test //tests/cocotb/tutorial:tutorial --test_output=all
```

#### 测试脚本示例

```python
import cocotb
from cocotb_test_utils import CoreMiniAxiInterface
import numpy as np

@cocotb.test()
async def test_hello_world(dut):
    """测试 Hello World 程序"""
    # 初始化测试环境
    core = CoreMiniAxiInterface(dut)
    await core.init()
    await core.reset()
    cocotb.start_soon(core.clock.start())
    
    # 加载 ELF 文件
    with open("coralnpu_v2_hello_world.elf", "rb") as f:
        entry_point = await core.load_elf(f)
        
        # 查找缓冲区地址
        input1_addr = core.lookup_symbol(f, "input1_buffer")
        input2_addr = core.lookup_symbol(f, "input2_buffer")
        output_addr = core.lookup_symbol(f, "output_buffer")
    
    # 准备输入数据
    input1 = np.arange(8, dtype=np.uint32)
    input2 = np.full(8, 100, dtype=np.uint32)
    
    # 写入输入数据
    await core.write(input1_addr, input1)
    await core.write(input2_addr, input2)
    
    # 执行程序
    await core.execute_from(entry_point)
    await core.wait_for_halted()
    
    # 读取输出
    output = (await core.read(output_addr, 4 * 8)).view(np.uint32)
    
    # 验证结果
    expected = input1 + input2
    assert np.array_equal(output, expected), f"Expected {expected}, got {output}"
    
    print(f"Test passed! Output: {output}")
```

### 在 FPGA 上运行

#### 1. 生成比特流

```bash
# 构建 FPGA 比特流
bazel build //fpga:coralnpu_fpga_bitstream
```

#### 2. 烧录 FPGA

```bash
# 使用 OpenOCD 烧录
openocd -f board/arty_a7.cfg -c "program bitstream.bit verify reset exit"
```

#### 3. 加载程序

```bash
# 通过 UART 或 JTAG 加载程序
./load_program.sh coralnpu_v2_hello_world.elf
```

#### 4. 查看输出

```bash
# 连接串口查看输出
screen /dev/ttyUSB0 115200
```

## 7.5.7 常见问题

### 编译错误

#### 问题：找不到头文件

```
fatal error: riscv_vector.h: No such file or directory
```

解决方法：
- 确保使用正确的工具链
- 检查 BUILD 文件中的依赖项
- 添加 `--config=coralnpu_v2` 编译选项

#### 问题：链接错误

```
undefined reference to `__riscv_vle32_v_i32m1'
```

解决方法：
- 确保启用了向量扩展：`-march=rv32imv`
- 检查链接器脚本是否正确

### 运行时错误

#### 问题：程序不停止

可能原因：
- 死循环
- 等待外部事件
- 异常未处理

调试方法：
- 添加打印语句
- 使用调试器单步执行
- 检查循环条件

#### 问题：输出结果错误

可能原因：
- 输入数据错误
- 缓冲区地址错误
- 数据类型不匹配
- 字节序问题

调试方法：
- 打印输入数据
- 验证缓冲区地址
- 检查数据类型
- 使用已知输入测试

#### 问题：内存不足

```
AllocateTensors() failed: arena too small
```

解决方法：
- 增加 `kTensorArenaSize`
- 使用更小的模型
- 减少批次大小

### 性能问题

#### 问题：运行速度慢

优化方法：
1. 使用向量指令
2. 启用编译器优化（`-O3`）
3. 减少内存访问
4. 使用量化模型
5. 优化循环结构

#### 问题：功耗过高

优化方法：
1. 降低时钟频率
2. 使用低功耗模式
3. 减少不必要的计算
4. 优化内存访问模式

## 7.5.8 进阶主题

### 自定义算子

如果 TFLM 不支持某个操作，可以自定义算子：

```c
// 自定义算子的实现
TfLiteStatus CustomOpEval(TfLiteContext* context, TfLiteNode* node) {
  // 获取输入张量
  const TfLiteTensor* input = GetInput(context, node, 0);
  
  // 获取输出张量
  TfLiteTensor* output = GetOutput(context, node, 0);
  
  // 执行自定义操作
  for (int i = 0; i < input->bytes; i++) {
    output->data.uint8[i] = custom_function(input->data.uint8[i]);
  }
  
  return kTfLiteOk;
}

// 注册自定义算子
TfLiteRegistration* Register_CUSTOM_OP() {
  static TfLiteRegistration r = {
    .init = nullptr,
    .free = nullptr,
    .prepare = CustomOpPrepare,
    .invoke = CustomOpEval,
  };
  return &r;
}

// 在解析器中注册
resolver.AddCustom("CustomOp", Register_CUSTOM_OP());
```

### 多核并行

如果 CoralNPU 有多个核心，可以并行执行：

```c
// 核心 0：处理前半部分数据
void core0_task() {
  for (int i = 0; i < N/2; i++) {
    output[i] = process(input[i]);
  }
}

// 核心 1：处理后半部分数据
void core1_task() {
  for (int i = N/2; i < N; i++) {
    output[i] = process(input[i]);
  }
}

int main() {
  int core_id = get_core_id();
  
  if (core_id == 0) {
    core0_task();
  } else if (core_id == 1) {
    core1_task();
  }
  
  // 同步等待所有核心完成
  barrier_wait();
  
  return 0;
}
```

### DMA 传输

使用 DMA 可以在后台传输数据，提高效率：

```c
// 启动 DMA 传输
dma_transfer(src_addr, dst_addr, size);

// CPU 可以继续执行其他任务
do_other_work();

// 等待 DMA 完成
dma_wait();

// 现在可以使用传输的数据
process_data(dst_addr);
```

### 流水线处理

将计算分成多个阶段，形成流水线：

```c
// 三级流水线：加载 -> 计算 -> 存储

// 阶段 1：加载数据
load_data(buffer_a, 0);

for (int i = 1; i < N; i++) {
  // 阶段 1：加载下一批数据
  load_data(buffer_b, i);
  
  // 阶段 2：计算当前数据
  compute(buffer_a, result_a);
  
  // 阶段 3：存储上一批结果
  if (i > 1) {
    store_data(result_b, i-1);
  }
  
  // 交换缓冲区
  swap(buffer_a, buffer_b);
  swap(result_a, result_b);
}

// 处理最后一批
compute(buffer_a, result_a);
store_data(result_a, N-1);
```

## 7.5.9 总结

本章介绍了 CoralNPU 的示例程序，从简单的整数加法到复杂的机器学习推理。关键要点：

1. **基础程序结构**：
   - 定义输入输出缓冲区
   - 实现计算逻辑
   - 程序返回时核心停止

2. **向量化优化**：
   - 使用 RISC-V 向量扩展
   - 一次处理多个数据
   - 显著提升性能

3. **机器学习推理**：
   - 使用 TensorFlow Lite Micro
   - 加载量化模型
   - 运行推理并获取结果

4. **性能优化**：
   - 选择合适的模型
   - 使用量化
   - 优化内存使用
   - 编写向量化代码

5. **调试技巧**：
   - 检查张量形状
   - 验证输入输出
   - 使用测试框架

通过这些示例，你应该能够开始编写自己的 CoralNPU 程序。建议从简单的例子开始，逐步尝试更复杂的应用。

### 下一步

- 阅读 RISC-V 向量扩展规范，深入理解向量指令
- 学习 TensorFlow Lite Micro 文档，了解更多算子
- 尝试移植自己的机器学习模型到 CoralNPU
- 优化关键代码路径，提升性能
- 参与 CoralNPU 社区，分享经验和代码

### 参考资源

- [RISC-V 向量扩展规范](https://github.com/riscv/riscv-v-spec)
- [TensorFlow Lite Micro 文档](https://www.tensorflow.org/lite/microcontrollers)
- [CoralNPU 官方文档](https://github.com/google/coralnpu)
- [Bazel 构建系统](https://bazel.build/)
- [Cocotb 测试框架](https://docs.cocotb.org/)
