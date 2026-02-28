# 第十二章：ML 应用

## 12.1 ML 应用概述

### 12.1.1 什么是机器学习（ML）

机器学习（Machine Learning，ML）是人工智能的一个分支，它让计算机能够从数据中学习规律，而不需要明确编程。简单来说：

- **传统编程**：人写规则 → 计算机执行规则 → 得到结果
- **机器学习**：给计算机数据和结果 → 计算机学习规则 → 用规则处理新数据

举个例子：
- 传统方式识别猫：你需要写代码描述"猫有尖耳朵、胡须、四条腿..."
- 机器学习方式：给计算机看 1000 张猫的图片，它自己学会什么是猫

### 12.1.2 CoralNPU 支持的 ML 应用

CoralNPU 是一个专门为机器学习设计的硬件加速器，它可以加速以下类型的 ML 应用：

1. **图像分类**
   - 识别图片中的物体（猫、狗、汽车等）
   - 应用场景：相册分类、商品识别

2. **目标检测**
   - 找出图片中物体的位置和类别
   - 应用场景：自动驾驶、安防监控

3. **语义分割**
   - 将图片中每个像素分类
   - 应用场景：医学影像分析、背景虚化

4. **自然语言处理**
   - 文本生成、对话系统
   - 应用场景：智能助手、文本摘要

### 12.1.3 CoralNPU 的优势

CoralNPU 相比传统 CPU 运行 ML 模型有以下优势：

1. **速度快**：专门的硬件加速，比 CPU 快 10-100 倍
2. **功耗低**：专用硬件比通用 CPU 更节能
3. **实时性好**：可以实时处理视频流
4. **成本低**：不需要昂贵的 GPU

### 12.1.4 适用场景

CoralNPU 特别适合以下场景：

- **边缘设备**：手机、摄像头、机器人等资源受限的设备
- **实时应用**：需要快速响应的应用（如人脸识别门禁）
- **低功耗场景**：电池供电的设备
- **小型模型**：参数量在几百万到几十亿之间的模型

## 12.2 支持的模型

### 12.2.1 什么是神经网络模型

神经网络模型是机器学习的核心。它模仿人脑的神经元结构：

```
输入层 → 隐藏层 → 隐藏层 → ... → 输出层
```

每一层都包含很多"神经元"，它们通过"权重"连接。训练模型就是调整这些权重，让模型能正确预测。

**关键概念**：
- **参数（Parameters）**：模型中的权重数量，如"15M 参数"表示 1500 万个权重
- **层（Layer）**：神经网络的一层，包含多个神经元
- **激活函数（Activation）**：决定神经元是否"激活"的函数

### 12.2.2 MobileNet

MobileNet 是一个轻量级的图像分类模型，专为移动设备设计。

**特点**：
- 参数量小（约 4M）
- 速度快
- 准确率较高（ImageNet Top-1 约 70%）

**应用场景**：
- 实时图像分类
- 移动应用中的物体识别

**模型结构**：
```
输入图像 (224x224x3)
    ↓
卷积层 + 批归一化 + ReLU
    ↓
深度可分离卷积 × 13
    ↓
全局平均池化
    ↓
全连接层
    ↓
Softmax（输出 1000 类概率）
```

**深度可分离卷积**是 MobileNet 的核心技术，它将标准卷积分解为两步：
1. 深度卷积（Depthwise Convolution）：每个通道单独卷积
2.逐点卷积（Pointwise Convolution）：1x1 卷积混合通道

这样可以大幅减少计算量。

### 12.2.3 ResNet

ResNet（残差网络）是一个深层神经网络，通过"跳跃连接"解决了深层网络难以训练的问题。

**特点**：
- 可以训练非常深的网络（50 层、101 层、152 层）
- 准确率高（ImageNet Top-1 约 76%）
- 参数量较大（ResNet-50 约 25M）

**残差块（Residual Block）**：
```
输入 x
    ↓
卷积 + BN + ReLU
    ↓
卷积 + BN
    ↓
加上输入 x（跳跃连接）
    ↓
ReLU
    ↓
输出
```

**为什么需要跳跃连接**？
- 深层网络容易出现"梯度消失"问题（训练时梯度变得很小）
- 跳跃连接让梯度可以直接传播，解决了这个问题

### 12.2.4 YOLO

YOLO（You Only Look Once）是一个实时目标检测模型。

**特点**：
- 速度非常快（可以达到 30+ FPS）
- 一次前向传播就能检测所有物体
- 准确率较高

**工作原理**：
1. 将图像分成 S×S 的网格
2. 每个网格预测 B 个边界框（Bounding Box）
3. 每个边界框预测：
   - 位置（x, y, w, h）
   - 置信度（Confidence）
   - 类别概率

**输出格式**：
```
[x, y, w, h, confidence, class1_prob, class2_prob, ...]
```

**YOLO 版本**：
- YOLOv1：最初版本
- YOLOv2/YOLO9000：改进版
- YOLOv3：使用多尺度预测
- YOLOv4/v5：进一步优化
- YOLOv8：最新版本

### 12.2.5 Llama2

Llama2 是 Meta（Facebook）开源的大语言模型（LLM）。

**特点**：
- 基于 Transformer 架构
- 支持多种规模（7B、13B、70B 参数）
- 可以进行文本生成、对话等任务

**Transformer 架构**：
```
输入文本（Token 序列）
    ↓
Token Embedding + 位置编码
    ↓
Transformer 层 × N
    ├─ 多头自注意力（Multi-Head Attention）
    ├─ 残差连接 + LayerNorm
    ├─ 前馈网络（FFN）
    └─ 残差连接 + LayerNorm
    ↓
输出层（预测下一个 Token）
```

