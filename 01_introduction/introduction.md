# 项目介绍

## 项目概述

CoralNPU 是由 Google Research 设计并开源的机器学习推理硬件加速器。它是一个神经处理单元（NPU），也被称为 AI 加速器或深度学习处理器。CoralNPU 基于 32 位 RISC-V 指令集架构（ISA），专门针对超低功耗的可穿戴设备（如耳戴式设备、增强现实眼镜和智能手表）中的机器学习推理任务进行了优化。

CoralNPU 是一个完全开源的 IP 核心，可以免费集成到片上系统（SoC）中。它的设计理念是从机器学习加速器的数据平面特性出发，构建了一个带有自定义 SIMD 指令的 RISC-V CPU，并在微架构层面做出了与 ML 加速器特性相匹配的决策。

### 设计出发点

CoralNPU 的设计从矩阵（domain/matrix）能力开始，然后添加向量（vector）和标量（scalar）能力，形成一个融合的设计。这种设计方法与传统的 CPU 设计不同——传统 CPU 通常从标量核心开始，然后添加向量扩展；而 CoralNPU 则是以 ML 计算的核心需求（矩阵运算）为起点，反向构建整个处理器架构。

## 核心特性概览

CoralNPU 提供了以下主要特性：

- **RISC-V ISA**：支持 RV32IMF_Zve32x 指令集（具体为 `rv32imf_zve32x_zicsr_zifencei_zbb`）
- **三级流水线**：标量核心采用顺序分发、乱序完成架构
- **多发射能力**：标量指令每周期最多 4 条，向量指令每周期最多 2 条
- **紧耦合内存**：8 KB ITCM + 32 KB DTCM，提供可预测的低延迟访问
- **向量处理**：64 个 256 位向量寄存器，支持 8/16/32 位数据类型
- **矩阵加速**：外积 MAC 引擎，每周期 256 次 MAC 操作
- **AXI4 总线接口**：支持外部内存访问和配置

详细的架构设计请参见第三章《架构设计》。

## 面向的应用场景

CoralNPU 专门针对以下应用场景进行了优化：

### 超低功耗可穿戴设备

- 耳戴式设备（hearables）：语音识别、降噪、语音增强
- AR 眼镜：物体识别、场景理解、手势识别
- 智能手表：健康监测、活动识别、语音助手

### 边缘 AI 推理

- 实时推理：低延迟要求
- 离线运行：无需云端连接
- 隐私保护：数据不离开设备

### 量化神经网络

- 8 位整数量化模型
- MobileNet、EfficientNet 等轻量级模型
- 卷积神经网络（CNN）

### 不适合的场景

- 训练任务（CoralNPU 专注于推理）
- 高精度浮点计算（虽然支持 F 扩展，但主要优化是针对量化整数）
- 通用计算任务（这不是一个通用 CPU）


## 术语表

本文档中会用到以下关键术语和缩写：

### 处理器架构相关

| 术语 | 英文全称 | 中文解释 |
|------|---------|---------|
| NPU | Neural Processing Unit | 神经处理单元，专门用于加速神经网络计算的硬件 |
| ISA | Instruction Set Architecture | 指令集架构，定义了处理器支持的指令集合 |
| RISC-V | - | 开源的精简指令集架构 |
| SIMD | Single Instruction Multiple Data | 单指令多数据，一条指令同时处理多个数据 |
| MAC | Multiply-Accumulate | 乘累加操作，神经网络计算的基本操作 |
| CSR | Control and Status Register | 控制和状态寄存器 |

### 内存相关

| 术语 | 英文全称 | 中文解释 |
|------|---------|---------|
| TCM | Tightly-Coupled Memory | 紧耦合内存，与处理器核心直接连接的低延迟内存 |
| ITCM | Instruction TCM | 指令紧耦合内存，存储指令 |
| DTCM | Data TCM | 数据紧耦合内存，存储数据 |
| SRAM | Static Random-Access Memory | 静态随机存取内存 |
| Cache | - | 缓存，用于加速内存访问的高速缓冲存储器 |
| L1 | Level 1 | 一级缓存，最靠近处理器核心的缓存 |

### 流水线相关

| 术语 | 英文全称 | 中文解释 |
|------|---------|---------|
| Pipeline | - | 流水线，将指令执行分为多个阶段以提高吞吐量 |
| Fetch | Instruction Fetch | 指令获取，从内存读取指令 |
| Decode | Instruction Decode | 指令解码，解析指令的操作和操作数 |
| Dispatch | Instruction Dispatch | 指令分发，将指令发送到执行单元 |
| Execute | Instruction Execute | 指令执行，执行指令的操作 |
| Writeback | Register Writeback | 寄存器写回，将结果写回寄存器 |
| In-order | - | 顺序执行，指令按程序顺序执行 |
| Out-of-order | - | 乱序执行，指令可以不按程序顺序完成 |

