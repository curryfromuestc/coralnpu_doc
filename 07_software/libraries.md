# 7.3 库

本章介绍 CoralNPU 项目中可用的各种软件库，包括标准 C 库、数学库、向量优化库、机器学习库以及如何编写自定义库。

## 7.3.1 标准库

### C 标准库支持

CoralNPU 支持标准的 C 库函数，这些函数是编写嵌入式应用程序的基础。项目使用 newlib 或 picolibc 作为 C 标准库的实现。

**什么是 newlib/picolibc？**
- newlib：一个专为嵌入式系统设计的 C 标准库实现
- picolibc：newlib 的轻量级替代品，占用更少的内存和代码空间
- 两者都提供了标准 C 库函数（如 printf、malloc、memcpy 等）

### 常用标准库函数

#### 输入输出函数

```c
#include <stdio.h>

// 格式化输出到串口
printf("Hello, CoralNPU!\n");
printf("数值: %d, 十六进制: 0x%x\n", 42, 42);

// 格式化字符串
char buffer[100];
sprintf(buffer, "结果: %d", result);
```

**解释：**
- `printf`：将格式化的文本输出到标准输出（通常是串口）
- `%d`：整数占位符
- `%x`：十六进制占位符
- `\n`：换行符

#### 内存管理函数

```c
#include <stdlib.h>

// 动态分配内存
int *array = (int*)malloc(100 * sizeof(int));
if (array == NULL) {
    printf("内存分配失败!\n");
    return -1;
}

// 使用内存
for (int i = 0; i < 100; i++) {
    array[i] = i * 2;
}

// 释放内存
free(array);
```

**解释：**
- `malloc(size)`：分配 size 字节的内存，返回指向该内存的指针
- 如果分配失败，返回 NULL
- `free(ptr)`：释放之前分配的内存
- **重要**：每次 malloc 后必须对应一个 free，否则会造成内存泄漏

#### 字符串和内存操作函数

```c
#include <string.h>

// 复制内存
char src[20] = "Hello";
char dst[20];
memcpy(dst, src, strlen(src) + 1);  // +1 是为了复制结尾的 '\0'

// 设置内存
int buffer[100];
memset(buffer, 0, sizeof(buffer));  // 将 buffer 全部设为 0

// 字符串操作
char str1[50] = "Hello, ";
char str2[] = "World!";
strcat(str1, str2);  // str1 现在是 "Hello, World!"
int len = strlen(str1);  // len = 13
```

**解释：**
- `memcpy(dest, src, n)`：从 src 复制 n 个字节到 dest
- `memset(ptr, value, n)`：将 ptr 指向的 n 个字节都设置为 value
- `strlen(str)`：返回字符串的长度（不包括结尾的 '\0'）
- `strcat(dest, src)`：将 src 字符串追加到 dest 后面

#### 其他常用函数

```c
#include <stdlib.h>

// 字符串转数字
int num = atoi("123");        // 字符串转整数
float f = atof("3.14");       // 字符串转浮点数

// 随机数（需要先设置种子）
srand(time(NULL));            // 设置随机数种子
int random_num = rand();      // 生成随机数
```

## 7.3.2 数学库

### libm 数学函数

CoralNPU 支持标准的数学库（libm），提供各种数学运算函数。

```c
#include <math.h>

// 基本运算
float x = 2.5;
float y = sqrt(x);           // 平方根: 1.58...
float z = pow(x, 3);         // x 的 3 次方: 15.625
float abs_val = fabs(-3.14); // 绝对值: 3.14

// 三角函数（参数是弧度）
float angle = 3.14159 / 4;   // 45 度
float s = sin(angle);        // 正弦值
float c = cos(angle);        // 余弦值
float t = tan(angle);        // 正切值

// 反三角函数
float asin_val = asin(0.5);  // 反正弦
float acos_val = acos(0.5);  // 反余弦
float atan_val = atan(1.0);  // 反正切

// 指数和对数
float exp_val = exp(1.0);    // e^1 = 2.718...
float log_val = log(2.718);  // 自然对数 ln(2.718) ≈ 1
float log10_val = log10(100);// 以 10 为底的对数 = 2

// 取整函数
float ceil_val = ceil(3.2);  // 向上取整: 4.0
float floor_val = floor(3.8);// 向下取整: 3.0
float round_val = round(3.5);// 四舍五入: 4.0
```