**关键概念**：
- **Token**：文本的基本单位，一个词或词的一部分
- **Embedding**：将 Token 转换为向量
- **注意力机制（Attention）**：让模型关注输入中的重要部分
- **自回归（Autoregressive）**：根据前面的 Token 预测下一个 Token

**Llama2 的改进**：
- 使用 RMSNorm 代替 LayerNorm（更快）
- 使用 RoPE 位置编码（更好的长度泛化）
- 使用 SwiGLU 激活函数（更好的性能）
- 使用 Grouped-Query Attention（减少 KV Cache）

### 12.2.6 其他模型

CoralNPU 还支持其他常见模型：

1. **EfficientNet**
   - 通过神经架构搜索（NAS）找到的高效模型
   - 在准确率和效率之间取得很好的平衡

2. **SqueezeNet**
   - 极小的模型（< 5MB）
   - 适合极度资源受限的设备

3. **Inception**
   - 使用多尺度卷积核
   - 可以捕获不同尺度的特征

4. **Transformer 系列**
   - BERT：双向编码器，适合理解任务
   - GPT：单向解码器，适合生成任务
   - T5：编码器-解码器，适合翻译等任务

## 12.3 模型部署

### 12.3.1 模型部署流程

将训练好的模型部署到 CoralNPU 需要以下步骤：

```
1. 训练模型（PyTorch/TensorFlow）
    ↓
2. 导出模型（ONNX/SavedModel）
    ↓
3. 模型转换（转换为 TFLite）
    ↓
4. 模型量化（INT8 量化）
    ↓
5. 编译模型（生成 CoralNPU 可执行文件）
    ↓
6. 部署到设备
```

### 12.3.2 模型转换

**什么是模型转换**？

训练框架（如 PyTorch）使用的模型格式不能直接在嵌入式设备上运行，需要转换为轻量级格式。

**常见格式**：
- **ONNX**：开放神经网络交换格式，跨框架通用
- **TFLite**：TensorFlow Lite，专为移动和嵌入式设备设计
- **CoreML**：苹果设备专用格式

**PyTorch 模型转换示例**：

```python
import torch
import torch.onnx

# 1. 加载 PyTorch 模型
model = torch.load('model.pth')
model.eval()

# 2. 准备示例输入
dummy_input = torch.randn(1, 3, 224, 224)

# 3. 导出为 ONNX
torch.onnx.export(
    model,
    dummy_input,
    'model.onnx',
    input_names=['input'],
    output_names=['output'],
    dynamic_axes={
        'input': {0: 'batch_size'},
        'output': {0: 'batch_size'}
    }
)
```

**ONNX 转 TFLite**：

```bash
# 使用 onnx-tf 工具
pip install onnx-tf

# 转换
onnx-tf convert -i model.onnx -o model_tf

# 转换为 TFLite
import tensorflow as tf

converter = tf.lite.TFLiteConverter.from_saved_model('model_tf')
tflite_model = converter.convert()

with open('model.tflite', 'wb') as f:
    f.write(tflite_model)
```

### 12.3.3 模型优化

模型优化可以减小模型大小、提高推理速度。

**常见优化技术**：

1. **剪枝（Pruning）**
   - 移除不重要的权重（设为 0）
   - 可以减少 30-50% 的参数量
   - 准确率下降很小（< 1%）

2. **知识蒸馏（Knowledge Distillation）**
   - 用大模型（教师）训练小模型（学生）
   - 学生模型学习教师的"软标签"
   - 可以得到更好的小模型

3. **算子融合（Operator Fusion）**
   - 将多个操作合并为一个
   - 减少内存访问，提高速度
   - 例如：Conv + BN + ReLU → 融合算子

4. **层融合（Layer Fusion）**
   - 将多个层合并
   - 减少中间结果的存储

**TFLite 优化示例**：

```python
import tensorflow as tf

converter = tf.lite.TFLiteConverter.from_saved_model('model')

# 启用优化
converter.optimizations = [tf.lite.Optimize.DEFAULT]

# 转换
tflite_model = converter.convert()
```

### 12.3.4 部署流程详解

**步骤 1：准备模型**

```python
# 训练完成后保存模型
model.save('my_model.h5')
```

**步骤 2：转换为 TFLite**

```python
import tensorflow as tf

# 加载模型
model = tf.keras.models.load_model('my_model.h5')

# 转换
converter = tf.lite.TFLiteConverter.from_keras_model(model)
converter.optimizations = [tf.lite.Optimize.DEFAULT]

# 量化（见下一节）
converter.target_spec.supported_types = [tf.int8]

tflite_model = converter.convert()

# 保存
with open('model_quantized.tflite', 'wb') as f:
    f.write(tflite_model)
```

**步骤 3：验证模型**

```python
import numpy as np

# 加载 TFLite 模型
interpreter = tf.lite.Interpreter(model_path='model_quantized.tflite')
interpreter.allocate_tensors()

# 获取输入输出信息
input_details = interpreter.get_input_details()
output_details = interpreter.get_output_details()

# 准备测试数据
test_input = np.random.randn(1, 224, 224, 3).astype(np.float32)

# 推理
interpreter.set_tensor(input_details[0]['index'], test_input)
interpreter.invoke()
output = interpreter.get_tensor(output_details[0]['index'])

print('输出形状:', output.shape)
print('输出:', output)
```

**步骤 4：部署到 CoralNPU**