### 指令集相关

| 术语 | 英文全称 | 中文解释 |
|------|---------|---------|
| RV32I | RISC-V 32-bit Integer | RISC-V 32 位整数基础指令集 |
| M 扩展 | Multiply/Divide Extension | 乘除法扩展 |
| F 扩展 | Single-Precision Floating-Point | 单精度浮点扩展 |
| C 扩展 | Compressed Instructions | 压缩指令扩展（CoralNPU 中被回收用于 SIMD） |
| Zve32x | Vector Extension for Embedded | 嵌入式向量扩展（32 位元素） |
| Zicsr | CSR Instructions | CSR 指令扩展 |
| Zifencei | Instruction Fence | 指令屏障扩展 |
| Zbb | Basic Bit Manipulation | 基本位操作扩展 |

### 向量/矩阵相关

| 术语 | 英文全称 | 中文解释 |
|------|---------|---------|
| Vector | - | 向量，一维数组数据 |
| Matrix | - | 矩阵，二维数组数据 |
| Outer Product | - | 外积，两个向量相乘得到矩阵 |
| Accumulator | - | 累加器，用于累加计算结果的寄存器 |
| Stripmining | - | 条带挖掘，将大数组操作分解为适合硬件的小块 |
| VDOT | Vector Dot Product | 向量点积操作 |
| Regfile | Register File | 寄存器文件，存储寄存器的硬件结构 |

### 总线和接口相关

| 术语 | 英文全称 | 中文解释 |
|------|---------|---------|
| AXI | Advanced eXtensible Interface | 高级可扩展接口，ARM 定义的片上总线协议 |
| AXI4 | AXI version 4 | AXI 协议第 4 版 |
| Manager | Bus Manager | 总线管理者（主设备），发起总线事务 |
| Subordinate | Bus Subordinate | 总线从属（从设备），响应总线事务 |
| TileLink | - | 另一种片上总线协议 |

### 机器学习相关

| 术语 | 英文全称 | 中文解释 |
|------|---------|---------|
| Inference | - | 推理，使用训练好的模型进行预测 |
| Quantization | - | 量化，将浮点数转换为低精度整数以减少计算量 |
| CNN | Convolutional Neural Network | 卷积神经网络 |
| Convolution | - | 卷积操作，CNN 的核心操作 |
| Batch | - | 批次，同时处理的多个输入样本 |
| MobileNet | - | 一种轻量级神经网络架构 |
| EfficientNet | - | 另一种高效的神经网络架构 |

### 其他术语

| 术语 | 英文全称 | 中文解释 |
|------|---------|---------|
| SoC | System-on-Chip | 片上系统，将整个系统集成在一个芯片上 |
| IP | Intellectual Property | 知识产权，这里指可复用的硬件设计模块 |
| SMC | System Management Controller | 系统管理控制器 |
| FIFO | First-In-First-Out | 先进先出队列 |
| Scoreboard | - | 记分板，用于跟踪指令依赖关系的硬件结构 |
| Interlock | - | 互锁，防止数据冒险的机制 |

### 数据类型缩写

| 缩写 | 含义 |
|------|------|
| int8 | 8 位有符号整数 |
| int16 | 16 位有符号整数 |
| int32 | 32 位有符号整数 |
| uint8 | 8 位无符号整数 |
| float32 | 32 位单精度浮点数 |

## 重要说明

- **SIMD vs 向量**：在 CoralNPU 文档中，这两个术语通常可以互换使用。严格来说，CoralNPU 的向量指令是固定长度的 SIMD 指令，不是可变长度的向量指令（如 ARM SVE 或 RISC-V RVV）。

- **C 扩展的回收**：CoralNPU 回收了 RISC-V C 扩展的编码空间用于自定义 SIMD 指令。这意味着 CoralNPU 不支持标准的 RISC-V 压缩指令，但获得了更大的编码空间来支持 64 个向量寄存器（需要 6 位索引）。

- **运行到完成模型**：CoralNPU 采用的执行模型不支持操作系统和中断。这简化了硬件设计，但也意味着它主要用于专用的 ML 推理任务，而不是通用计算。

- **量化优先**：虽然 CoralNPU 支持浮点运算（F 扩展），但其硬件优化主要针对 8 位整数量化计算。如果你的应用需要高精度浮点计算，CoralNPU 可能不是最佳选择。