**解释：**
- 三角函数的参数单位是**弧度**，不是角度
- 角度转弧度：`弧度 = 角度 * π / 180`
- 弧度转角度：`角度 = 弧度 * 180 / π`
- `π ≈ 3.14159`

### 浮点运算注意事项

```c
// 浮点数比较不要直接用 ==
float a = 0.1 + 0.2;
float b = 0.3;

// 错误的做法
if (a == b) {  // 可能不相等！
    // ...
}

// 正确的做法：使用误差范围
float epsilon = 0.00001;
if (fabs(a - b) < epsilon) {  // 差值小于误差范围就认为相等
    // ...
}
```

**解释：**
- 浮点数在计算机中是近似表示的，不是精确值
- 直接用 `==` 比较浮点数可能会出错
- 应该判断两个浮点数的差值是否小于一个很小的数（epsilon）

## 7.3.3 向量库

CoralNPU 支持 RISC-V 向量扩展（RVV），可以使用向量指令加速数据处理。

### 什么是向量操作？

向量操作可以一次处理多个数据，比逐个处理快得多。

**普通方式（标量）：**
```c
// 逐个元素相加，需要循环 100 次
for (int i = 0; i < 100; i++) {
    c[i] = a[i] + b[i];
}
```

**向量方式：**
```c
// 一次处理多个元素，循环次数大大减少
// 具体一次能处理多少个，由硬件的向量长度决定
```

### RISC-V 向量内置函数

CoralNPU 提供了 RISC-V 向量内置函数（intrinsics），这些函数以 `__riscv_` 开头。

#### 基本向量操作示例

```c
#include <riscv_vector.h>

// 向量化的内存复制函数
void* vector_memcpy(void* dst, const void* src, size_t n) {
    const uint8_t* s = (const uint8_t*)src;
    uint8_t* d = (uint8_t*)dst;
    size_t vl = 0;  // 向量长度
    
    while (n > 0) {
        // 设置向量长度：告诉硬件这次要处理多少个元素
        vl = __riscv_vsetvl_e8m8(n);
        
        // 从源地址加载 vl 个字节到向量寄存器
        vuint8m8_t vload_data = __riscv_vle8_v_u8m8(s, vl);
        
        // 将向量寄存器的数据存储到目标地址
        __riscv_vse8_v_u8m8(d, vload_data, vl);
        
        // 更新指针和剩余字节数
        s += vl;
        d += vl;
        n -= vl;
    }
    
    return dst;
}
```

**解释：**
- `__riscv_vsetvl_e8m8(n)`：设置向量长度
  - `e8`：每个元素 8 位（1 字节）
  - `m8`：使用 8 个向量寄存器组
  - 返回值 vl：实际能处理的元素个数
- `__riscv_vle8_v_u8m8(addr, vl)`：从内存加载 vl 个 8 位无符号整数
- `__riscv_vse8_v_u8m8(addr, data, vl)`：将向量数据存储到内存

#### 向量算术运算

```c
#include <riscv_vector.h>

// 向量加法：c[i] = a[i] + b[i]
void vector_add(int32_t* c, const int32_t* a, const int32_t* b, size_t n) {
    size_t vl;
    
    while (n > 0) {
        vl = __riscv_vsetvl_e32m8(n);  // 32 位整数
        
        // 加载两个向量
        vint32m8_t va = __riscv_vle32_v_i32m8(a, vl);
        vint32m8_t vb = __riscv_vle32_v_i32m8(b, vl);
        
        // 向量加法
        vint32m8_t vc = __riscv_vadd_vv_i32m8(va, vb, vl);
        
        // 存储结果
        __riscv_vse32_v_i32m8(c, vc, vl);
        
        a += vl;
        b += vl;
        c += vl;
        n -= vl;
    }
}
```