```c
// C 代码示例
#include "tensorflow/lite/micro/micro_interpreter.h"
#include "tensorflow/lite/micro/micro_mutable_op_resolver.h"
#include "model_data.h"  // 包含模型数据

// 1. 设置算子解析器
tflite::MicroMutableOpResolver<5> resolver;
resolver.AddConv2D();
resolver.AddMaxPool2D();
resolver.AddFullyConnected();
resolver.AddReshape();
resolver.AddSoftmax();

// 2. 创建解释器
constexpr int kTensorArenaSize = 100 * 1024;  // 100KB
uint8_t tensor_arena[kTensorArenaSize];

tflite::MicroInterpreter interpreter(
    model_data,
    resolver,
    tensor_arena,
    kTensorArenaSize
);

// 3. 分配张量
interpreter.AllocateTensors();

// 4. 获取输入张量
TfLiteTensor* input = interpreter.input(0);

// 5. 填充输入数据
for (int i = 0; i < input->bytes; i++) {
    input->data.int8[i] = /* 你的输入数据 */;
}

// 6. 运行推理
interpreter.Invoke();

// 7. 获取输出
TfLiteTensor* output = interpreter.output(0);
int8_t* output_data = output->data.int8;
```


## 12.4 量化

### 12.4.1 什么是量化

**量化（Quantization）**是将模型的权重和激活值从高精度（如 float32）转换为低精度（如 int8）的过程。

**为什么需要量化**？

1. **减小模型大小**
   - float32：每个参数 4 字节
   - int8：每个参数 1 字节
   - 模型大小减少 4 倍

2. **提高推理速度**
   - 整数运算比浮点运算快
   - 可以使用专门的整数加速器

3. **降低功耗**
   - 整数运算功耗更低
   - 适合移动设备

4. **减少内存带宽**
   - 数据量减少，内存访问更快

**量化的代价**：
- 精度略有下降（通常 < 1%）
- 需要额外的量化/反量化操作

### 12.4.2 量化原理

**量化公式**：

```
量化：q = round(r / scale) + zero_point
反量化：r = (q - zero_point) * scale
```

其中：
- `r`：实数值（float32）
- `q`：量化值（int8）
- `scale`：缩放因子
- `zero_point`：零点偏移

**示例**：

假设我们要将 [-10.0, 10.0] 的浮点数量化到 int8 [-128, 127]：

```
scale = (10.0 - (-10.0)) / (127 - (-128)) = 20.0 / 255 ≈ 0.0784
zero_point = 0

量化 5.0：
q = round(5.0 / 0.0784) + 0 = round(63.78) = 64

反量化 64：
r = (64 - 0) * 0.0784 = 5.0176 ≈ 5.0
```

**对称量化 vs 非对称量化**：

1. **对称量化**（zero_point = 0）
   - 简单，计算快
   - 适合权重（通常对称分布）

2. **非对称量化**（zero_point ≠ 0）
   - 更精确
   - 适合激活值（可能不对称）

### 12.4.3 INT8 量化

INT8 量化是最常用的量化方式，将 float32 转换为 int8。

**量化方案**：

1. **仅权重量化（Weight-only Quantization）**
   - 只量化权重，激活值保持 float32
   - 减小模型大小，但速度提升有限

2. **全整数量化（Full Integer Quantization）**
   - 权重和激活值都量化为 int8
   - 模型大小和速度都有提升
   - 需要校准数据集

**TFLite INT8 量化示例**：

```python
import tensorflow as tf
import numpy as np

# 准备代表性数据集（用于校准）
def representative_dataset():
    for _ in range(100):
        # 生成随机输入（实际应用中应使用真实数据）
        data = np.random.randn(1, 224, 224, 3).astype(np.float32)
        yield [data]

# 加载模型
model = tf.keras.models.load_model('my_model.h5')

# 创建转换器
converter = tf.lite.TFLiteConverter.from_keras_model(model)

# 设置优化选项
converter.optimizations = [tf.lite.Optimize.DEFAULT]

# 设置为全整数量化
converter.representative_dataset = representative_dataset
converter.target_spec.supported_ops = [tf.lite.OpsSet.TFLITE_BUILTINS_INT8]
converter.inference_input_type = tf.int8
converter.inference_output_type = tf.int8

# 转换
tflite_quant_model = converter.convert()

# 保存
with open('model_int8.tflite', 'wb') as f:
    f.write(tflite_quant_model)
```

**量化效果对比**：

| 模型 | 大小 | 推理时间 | 准确率 |
|------|------|----------|--------|
| Float32 | 100 MB | 100 ms | 75.0% |
| INT8 | 25 MB | 30 ms | 74.5% |

### 12.4.4 量化感知训练（QAT）

**量化感知训练（Quantization-Aware Training, QAT）**在训练时就模拟量化效果，让模型适应量化误差。

**原理**：
1. 在前向传播时插入"伪量化"节点
2. 模拟量化和反量化过程
3. 反向传播时使用直通估计器（Straight-Through Estimator）

**伪量化**：
```
x_quant = fake_quant(x) = dequant(quant(x))
```

**优点**：
- 准确率损失更小（< 0.5%）
- 适合对精度要求高的应用

**缺点**：
- 需要重新训练模型
- 训练时间更长

**TensorFlow QAT 示例**：

```python
import tensorflow as tf
import tensorflow_model_optimization as tfmot

# 加载预训练模型
model = tf.keras.models.load_model('my_model.h5')

# 应用量化感知训练
quantize_model = tfmot.quantization.keras.quantize_model

q_aware_model = quantize_model(model)

# 编译模型
q_aware_model.compile(
    optimizer='adam',
    loss='sparse_categorical_crossentropy',
    metrics=['accuracy']
)

# 训练（微调）
q_aware_model.fit(
    train_dataset,
    epochs=5,
    validation_data=val_dataset
)

# 转换为 TFLite
converter = tf.lite.TFLiteConverter.from_keras_model(q_aware_model)
converter.optimizations = [tf.lite.Optimize.DEFAULT]
tflite_model = converter.convert()
```

**QAT 训练技巧**：
1. 从预训练模型开始（不要从头训练）
2. 使用较小的学习率（如原来的 1/10）
3. 训练 5-10 个 epoch 即可
4. 使用余弦退火学习率调度

### 12.4.5 后训练量化（PTQ）

**后训练量化（Post-Training Quantization, PTQ）**在训练完成后直接量化模型，不需要重新训练。

**优点**：
- 简单快速
- 不需要训练数据和训练过程

**缺点**：
- 准确率损失较大（1-2%）
- 对某些模型效果不好

**PTQ 类型**：

1. **动态范围量化（Dynamic Range Quantization）**
   - 只量化权重
   - 激活值在推理时动态量化
   - 最简单的方法

```python
converter = tf.lite.TFLiteConverter.from_keras_model(model)
converter.optimizations = [tf.lite.Optimize.DEFAULT]
tflite_model = converter.convert()
```

2. **全整数量化（Full Integer Quantization）**
   - 权重和激活值都量化
   - 需要代表性数据集

```python
def representative_dataset():
    for data in calibration_data:
        yield [data]

converter = tf.lite.TFLiteConverter.from_keras_model(model)
converter.optimizations = [tf.lite.Optimize.DEFAULT]
converter.representative_dataset = representative_dataset
converter.target_spec.supported_ops = [tf.lite.OpsSet.TFLITE_BUILTINS_INT8]
tflite_model = converter.convert()
```

3. **Float16 量化**
   - 量化为 float16
   - 精度损失很小
   - 模型大小减半

```python
converter = tf.lite.TFLiteConverter.from_keras_model(model)
converter.optimizations = [tf.lite.Optimize.DEFAULT]
converter.target_spec.supported_types = [tf.float16]
tflite_model = converter.convert()
```

**选择 QAT 还是 PTQ**？

| 场景 | 推荐方法 |
|------|----------|
| 快速原型 | PTQ |
| 精度要求高 | QAT |
| 没有训练数据 | PTQ |
| 有充足时间 | QAT |
| 模型对量化敏感 | QAT |

### 12.4.6 量化调试

**常见问题**：

1. **量化后精度下降太多**
   - 检查是否使用了代表性数据集
   - 尝试 QAT
   - 检查是否有不支持的算子

2. **某些层量化效果差**
   - 跳过敏感层（保持 float32）
   - 使用混合精度量化

3. **量化后模型不收敛**
   - 降低学习率
   - 增加训练轮数
   - 检查 BatchNorm 层

**混合精度量化示例**：

```python
# 定义哪些层不量化
def apply_quantization(layer):
    # 跳过第一层和最后一层
    if isinstance(layer, tf.keras.layers.InputLayer):
        return layer
    if layer.name == 'output_layer':
        return layer
    # 其他层量化
    return tfmot.quantization.keras.quantize_annotate_layer(layer)

# 应用到模型
annotated_model = tf.keras.models.clone_model(
    model,
    clone_function=apply_quantization,
)

q_aware_model = tfmot.quantization.keras.quantize_apply(annotated_model)
```

## 12.5 TFLite Micro

### 12.5.1 TFLite Micro 简介

**TensorFlow Lite Micro（TFLite Micro）**是 TensorFlow Lite 的微控制器版本，专为嵌入式设备设计。

**特点**：
- 极小的内存占用（< 20KB）
- 不依赖操作系统
- 纯 C++ 实现
- 支持常见的 ML 算子

**与 TFLite 的区别**：

| 特性 | TFLite | TFLite Micro |
|------|--------|--------------|
| 目标平台 | 移动设备 | 微控制器 |
| 内存需求 | MB 级 | KB 级 |
| 依赖 | 标准库 | 无依赖 |
| 动态内存 | 支持 | 不支持 |
| 文件系统 | 需要 | 不需要 |

### 12.5.2 TFLite Micro 架构

**核心组件**：

```
┌─────────────────────────────────────┐
│         应用代码                     │
├─────────────────────────────────────┤
│      MicroInterpreter               │  ← 解释器
├─────────────────────────────────────┤
│      MicroOpResolver                │  ← 算子解析器
├─────────────────────────────────────┤
│      Kernel Implementations         │  ← 算子实现
├─────────────────────────────────────┤
│      Memory Management              │  ← 内存管理
└─────────────────────────────────────┘
```

**内存管理**：
- 使用静态分配的 Tensor Arena
- 不使用动态内存（malloc/free）
- 所有内存在编译时确定

### 12.5.3 如何使用 TFLite Micro

**步骤 1：准备模型**

```python
# 转换为 TFLite 模型
converter = tf.lite.TFLiteConverter.from_keras_model(model)
converter.optimizations = [tf.lite.Optimize.DEFAULT]
tflite_model = converter.convert()

# 保存
with open('model.tflite', 'wb') as f:
    f.write(tflite_model)
```

**步骤 2：转换为 C 数组**

```bash
# 使用 xxd 工具
xxd -i model.tflite > model_data.cc
```

生成的文件：
```cpp
// model_data.cc
unsigned char model_tflite[] = {
  0x1c, 0x00, 0x00, 0x00, 0x54, 0x46, 0x4c, 0x33,
  // ... 更多数据
};
unsigned int model_tflite_len = 12345;
```

**步骤 3：编写推理代码**