**解释：**
- `__riscv_vadd_vv_i32m8(va, vb, vl)`：向量加法
  - `vadd`：向量加法指令
  - `vv`：两个操作数都是向量（vector-vector）
  - `i32`：32 位有符号整数
  - `m8`：使用 8 个向量寄存器组

#### 向量类型转换

```c
// 8 位整数扩展到 16 位
vint8m2_t v8 = __riscv_vle8_v_i8m2(data, vl);
vint16m4_t v16 = __riscv_vsext_vf2_i16m4(v8, vl);  // 符号扩展

// 16 位整数缩窄到 8 位
vint16m4_t v16 = __riscv_vle16_v_i16m4(data, vl);
vint8m2_t v8 = __riscv_vncvt_x_x_w_i8m2(v16, vl);  // 缩窄转换
```

**解释：**
- `vsext_vf2`：符号扩展，宽度变为 2 倍（vf2 = vector factor 2）
- `vncvt`：缩窄转换（narrow convert）
- 扩展时寄存器组数量会翻倍（m2 → m4）
- 缩窄时寄存器组数量会减半（m4 → m2）

### 向量函数命名规则

RISC-V 向量内置函数的命名遵循一定规则：

```
__riscv_<操作>_<操作数类型>_<数据类型><LMUL>
```

- **操作**：`vle`（加载）、`vse`（存储）、`vadd`（加法）、`vmul`（乘法）等
- **操作数类型**：
  - `vv`：向量-向量操作
  - `vx`：向量-标量操作
  - `v`：单向量操作
- **数据类型**：
  - `i8/i16/i32/i64`：有符号整数
  - `u8/u16/u32/u64`：无符号整数
  - `f32/f64`：浮点数
- **LMUL**：寄存器组数量（m1, m2, m4, m8）

示例：
- `__riscv_vle32_v_i32m8`：加载 32 位有符号整数向量，使用 8 个寄存器组
- `__riscv_vadd_vx_i32m4`：向量与标量相加，32 位整数，4 个寄存器组
- `__riscv_vmul_vv_f32m2`：向量乘法，32 位浮点数，2 个寄存器组

## 7.3.4 机器学习库

### TensorFlow Lite Micro

CoralNPU 集成了 TensorFlow Lite Micro（TFLite Micro），这是一个专为微控制器和嵌入式设备设计的轻量级机器学习推理库。

**什么是 TFLite Micro？**
- TensorFlow Lite 的精简版本
- 不需要操作系统支持
- 内存占用小，适合嵌入式设备
- 支持量化模型，推理速度快

### 模型推理基本流程

```c
#include "tensorflow/lite/micro/micro_interpreter.h"
#include "tensorflow/lite/micro/micro_mutable_op_resolver.h"
#include "tensorflow/lite/schema/schema_generated.h"

// 1. 准备模型数据（通常存储在 Flash 中）
extern const unsigned char model_data[];
extern const int model_data_len;

// 2. 分配内存（tensor arena）
constexpr int kTensorArenaSize = 10 * 1024;  // 10KB
uint8_t tensor_arena[kTensorArenaSize];

// 3. 加载模型
const tflite::Model* model = tflite::GetModel(model_data);

// 4. 注册需要的算子（操作）
tflite::MicroMutableOpResolver<5> resolver;
resolver.AddFullyConnected();  // 全连接层
resolver.AddConv2D();          // 2D 卷积
resolver.AddDepthwiseConv2D(); // 深度可分离卷积
resolver.AddReshape();         // 重塑
resolver.AddSoftmax();         // Softmax 激活

// 5. 创建解释器
tflite::MicroInterpreter interpreter(
    model, resolver, tensor_arena, kTensorArenaSize);

// 6. 分配张量内存
interpreter.AllocateTensors();

// 7. 获取输入张量
TfLiteTensor* input = interpreter.input(0);

// 8. 填充输入数据
for (int i = 0; i < input->bytes; i++) {
    input->data.int8[i] = input_data[i];
}

// 9. 执行推理
interpreter.Invoke();

// 10. 获取输出结果
TfLiteTensor* output = interpreter.output(0);
int8_t* output_data = output->data.int8;
```