```cpp
#include "tensorflow/lite/micro/micro_interpreter.h"
#include "tensorflow/lite/micro/micro_mutable_op_resolver.h"
#include "tensorflow/lite/micro/system_setup.h"
#include "tensorflow/lite/schema/schema_generated.h"
#include "model_data.h"

// 1. 设置算子解析器
tflite::MicroMutableOpResolver<3> micro_op_resolver;
micro_op_resolver.AddFullyConnected();
micro_op_resolver.AddSoftmax();
micro_op_resolver.AddReshape();

// 2. 加载模型
const tflite::Model* model = tflite::GetModel(model_tflite);
if (model->version() != TFLITE_SCHEMA_VERSION) {
  printf("模型版本不匹配!\n");
  return;
}

// 3. 分配 Tensor Arena
constexpr int kTensorArenaSize = 10 * 1024;  // 10KB
alignas(16) uint8_t tensor_arena[kTensorArenaSize];

// 4. 创建解释器
tflite::MicroInterpreter interpreter(
    model,
    micro_op_resolver,
    tensor_arena,
    kTensorArenaSize
);

// 5. 分配张量
TfLiteStatus allocate_status = interpreter.AllocateTensors();
if (allocate_status != kTfLiteOk) {
  printf("分配张量失败!\n");
  return;
}

// 6. 获取输入张量
TfLiteTensor* input = interpreter.input(0);

// 7. 填充输入数据
for (int i = 0; i < input->bytes; i++) {
  input->data.int8[i] = /* 你的输入数据 */;
}

// 8. 运行推理
TfLiteStatus invoke_status = interpreter.Invoke();
if (invoke_status != kTfLiteOk) {
  printf("推理失败!\n");
  return;
}

// 9. 获取输出
TfLiteTensor* output = interpreter.output(0);
int8_t* output_data = output->data.int8;

// 10. 处理输出
for (int i = 0; i < output->dims->data[1]; i++) {
  printf("类别 %d: %d\n", i, output_data[i]);
}
```

**步骤 4：编译**

```makefile
# Makefile
TFLITE_MICRO_PATH = third_party/tflite-micro

INCLUDES = \
  -I$(TFLITE_MICRO_PATH) \
  -I$(TFLITE_MICRO_PATH)/tensorflow/lite/micro

CXXFLAGS = -std=c++11 -O3 $(INCLUDES)

main: main.cc model_data.cc
	$(CXX) $(CXXFLAGS) $^ -o $@
```

### 12.5.4 算子支持

TFLite Micro 支持的常见算子：

**基础算子**：
- Add, Sub, Mul, Div
- ReLU, ReLU6, Sigmoid, Tanh
- Softmax, LogSoftmax

**卷积算子**：
- Conv2D
- DepthwiseConv2D
- TransposeConv2D

**池化算子**：
- MaxPool2D
- AveragePool2D

**全连接**：
- FullyConnected

**归一化**：
- BatchNormalization
- L2Normalization

**其他**：
- Reshape, Transpose
- Concatenation, Split
- Pad, Slice

**查看模型使用的算子**：

```python
import tensorflow as tf

# 加载模型
interpreter = tf.lite.Interpreter(model_path='model.tflite')
interpreter.allocate_tensors()

# 获取算子列表
ops = set()
for op in interpreter._get_ops_details():
    ops.add(op['op_name'])

print('模型使用的算子:')
for op in sorted(ops):
    print(f'  - {op}')
```

**添加自定义算子**：

```cpp
// 定义自定义算子
TfLiteStatus MyCustomOpEval(TfLiteContext* context, TfLiteNode* node) {
  // 实现算子逻辑
  return kTfLiteOk;
}

// 注册算子
TfLiteRegistration* Register_MY_CUSTOM_OP() {
  static TfLiteRegistration r = {
    nullptr,  // init
    nullptr,  // free
    nullptr,  // prepare
    MyCustomOpEval  // invoke
  };
  return &r;
}

// 在解析器中添加
micro_op_resolver.AddCustom("MyCustomOp", Register_MY_CUSTOM_OP());
```


### 12.5.5 内存优化

**确定 Tensor Arena 大小**：

```cpp
// 方法 1：逐步增加
constexpr int kTensorArenaSize = 10 * 1024;  // 从 10KB 开始
// 如果分配失败，增加大小

// 方法 2：使用 arena_used_bytes()
interpreter.AllocateTensors();
size_t used_bytes = interpreter.arena_used_bytes();
printf("实际使用: %zu 字节\n", used_bytes);
```

**减少内存使用**：

1. **使用量化模型**（INT8 比 Float32 小 4 倍）
2. **减少模型复杂度**（更少的层、更小的通道数）
3. **使用深度可分离卷积**（参数量更少）
4. **移除不必要的算子**

### 12.5.6 性能优化

**优化技巧**：

1. **使用优化的内核**
   - CMSIS-NN（ARM Cortex-M）
   - RISC-V 向量扩展（RVV）

2. **启用编译器优化**
   ```makefile
   CXXFLAGS += -O3 -march=native
   ```

3. **使用固定点运算**
   - INT8 比 Float32 快得多

4. **批处理**（如果内存允许）
   - 一次处理多个输入

**CoralNPU 加速示例**：

```cpp
// 使用 CoralNPU 加速卷积
#include "coralnpu/npu_ops.h"

// 在算子实现中调用 NPU
TfLiteStatus Conv2DEval(TfLiteContext* context, TfLiteNode* node) {
  // 获取输入输出
  const TfLiteTensor* input = GetInput(context, node, 0);
  const TfLiteTensor* filter = GetInput(context, node, 1);
  TfLiteTensor* output = GetOutput(context, node, 0);
  
  // 调用 NPU 加速
  npu_conv2d(
    input->data.int8,
    filter->data.int8,
    output->data.int8,
    input->dims,
    filter->dims,
    output->dims
  );
  
  return kTfLiteOk;
}
```

## 12.6 Llama2 示例

### 12.6.1 Llama2.c 项目

**Llama2.c** 是一个用纯 C 语言实现的 Llama2 推理引擎，由 Andrej Karpathy 开发。

**项目特点**：
- 纯 C 实现，只有一个文件（run.c，约 700 行）
- 不依赖任何库（除了标准 C 库）
- 支持 float32 和 int8 量化
- 可以在各种平台运行（包括微控制器）

**项目结构**：
```
llama2.c/
├── run.c              # 推理引擎（CPU 版本）
├── runq.c             # 推理引擎（INT8 量化版本）
├── train.py           # 训练脚本
├── export.py          # 模型导出脚本
├── model.py           # 模型定义
├── tokenizer.py       # 分词器
└── stories15M.bin     # 预训练模型（15M 参数）
```

### 12.6.2 Llama2 模型结构

**Transformer 层**：

```c
typedef struct {
    int dim;           // 模型维度（如 512）
    int hidden_dim;    // FFN 隐藏层维度（如 2048）
    int n_layers;      // 层数（如 6）
    int n_heads;       // 注意力头数（如 8）
    int n_kv_heads;    // KV 头数（可以小于 n_heads）
    int vocab_size;    // 词表大小（如 32000）
    int seq_len;       // 最大序列长度（如 512）
} Config;
```

**权重结构**：

```c
typedef struct {
    // Token Embedding
    float* token_embedding_table;  // (vocab_size, dim)
    
    // RMSNorm 权重
    float* rms_att_weight;  // (layer, dim)
    float* rms_ffn_weight;  // (layer, dim)
    
    // 注意力权重
    float* wq;  // (layer, dim, n_heads * head_size)
    float* wk;  // (layer, dim, n_kv_heads * head_size)
    float* wv;  // (layer, dim, n_kv_heads * head_size)
    float* wo;  // (layer, n_heads * head_size, dim)
    
    // FFN 权重
    float* w1;  // (layer, hidden_dim, dim)
    float* w2;  // (layer, dim, hidden_dim)
    float* w3;  // (layer, hidden_dim, dim)
    
    // 输出层
    float* rms_final_weight;  // (dim,)
    float* wcls;              // (vocab_size, dim)
} TransformerWeights;
```

**前向传播流程**：

```
输入 Token
    ↓
Token Embedding
    ↓
对于每一层：
    ├─ RMSNorm
    ├─ 多头自注意力
    │   ├─ Q = x @ Wq
    │   ├─ K = x @ Wk
    │   ├─ V = x @ Wv
    │   ├─ Attention = softmax(Q @ K^T / sqrt(d)) @ V
    │   └─ Output = Attention @ Wo
    ├─ 残差连接
    ├─ RMSNorm
    ├─ FFN
    │   ├─ gate = x @ W1
    │   ├─ up = x @ W3
    │   ├─ hidden = SwiGLU(gate, up)
    │   └─ output = hidden @ W2
    └─ 残差连接
    ↓
RMSNorm
    ↓
输出层（x @ Wcls）
    ↓
Softmax
    ↓
预测的 Token
```

### 12.6.3 在 CoralNPU 上运行 Llama2

CoralNPU 版本的 Llama2 集成了硬件加速，主要加速以下操作：

1. **矩阵乘法（GEMV）**：最耗时的操作
2. **RMSNorm**：归一化操作
3. **RoPE**：旋转位置编码
4. **Attention**：注意力计算
5. **SwiGLU**：激活函数

**CoralNPU 加速的 Llama2 代码**：