**解释：**
- **模型数据**：训练好的神经网络模型，通常以 `.tflite` 格式存储
- **Tensor Arena**：用于存储中间计算结果的内存区域
- **算子（Operator）**：神经网络中的操作，如卷积、全连接等
- **张量（Tensor）**：多维数组，用于存储数据
- **推理（Inference）**：使用训练好的模型进行预测

### 量化支持

TFLite Micro 支持量化模型，可以大幅减少内存占用和提高推理速度。

**什么是量化？**
- 将 32 位浮点数转换为 8 位整数
- 模型大小减少约 4 倍
- 推理速度提升，功耗降低
- 精度略有下降，但通常可以接受

```c
// 量化模型的输入输出通常是 int8 类型
TfLiteTensor* input = interpreter.input(0);

// 检查输入类型
if (input->type == kTfLiteInt8) {
    // 量化模型
    // 输入范围通常是 -128 到 127
    input->data.int8[0] = quantized_value;
} else if (input->type == kTfLiteFloat32) {
    // 浮点模型
    input->data.f[0] = float_value;
}
```

### CoralNPU 优化的算子

CoralNPU 为 TFLite Micro 提供了向量优化的算子实现，位于 `/home/curry/code/coralnpu/sw/opt/litert-micro/` 目录。

```c
#include "sw/opt/litert-micro/depthwise_conv.h"

// 使用 CoralNPU 优化的深度可分离卷积
namespace coralnpu_v2::opt::litert_micro {
    TFLMRegistration Register_DEPTHWISE_CONV_2D();
}

// 在注册算子时使用优化版本
resolver.AddCustom("DEPTHWISE_CONV_2D", 
                   coralnpu_v2::opt::litert_micro::Register_DEPTHWISE_CONV_2D());
```

这些优化的算子使用了 RISC-V 向量指令，可以显著提升推理性能。

## 7.3.5 自定义库

### 如何编写自己的库

#### 库的基本结构

一个库通常包含头文件（.h）和实现文件（.c 或 .cpp）。

**头文件（mylib.h）：**
```c
#ifndef MYLIB_H
#define MYLIB_H

// 函数声明
int add(int a, int b);
int multiply(int a, int b);

#endif  // MYLIB_H
```

**实现文件（mylib.c）：**
```c
#include "mylib.h"

int add(int a, int b) {
    return a + b;
}

int multiply(int a, int b) {
    return a * b;
}
```

**使用库（main.c）：**
```c
#include <stdio.h>
#include "mylib.h"

int main() {
    int result1 = add(3, 4);
    int result2 = multiply(5, 6);
    printf("3 + 4 = %d\n", result1);
    printf("5 * 6 = %d\n", result2);
    return 0;
}
```

#### 头文件保护

```c
#ifndef MYLIB_H
#define MYLIB_H
// ... 头文件内容 ...
#endif
```

**解释：**
- 这叫做"头文件保护"（header guard）
- 防止头文件被重复包含
- `#ifndef`：如果没有定义
- `#define`：定义一个宏
- `#endif`：结束条件编译

如果没有头文件保护，同一个头文件被包含多次会导致编译错误。

### 库的组织

#### 目录结构

```
my_project/
├── include/          # 头文件目录
│   └── mylib.h
├── src/              # 源文件目录
│   └── mylib.c
├── examples/         # 示例代码
│   └── example.c
└── BUILD             # Bazel 构建文件
```

#### 使用 Bazel 构建库

CoralNPU 项目使用 Bazel 作为构建系统。

**BUILD 文件示例：**
```python
# 定义一个 C 库
cc_library(
    name = "mylib",
    srcs = ["src/mylib.c"],      # 源文件
    hdrs = ["include/mylib.h"],  # 头文件
    visibility = ["//visibility:public"],  # 可见性：公开
)

# 定义一个使用该库的可执行文件
cc_binary(
    name = "example",
    srcs = ["examples/example.c"],
    deps = [":mylib"],  # 依赖 mylib
)
```