```c
// run.c（CoralNPU 版本）

#include "vortex.h"  // CoralNPU 驱动
#include "kernels/kernels.h"  // NPU 内核

// NPU 设备句柄
static vx_device_h device = NULL;

// NPU 内核缓冲区
static vx_buffer_h vx_gemv_buf = NULL;
static vx_buffer_h vx_rmsnorm_buf = NULL;
static vx_buffer_h vx_rope_buf = NULL;
static vx_buffer_h vx_attention_buf = NULL;
static vx_buffer_h vx_swiglu_buf = NULL;

// 初始化 NPU
void init_npu() {
    // 打开设备
    vx_dev_open(&device);
    
    // 加载内核
    vx_buf_alloc(device, "./kernels/build/gemv.vxbin", &vx_gemv_buf);
    vx_buf_alloc(device, "./kernels/build/rmsnorm.vxbin", &vx_rmsnorm_buf);
    vx_buf_alloc(device, "./kernels/build/rope.vxbin", &vx_rope_buf);
    vx_buf_alloc(device, "./kernels/build/attention.vxbin", &vx_attention_buf);
    vx_buf_alloc(device, "./kernels/build/swiglu.vxbin", &vx_swiglu_buf);
}

// NPU 加速的矩阵向量乘法
void matmul_npu(float* out, float* x, float* w, int n, int d) {
    // 分配 NPU 内存
    vx_buffer_h x_buf, w_buf, out_buf;
    vx_buf_alloc(device, d * sizeof(float), &x_buf);
    vx_buf_alloc(device, n * d * sizeof(float), &w_buf);
    vx_buf_alloc(device, n * sizeof(float), &out_buf);
    
    // 拷贝数据到 NPU
    vx_copy_to_dev(x_buf, x, d * sizeof(float), 0);
    vx_copy_to_dev(w_buf, w, n * d * sizeof(float), 0);
    
    // 设置内核参数
    vx_start(device);
    vx_set_arg(device, 0, sizeof(vx_buffer_h), &x_buf);
    vx_set_arg(device, 1, sizeof(vx_buffer_h), &w_buf);
    vx_set_arg(device, 2, sizeof(vx_buffer_h), &out_buf);
    vx_set_arg(device, 3, sizeof(int), &n);
    vx_set_arg(device, 4, sizeof(int), &d);
    
    // 运行内核
    vx_run(device, vx_gemv_buf);
    
    // 等待完成
    vx_ready_wait(device, VX_MAX_TIMEOUT);
    
    // 拷贝结果回 CPU
    vx_copy_from_dev(out, out_buf, n * sizeof(float), 0);
    
    // 释放内存
    vx_buf_release(x_buf);
    vx_buf_release(w_buf);
    vx_buf_release(out_buf);
}

// NPU 加速的 RMSNorm
void rmsnorm_npu(float* o, float* x, float* weight, int size) {
    // 类似的 NPU 调用
    // ...
}

// Transformer 前向传播
float* forward(Transformer* transformer, int token, int pos) {
    Config* p = &transformer->config;
    TransformerWeights* w = &transformer->weights;
    RunState* s = &transformer->state;
    float *x = s->x;
    int dim = p->dim;
    int hidden_dim = p->hidden_dim;
    int head_size = dim / p->n_heads;

    // 1. Token Embedding
    float* content_row = w->token_embedding_table + token * dim;
    memcpy(x, content_row, dim * sizeof(float));

    // 2. 对于每一层
    for (int l = 0; l < p->n_layers; l++) {
        // 2.1 RMSNorm（使用 NPU）
        rmsnorm_npu(s->xb, x, w->rms_att_weight + l * dim, dim);

        // 2.2 计算 Q, K, V（使用 NPU）
        matmul_npu(s->q, s->xb, w->wq + l * dim * dim, dim, dim);
        matmul_npu(s->k, s->xb, w->wk + l * dim * dim, dim, dim);
        matmul_npu(s->v, s->xb, w->wv + l * dim * dim, dim, dim);

        // 2.3 RoPE（使用 NPU）
        rope_npu(s->q, s->k, pos, head_size, p->n_heads);

        // 2.4 多头注意力（使用 NPU）
        attention_npu(s->xb, s->q, s->k, s->v, s->att, 
                      p->n_heads, head_size, dim, pos);

        // 2.5 输出投影（使用 NPU）
        matmul_npu(s->xb2, s->xb, w->wo + l * dim * dim, dim, dim);

        // 2.6 残差连接
        for (int i = 0; i < dim; i++) {
            x[i] += s->xb2[i];
        }

        // 2.7 FFN RMSNorm（使用 NPU）
        rmsnorm_npu(s->xb, x, w->rms_ffn_weight + l * dim, dim);

        // 2.8 FFN（使用 NPU）
        matmul_npu(s->hb, s->xb, w->w1 + l * dim * hidden_dim, hidden_dim, dim);
        matmul_npu(s->hb2, s->xb, w->w3 + l * dim * hidden_dim, hidden_dim, dim);
        
        // SwiGLU（使用 NPU）
        swiglu_npu(s->hb, s->hb2, hidden_dim);
        
        // 输出投影（使用 NPU）
        matmul_npu(s->xb, s->hb, w->w2 + l * hidden_dim * dim, dim, hidden_dim);

        // 2.9 残差连接
        for (int i = 0; i < dim; i++) {
            x[i] += s->xb[i];
        }
    }

    // 3. 最终 RMSNorm
    rmsnorm_npu(x, x, w->rms_final_weight, dim);

    // 4. 分类器
    matmul_npu(s->logits, x, w->wcls, p->vocab_size, dim);

    return s->logits;
}
```

**编译和运行**：

```bash
# 1. 编译（需要 Vortex 环境）
cd llama2.c
make runperf MODE=PERF

# 2. 下载模型
wget https://huggingface.co/karpathy/tinyllamas/resolve/main/stories15M.bin

# 3. 运行（使用 NPU）
LD_LIBRARY_PATH=$VORTEX_RT_PATH:$LD_LIBRARY_PATH \
VORTEX_DRIVER=simx \
./run stories15M.bin -n 256 -i "Once upon a time"
```

**运行参数**：
- `-n`：生成的 token 数量
- `-i`：输入提示（prompt）
- `-t`：温度（控制随机性，0.0-1.0）
- `-p`：top-p 采样阈值
- `-s`：随机种子

### 12.6.4 性能分析

**性能对比**：

| 平台 | 模型 | 推理速度 | 功耗 |
|------|------|----------|------|
| CPU (x86) | 15M | 10 tok/s | 15W |
| CPU (ARM) | 15M | 5 tok/s | 5W |
| CoralNPU | 15M | 50 tok/s | 2W |
| CPU (x86) | 110M | 2 tok/s | 20W |
| CoralNPU | 110M | 15 tok/s | 3W |

**性能瓶颈分析**：

使用性能分析工具：

```bash
# 启用性能分析
VORTEX_PROFILING=1 ./run stories15M.bin -a -n 32
```

输出示例：
```
=== 性能分析 ===
总时间: 1000 ms
  - Token Embedding: 10 ms (1%)
  - RMSNorm: 50 ms (5%)
  - GEMV: 600 ms (60%)
  - Attention: 200 ms (20%)
  - SwiGLU: 80 ms (8%)
  - 其他: 60 ms (6%)

NPU 利用率: 85%
内存带宽: 10 GB/s
```

**优化建议**：

1. **批处理**
   - 一次处理多个 token（如果内存允许）
   - 可以提高 NPU 利用率

2. **KV Cache**
   - 缓存之前计算的 Key 和 Value
   - 避免重复计算

3. **量化**
   - 使用 INT8 量化
   - 速度提升 2-3 倍

4. **算子融合**
   - 将多个操作合并
   - 减少内存访问

**KV Cache 实现**：

```c
// 缓存结构
typedef struct {
    float* key_cache;    // (n_layers, seq_len, dim)
    float* value_cache;  // (n_layers, seq_len, dim)
    int cache_pos;       // 当前缓存位置
} KVCache;

// 使用 KV Cache 的注意力计算
void attention_with_cache(RunState* s, KVCache* cache, int layer, int pos) {
    // 只计算当前位置的 K, V
    // 之前的 K, V 从缓存中读取
    
    // 保存当前 K, V 到缓存
    memcpy(cache->key_cache + layer * seq_len * dim + pos * dim,
           s->k, dim * sizeof(float));
    memcpy(cache->value_cache + layer * seq_len * dim + pos * dim,
           s->v, dim * sizeof(float));
    
    // 使用缓存的 K, V 计算注意力
    for (int t = 0; t <= pos; t++) {
        float* k = cache->key_cache + layer * seq_len * dim + t * dim;
        float* v = cache->value_cache + layer * seq_len * dim + t * dim;
        // 计算注意力分数
        // ...
    }
}
```

### 12.6.5 INT8 量化版本

**量化 Llama2 模型**：

```bash
# 1. 导出量化模型
python export.py stories15M_q80.bin --version 2 --checkpoint stories15M.pt

# 2. 运行量化版本
./runq stories15M_q80.bin -n 256
```

**量化实现**：

```c
// 量化参数
typedef struct {
    float scale;
    int8_t zero_point;
} QuantParams;

// 量化
int8_t quantize(float x, QuantParams* qp) {
    int32_t q = (int32_t)(x / qp->scale) + qp->zero_point;
    // 截断到 [-128, 127]
    if (q < -128) q = -128;
    if (q > 127) q = 127;
    return (int8_t)q;
}

// 反量化
float dequantize(int8_t q, QuantParams* qp) {
    return (q - qp->zero_point) * qp->scale;
}

// INT8 矩阵乘法
void matmul_int8(int8_t* out, int8_t* x, int8_t* w, 
                 QuantParams* x_qp, QuantParams* w_qp, QuantParams* out_qp,
                 int n, int d) {
    for (int i = 0; i < n; i++) {
        int32_t sum = 0;
        for (int j = 0; j < d; j++) {
            // INT8 乘法
            sum += (int32_t)x[j] * (int32_t)w[i * d + j];
        }
        // 反量化 + 量化
        float f = dequantize(sum, ...);
        out[i] = quantize(f, out_qp);
    }
}
```

**性能对比**：

| 版本 | 模型大小 | 推理速度 | 准确率 |
|------|----------|----------|--------|
| Float32 | 60 MB | 50 tok/s | 基准 |
| INT8 | 15 MB | 120 tok/s | -0.5% |

### 12.6.6 实际应用示例

**示例 1：故事生成**

```bash
./run stories15M.bin -n 256 -i "Once upon a time, there was a little girl named Lily."
```

输出：
```
Once upon a time, there was a little girl named Lily. She loved to play 
outside in the sunshine. One day, she saw a big tree with red apples. 
Lily wanted to pick an apple, but the tree was too tall. She tried to 
jump, but she couldn't reach. Then, a kind bird flew down and helped 
Lily. The bird picked an apple and gave it to Lily. Lily was so happy 
and thanked the bird. From that day on, Lily and the bird became best 
friends and played together every day.
```

**示例 2：对话系统**

```bash
./run stories15M.bin -m chat -i "Hello, how are you?"
```

输出：
```
User: Hello, how are you?
Assistant: I'm doing great! How can I help you today?
User: Tell me a story about a cat.
Assistant: Once upon a time, there was a curious cat named Whiskers...
```

**示例 3：代码生成**

```bash
./run codellama_7b.bin -i "def fibonacci(n):"
```

输出：
```python
def fibonacci(n):
    if n <= 1:
        return n
    else:
        return fibonacci(n-1) + fibonacci(n-2)
```

## 12.7 总结

本章介绍了 CoralNPU 的 ML 应用，包括：

1. **ML 应用概述**
   - 机器学习基础概念
   - CoralNPU 的优势和适用场景

2. **支持的模型**
   - MobileNet、ResNet、YOLO、Llama2 等
   - 各种模型的特点和应用场景

3. **模型部署**
   - 模型转换流程
   - 模型优化技术

4. **量化**
   - 量化原理和方法
   - INT8 量化、QAT、PTQ

5. **TFLite Micro**
   - 嵌入式 ML 框架
   - 使用方法和优化技巧

6. **Llama2 示例**
   - Llama2.c 项目
   - 在 CoralNPU 上运行和优化

通过本章的学习，你应该能够：
- 理解机器学习的基本概念
- 选择合适的模型和部署方案
- 使用量化技术优化模型
- 在 CoralNPU 上部署和运行 ML 应用

## 12.8 参考资源

**官方文档**：
- TensorFlow Lite: https://www.tensorflow.org/lite
- TFLite Micro: https://www.tensorflow.org/lite/microcontrollers
- Llama2: https://ai.meta.com/llama/

**开源项目**：
- Llama2.c: https://github.com/karpathy/llama2.c
- TFLite Micro: https://github.com/tensorflow/tflite-micro

**论文**：
- MobileNets: https://arxiv.org/abs/1704.04861
- ResNet: https://arxiv.org/abs/1512.03385
- YOLO: https://arxiv.org/abs/1506.02640
- Llama2: https://arxiv.org/abs/2307.09288
- Quantization: https://arxiv.org/abs/1712.05877

**教程**：
- TensorFlow 量化指南
- PyTorch 量化教程
- RISC-V 向量扩展编程指南