**解释：**
- `cc_library`：定义一个 C/C++ 库
- `cc_binary`：定义一个可执行文件
- `srcs`：源文件列表
- `hdrs`：头文件列表
- `deps`：依赖的其他库
- `visibility`：控制哪些目标可以使用这个库

#### 编译和使用

```bash
# 编译库
bazel build //path/to:mylib

# 编译并运行示例
bazel run //path/to:example
```

### 编写向量优化的库

如果你想利用 CoralNPU 的向量处理能力，可以编写使用 RVV 指令的库。

**示例：向量化的数组求和**

```c
#include <riscv_vector.h>
#include <stddef.h>
#include <stdint.h>

// 向量化的数组求和函数
int32_t vector_sum(const int32_t* array, size_t n) {
    int32_t sum = 0;
    size_t vl;
    
    // 使用向量寄存器累加
    vint32m8_t vsum = __riscv_vmv_v_x_i32m8(0, 1);  // 初始化为 0
    
    while (n > 0) {
        vl = __riscv_vsetvl_e32m8(n);
        
        // 加载数据
        vint32m8_t vdata = __riscv_vle32_v_i32m8(array, vl);
        
        // 累加到 vsum
        vsum = __riscv_vadd_vv_i32m8(vsum, vdata, vl);
        
        array += vl;
        n -= vl;
    }
    
    // 将向量寄存器中的所有元素相加
    vint32m1_t vsum_reduced = __riscv_vmv_v_x_i32m1(0, 1);
    vsum_reduced = __riscv_vredsum_vs_i32m8_i32m1(vsum_reduced, vsum, vsum_reduced, vl);
    
    // 提取结果
    sum = __riscv_vmv_x_s_i32m1_i32(vsum_reduced);
    
    return sum;
}
```

**解释：**
- `__riscv_vmv_v_x_i32m8(0, 1)`：将标量 0 广播到向量寄存器
- `__riscv_vredsum_vs_i32m8_i32m1`：向量归约求和
  - `vredsum`：向量归约求和指令
  - `vs`：向量-标量操作
- `__riscv_vmv_x_s_i32m1_i32`：从向量寄存器提取标量值

### 库的最佳实践

1. **清晰的接口**：函数命名要有意义，参数要明确
2. **错误处理**：检查输入参数，返回错误码
3. **文档注释**：为每个函数添加注释说明
4. **测试**：编写测试用例验证功能
5. **性能优化**：对性能关键的部分使用向量指令

**示例：带错误处理的函数**

```c
#include <stddef.h>
#include <stdint.h>

/**
 * 计算两个数组的点积
 * 
 * @param a 第一个数组
 * @param b 第二个数组
 * @param n 数组长度
 * @param result 输出结果的指针
 * @return 0 表示成功，-1 表示参数错误
 */
int dot_product(const float* a, const float* b, size_t n, float* result) {
    // 参数检查
    if (a == NULL || b == NULL || result == NULL) {
        return -1;  // 空指针错误
    }
    
    if (n == 0) {
        *result = 0.0f;
        return 0;
    }
    
    // 计算点积
    float sum = 0.0f;
    for (size_t i = 0; i < n; i++) {
        sum += a[i] * b[i];
    }
    
    *result = sum;
    return 0;  // 成功
}
```

## 总结

本章介绍了 CoralNPU 项目中可用的各种库：

1. **标准库**：提供基本的 C 函数，如 printf、malloc、memcpy 等
2. **数学库**：提供数学运算函数，如 sin、cos、sqrt、pow 等
3. **向量库**：使用 RISC-V 向量扩展加速数据处理
4. **机器学习库**：TFLite Micro 用于神经网络推理
5. **自定义库**：如何编写和组织自己的库

掌握这些库的使用，可以帮助你更高效地开发 CoralNPU 应用程序。
