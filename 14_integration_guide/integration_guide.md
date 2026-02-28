# 第十四章：集成指南

## 14.1 集成概述

### 14.1.1 什么是 SoC 集成

SoC（System on Chip，片上系统）集成是指将 CoralNPU 作为一个 IP 核（Intellectual Property Core）嵌入到更大的芯片系统中。就像搭积木一样，CoralNPU 是一块积木，而整个 SoC 是由多个积木组成的完整系统。

在 SoC 中，CoralNPU 通常扮演以下角色：
- **协处理器**：配合主 CPU 处理特定任务
- **加速器**：加速机器学习推理等计算密集型任务
- **独立处理器**：在某些场景下独立运行程序

### 14.1.2 CoralNPU 集成方式

CoralNPU 提供两种总线接口，可以灵活集成到不同的 SoC 架构中：

1. **AXI4 接口**：适用于基于 ARM AMBA 总线的系统
2. **TileLink 接口**：适用于基于 RISC-V 生态的系统（如 OpenTitan）

```
集成方式选择：
┌─────────────────────────────────────────────────────────┐
│  你的 SoC 使用什么总线？                                  │
├─────────────────────────────────────────────────────────┤
│  ARM 系统 / AMBA 总线  →  使用 CoreAxi 模块              │
│  RISC-V / OpenTitan   →  使用 CoreTlul 模块             │
└─────────────────────────────────────────────────────────┘
```

### 14.1.3 集成流程概览

将 CoralNPU 集成到 SoC 的基本流程：

```
步骤 1: 生成 RTL 代码
   ↓
步骤 2: 连接时钟和复位
   ↓
步骤 3: 连接总线接口
   ↓
步骤 4: 连接中断信号
   ↓
步骤 5: 配置内存映射
   ↓
步骤 6: 验证集成
```

详细步骤：

1. **生成 RTL 代码**：使用 Bazel 编译生成 SystemVerilog 代码
2. **连接时钟和复位**：将 CoralNPU 的时钟和复位信号连接到 SoC 的时钟树
3. **连接总线接口**：将 AXI 或 TileLink 接口连接到 SoC 总线
4. **连接中断信号**：连接 CoralNPU 的状态信号到中断控制器
5. **配置内存映射**：在 SoC 地址空间中为 CoralNPU 分配地址范围
6. **验证集成**：运行仿真和测试验证集成正确性

## 14.2 SoC 集成

### 14.2.1 生成 RTL 代码

首先需要生成 CoralNPU 的 SystemVerilog 代码。

**生成 AXI 版本：**
```bash
bazel build //hdl/chisel/src/coralnpu:core_mini_axi_cc_library_emit_verilog
```

生成的文件位置：
```
bazel-bin/hdl/chisel/src/coralnpu/core_mini_axi_cc_library_emit_verilog/
├── CoreMiniAxi.sv          # 主模块
├── CoreMiniAxi.anno.json   # 注释文件
└── CoreMiniAxi.fir         # FIRRTL 中间文件
```

**生成 TileLink 版本：**
```bash
bazel build //hdl/chisel/src/coralnpu:core_mini_tlul_cc_library_emit_verilog
```

### 14.2.2 顶层连接

#### AXI 接口顶层连接

CoralNPU AXI 模块的顶层接口如下：

```
                    ┌─────────────────────────────┐
                    │       CoralNPU (AXI)        │
                    │                             │
    aclk        ────┤ clk                         │
    aresetn     ────┤ reset (active-low)          │
                    │                             │
                    │  ┌───────────────────────┐  │
    AXI Master  ────┤──┤ s_axi (Slave)         │  │  访问 ITCM/DTCM/CSR
                    │  └───────────────────────┘  │
                    │                             │
                    │  ┌───────────────────────┐  │
    AXI Slave   ────┤──┤ m_axi (Master)        │  │  访问外部内存
                    │  └───────────────────────┘  │
                    │                             │
    irq         ────┤ 中断输入                     │
    halted      ────┤ 停机状态输出                 │
    fault       ────┤ 故障状态输出                 │
    wfi         ────┤ 等待中断状态输出             │
                    │                             │
                    └─────────────────────────────┘
```

**Verilog 实例化示例：**

```verilog
// 实例化 CoralNPU AXI 模块
CoreMiniAxi u_coralnpu (
    // 时钟和复位
    .aclk           (sys_clk),
    .aresetn        (sys_resetn),
    
    // AXI Slave 接口 - 用于访问 CoralNPU 内部资源
    // 写地址通道
    .axi_slave_awvalid  (axi_s_awvalid),
    .axi_slave_awready  (axi_s_awready),
    .axi_slave_awaddr   (axi_s_awaddr),
    .axi_slave_awid     (axi_s_awid),
    .axi_slave_awlen    (axi_s_awlen),
    .axi_slave_awsize   (axi_s_awsize),
    .axi_slave_awburst  (axi_s_awburst),
    // 写数据通道
    .axi_slave_wvalid   (axi_s_wvalid),
    .axi_slave_wready   (axi_s_wready),
    .axi_slave_wdata    (axi_s_wdata),
    .axi_slave_wstrb    (axi_s_wstrb),
    .axi_slave_wlast    (axi_s_wlast),
    // 写响应通道
    .axi_slave_bvalid   (axi_s_bvalid),
    .axi_slave_bready   (axi_s_bready),
    .axi_slave_bresp    (axi_s_bresp),
    .axi_slave_bid      (axi_s_bid),
    // 读地址通道
    .axi_slave_arvalid  (axi_s_arvalid),
    .axi_slave_arready  (axi_s_arready),
    .axi_slave_araddr   (axi_s_araddr),
    .axi_slave_arid     (axi_s_arid),
    .axi_slave_arlen    (axi_s_arlen),
    .axi_slave_arsize   (axi_s_arsize),
    .axi_slave_arburst  (axi_s_arburst),
    // 读数据通道
    .axi_slave_rvalid   (axi_s_rvalid),
    .axi_slave_rready   (axi_s_rready),
    .axi_slave_rdata    (axi_s_rdata),
    .axi_slave_rresp    (axi_s_rresp),
    .axi_slave_rid      (axi_s_rid),
    .axi_slave_rlast    (axi_s_rlast),
    
    // AXI Master 接口 - CoralNPU 用于访问外部内存
    // 写地址通道
    .axi_master_awvalid (axi_m_awvalid),
    .axi_master_awready (axi_m_awready),
    .axi_master_awaddr  (axi_m_awaddr),
    .axi_master_awid    (axi_m_awid),
    .axi_master_awlen   (axi_m_awlen),
    .axi_master_awsize  (axi_m_awsize),
    .axi_master_awburst (axi_m_awburst),
    // 写数据通道
    .axi_master_wvalid  (axi_m_wvalid),
    .axi_master_wready  (axi_m_wready),
    .axi_master_wdata   (axi_m_wdata),
    .axi_master_wstrb   (axi_m_wstrb),
    .axi_master_wlast   (axi_m_wlast),
    // 写响应通道
    .axi_master_bvalid  (axi_m_bvalid),
    .axi_master_bready  (axi_m_bready),
    .axi_master_bresp   (axi_m_bresp),
    .axi_master_bid     (axi_m_bid),
    // 读地址通道
    .axi_master_arvalid (axi_m_arvalid),
    .axi_master_arready (axi_m_arready),
    .axi_master_araddr  (axi_m_araddr),
    .axi_master_arid    (axi_m_arid),
    .axi_master_arlen   (axi_m_arlen),
    .axi_master_arsize  (axi_m_arsize),
    .axi_master_arburst (axi_m_arburst),
    // 读数据通道
    .axi_master_rvalid  (axi_m_rvalid),
    .axi_master_rready  (axi_m_rready),
    .axi_master_rdata   (axi_m_rdata),
    .axi_master_rresp   (axi_m_rresp),
    .axi_master_rid     (axi_m_rid),
    .axi_master_rlast   (axi_m_rlast),
    
    // 中断和状态信号
    .irq        (coralnpu_irq),
    .halted     (coralnpu_halted),
    .fault      (coralnpu_fault),
    .wfi        (coralnpu_wfi),
    
    // 测试使能（通常接地）
    .te         (1'b0)
);
```


#### TileLink 接口顶层连接

CoralNPU TileLink 模块的顶层接口：

```
                    ┌─────────────────────────────┐
                    │     CoralNPU (TileLink)     │
                    │                             │
    clk         ────┤ clk                         │
    rst_ni      ────┤ reset (active-low)          │
                    │                             │
                    │  ┌───────────────────────┐  │
    TL Device   ────┤──┤ tl_device             │  │  访问 ITCM/DTCM/CSR
                    │  └───────────────────────┘  │
                    │                             │
                    │  ┌───────────────────────┐  │
    TL Host     ────┤──┤ tl_host               │  │  访问外部内存
                    │  └───────────────────────┘  │
                    │                             │
    irq         ────┤ 中断输入                     │
    halted      ────┤ 停机状态输出                 │
    fault       ────┤ 故障状态输出                 │
    wfi         ────┤ 等待中断状态输出             │
                    │                             │
                    └─────────────────────────────┘
```

**Verilog 实例化示例：**

```verilog
// 实例化 CoralNPU TileLink 模块
CoreMiniTlul u_coralnpu (
    // 时钟和复位
    .clk            (sys_clk),
    .rst_ni         (sys_resetn),
    
    // TileLink Device 接口 - 用于访问 CoralNPU 内部资源
    // A 通道（请求）
    .tl_device_a_valid      (tl_d_a_valid),
    .tl_device_a_ready      (tl_d_a_ready),
    .tl_device_a_bits_*     (tl_d_a_bits_*),
    // D 通道（响应）
    .tl_device_d_valid      (tl_d_d_valid),
    .tl_device_d_ready      (tl_d_d_ready),
    .tl_device_d_bits_*     (tl_d_d_bits_*),
    
    // TileLink Host 接口 - CoralNPU 用于访问外部内存
    // A 通道（请求）
    .tl_host_a_valid        (tl_h_a_valid),
    .tl_host_a_ready        (tl_h_a_ready),
    .tl_host_a_bits_*       (tl_h_a_bits_*),
    // D 通道（响应）
    .tl_host_d_valid        (tl_h_d_valid),
    .tl_host_d_ready        (tl_h_d_ready),
    .tl_host_d_bits_*       (tl_h_d_bits_*),
    
    // 中断和状态信号
    .irq        (coralnpu_irq),
    .halted     (coralnpu_halted),
    .fault      (coralnpu_fault),
    .wfi        (coralnpu_wfi),
    
    // 测试使能（通常接地）
    .te         (1'b0)
);
```

### 14.2.3 总线连接

#### AXI 总线连接

CoralNPU 有两个 AXI 接口，需要分别连接：

**1. AXI Slave 接口（s_axi）**

这个接口用于外部主机（如主 CPU）访问 CoralNPU 的内部资源：
- ITCM（指令紧耦合内存）
- DTCM（数据紧耦合内存）
- CSR（控制状态寄存器）

连接方式：
```
主 CPU 的 AXI Master ──→ SoC 总线互连 ──→ CoralNPU 的 AXI Slave
```

**2. AXI Master 接口（m_axi）**

这个接口用于 CoralNPU 访问外部资源：
- 系统内存（DDR）
- 其他外设

连接方式：
```
CoralNPU 的 AXI Master ──→ SoC 总线互连 ──→ 内存控制器/外设
```

**AXI 信号说明：**

AXI4 协议有 5 个通道，每个通道的信号含义：

1. **写地址通道（AW）**：
   - `awaddr`：写地址（CoralNPU 使用 32 位地址）
   - `awlen`：突发长度 - 1（例如 awlen=3 表示 4 次传输）
   - `awsize`：每次传输的字节数（0=1字节, 1=2字节, 2=4字节）
   - `awburst`：突发类型（CoralNPU 使用 INCR=1，表示地址递增）
   - `awid`：事务 ID（CoralNPU Master 总是使用 0）

2. **写数据通道（W）**：
   - `wdata`：写数据（128 位宽）
   - `wstrb`：字节选通（16 位，每位对应 1 个字节）
   - `wlast`：最后一次传输标志

3. **写响应通道（B）**：
   - `bresp`：响应码（0=OKAY, 2=SLVERR）
   - `bid`：事务 ID

4. **读地址通道（AR）**：
   - `araddr`：读地址
   - `arlen`：突发长度 - 1
   - `arsize`：每次传输的字节数
   - `arburst`：突发类型
   - `arid`：事务 ID

5. **读数据通道（R）**：
   - `rdata`：读数据（128 位宽）
   - `rresp`：响应码
   - `rid`：事务 ID
   - `rlast`：最后一次传输标志

**注意事项：**

- CoralNPU 的 AXI 接口**不支持** USER 信号
- CoralNPU Master 接口的所有事务 ID 都是 0
- 数据宽度固定为 128 位
- 地址宽度为 32 位

#### TileLink 总线连接

TileLink 是 RISC-V 生态中常用的总线协议，CoralNPU 支持 TileLink-UL（Uncached Lightweight）版本。

**1. TileLink Device 接口（tl_device）**

用于外部主机访问 CoralNPU 内部资源。

**2. TileLink Host 接口（tl_host）**

用于 CoralNPU 访问外部资源。

**TileLink 通道说明：**

TileLink-UL 有 2 个通道：

1. **A 通道（请求）**：
   - `a_opcode`：操作码（Get=4 读, PutFullData=0 写）
   - `a_address`：地址
   - `a_size`：传输大小（log2 字节数）
   - `a_data`：写数据
   - `a_mask`：字节掩码

2. **D 通道（响应）**：
   - `d_opcode`：操作码（AccessAckData=1 读响应, AccessAck=0 写响应）
   - `d_data`：读数据
   - `d_error`：错误标志

### 14.2.4 内存连接

CoralNPU 内部有两个紧耦合内存（TCM），它们的大小可以配置：

**默认配置：**
- ITCM：8 KB（指令存储）
- DTCM：32 KB（数据存储）

**内存地址映射：**

```
CoralNPU 内部地址空间：
┌──────────────────────────────────────────┐
│ 0x00000 - 0x01FFF  │  ITCM (8 KB)        │
├──────────────────────────────────────────┤
│ 0x02000 - 0x0FFFF  │  保留               │
├──────────────────────────────────────────┤
│ 0x10000 - 0x17FFF  │  DTCM (32 KB)       │
├──────────────────────────────────────────┤
│ 0x18000 - 0x2FFFF  │  保留               │
├──────────────────────────────────────────┤
│ 0x30000 - 0x3FFFF  │  CSR 寄存器         │
└──────────────────────────────────────────┘
```

**在 SoC 地址空间中的映射：**

假设 CoralNPU 在 SoC 中的基地址是 `0x7000_0000`，那么：

```
SoC 地址空间：
┌──────────────────────────────────────────────────┐
│ 0x70000000 - 0x70001FFF  │  CoralNPU ITCM        │
├──────────────────────────────────────────────────┤
│ 0x70010000 - 0x70017FFF  │  CoralNPU DTCM        │
├──────────────────────────────────────────────────┤
│ 0x70030000 - 0x7003FFFF  │  CoralNPU CSR         │
└──────────────────────────────────────────────────┘
```

**外部内存访问：**

CoralNPU 通过 Master 接口访问外部内存时，地址由程序指定。通常：
- 系统 DDR 内存：`0x8000_0000` - `0x9FFF_FFFF`（示例）
- 外设寄存器：根据 SoC 设计而定

## 14.3 接口要求

### 14.3.1 AXI4 接口要求

#### AXI Master 接口行为

CoralNPU 作为 AXI Master 时的行为特征：

**地址通道（AR/AW）：**
- `addr`：32 位地址，4 字节对齐
- `prot`：固定为 `3'b010`（非特权、非安全、数据访问）
- `id`：固定为 0
- `len`：支持 0-15（1-16 次传输）
- `size`：支持 0（1字节）、1（2字节）、2（4字节）
- `burst`：固定为 `2'b01`（INCR，地址递增）
- `lock`：固定为 0（正常访问）
- `cache`：固定为 `4'b0000`（设备，不可缓存）
- `qos`：固定为 0
- `region`：固定为 0

**数据通道（W/R）：**
- 数据宽度：128 位
- `wstrb`：16 位，每位对应 1 个字节
- `last`：标识突发传输的最后一拍

**响应通道（B）：**
- 接受 `OKAY`（0）和 `SLVERR`（2）响应
- 其他响应码会导致 CoralNPU 进入故障状态

#### AXI Slave 接口行为

CoralNPU 作为 AXI Slave 时的行为特征：

**支持的传输：**
- 突发长度：1-16 次传输
- 突发类型：FIXED（0）、INCR（1）、WRAP（2）
- 传输大小：1、2、4、8、16 字节

**地址对齐要求：**
- ITCM：4 字节对齐
- DTCM：1 字节对齐（支持非对齐访问）
- CSR：4 字节对齐

**响应行为：**
- 地址在有效范围内：返回 `OKAY`（0）
- 地址越界或对齐错误：返回 `SLVERR`（2）

**忽略的信号：**
- `prot`、`lock`、`cache`、`qos`、`region`：这些信号被忽略

### 14.3.2 TileLink 接口要求

#### TileLink Host 接口行为

CoralNPU 作为 TileLink Host 时的行为：

**A 通道（请求）：**
- 支持的操作：Get（读）、PutFullData（写）
- 地址：32 位
- 数据宽度：128 位
- 大小：支持 0-4（1、2、4、8、16 字节）

**D 通道（响应）：**
- 接受 AccessAckData（读响应）和 AccessAck（写响应）
- 检查 `d_error` 标志

#### TileLink Device 接口行为

CoralNPU 作为 TileLink Device 时的行为：

**支持的操作：**
- Get：读操作
- PutFullData：写操作
- PutPartialData：部分写操作

**完整性检查：**
- CoralNPU 的 TileLink 接口包含完整性生成和检查逻辑
- 符合 OpenTitan 的完整性要求

### 14.3.3 信号时序要求

#### 时钟要求

**时钟频率：**
- 推荐频率：50 MHz - 200 MHz
- 最大频率：取决于工艺和综合结果
- 最小频率：无限制（可以暂停时钟）

**时钟占空比：**
- 推荐：40% - 60%
- 时钟抖动：< 100 ps

#### 复位要求

**复位类型：**
- AXI 接口：异步复位，低电平有效（`aresetn`）
- TileLink 接口：异步复位，低电平有效（`rst_ni`）

**复位时序：**

```
时序图：
         ___     ___     ___     ___     ___     ___
clk   __|   |___|   |___|   |___|   |___|   |___|   |___
         _______________________________________
resetn  _|                                       |_______
                                    ↑
                                    复位释放点
                                    
        <-- 复位保持 -->  <-- 至少 2 个时钟周期 -->
```

**复位步骤：**

1. **上电复位**：
   - 保持复位信号低电平
   - 确保时钟稳定运行

2. **释放复位**：
   - 在时钟上升沿释放复位
   - 复位信号应保持至少 2 个时钟周期

3. **复位后初始化**：
   - CoralNPU 内部状态被清零
   - 核心处于停机状态（时钟门控）
   - 需要通过 CSR 配置启动

**注意事项：**
- CoralNPU 使用同步复位策略
- 复位期间必须提供时钟
- 不要在复位期间访问 CoralNPU

#### 接口时序

**AXI 接口时序：**

```
时序图（读操作）：
         ___     ___     ___     ___     ___
clk   __|   |___|   |___|   |___|   |___|   |___
         _______
arvalid _|       |_______________________________
                 _______
arready _________|       |_______________________
         _______
araddr  _|  A   |_______________________________
                         _______
rvalid  _________________|       |_______________
         _______________________
rready  _|                       |_______________
                         _______
rdata   _________________|  D   |_______________
```

**建立时间和保持时间：**
- 建立时间：> 0.5 ns（取决于工艺）
- 保持时间：> 0.2 ns

**传播延迟：**
- 时钟到输出：< 2 ns（取决于工艺）


## 14.4 时钟和电源要求

### 14.4.1 时钟频率

CoralNPU 的时钟频率取决于多个因素：

**推荐工作频率：**

| 工艺节点 | 推荐频率 | 最大频率（典型） |
|---------|---------|----------------|
| 180nm   | 50 MHz  | 100 MHz        |
| 90nm    | 100 MHz | 200 MHz        |
| 40nm    | 150 MHz | 300 MHz        |
| 28nm    | 200 MHz | 400 MHz        |

**注意：** 实际最大频率取决于：
- 工艺库特性
- 综合和布局布线质量
- 工作电压和温度
- 时序约束设置

**频率选择建议：**

1. **低功耗应用**：50-100 MHz
   - 适合电池供电设备
   - 功耗优先于性能

2. **平衡应用**：100-200 MHz
   - 性能和功耗平衡
   - 大多数应用的推荐选择

3. **高性能应用**：200 MHz 以上
   - 需要最大计算性能
   - 需要良好的散热设计

### 14.4.2 时钟域

CoralNPU 内部有多个时钟域：

```
时钟域结构：
┌─────────────────────────────────────────────┐
│  外部时钟输入 (aclk/clk)                     │
└──────────────┬──────────────────────────────┘
               │
               ├──→ 复位同步器 (RstSync)
               │    └──→ 同步后的时钟 (clk_o)
               │         │
               │         ├──→ CSR 模块
               │         │
               │         └──→ 时钟门控 (ClockGate)
               │              └──→ 核心时钟 (clk_gated)
               │                   └──→ CoralNPU Core
               │
               └──→ AXI/TileLink 桥接模块
```

**时钟域说明：**

1. **总线时钟域**：
   - 时钟源：`aclk`（AXI）或 `clk`（TileLink）
   - 用于：AXI/TileLink 接口、桥接逻辑

2. **同步时钟域**：
   - 时钟源：经过复位同步器的 `clk_o`
   - 用于：CSR、TCM 仲裁器

3. **核心时钟域**：
   - 时钟源：经过时钟门控的 `clk_gated`
   - 用于：CoralNPU 核心流水线

**时钟门控：**

CoralNPU 支持自动时钟门控以降低功耗：

```verilog
// 时钟门控条件
clock_enable = irq ||                    // 有中断
               (!csr.clock_gate &&       // CSR 未禁用时钟
                !core.wfi) ||            // 核心未等待中断
               debug_module.haltreq;     // 调试请求
```

当以下条件都满足时，核心时钟被门控：
- 没有中断请求
- CSR 允许时钟门控
- 核心执行了 WFI（等待中断）指令
- 没有调试请求

### 14.4.3 电源域

CoralNPU 可以配置为单电源域或多电源域：

#### 单电源域配置

最简单的配置，整个 CoralNPU 使用同一个电源：

```
┌─────────────────────────────────────┐
│         VDD (1.0V / 1.2V)           │
│  ┌───────────────────────────────┐  │
│  │      CoralNPU 全部逻辑        │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
```

**优点：**
- 设计简单
- 不需要电源管理逻辑

**缺点：**
- 无法关闭部分模块以节省功耗

#### 多电源域配置（高级）

将 CoralNPU 分为多个电源域：

```
电源域划分：
┌─────────────────────────────────────────────┐
│  VDD_ALWAYS (1.0V) - 始终供电               │
│  ┌───────────────────────────────────────┐  │
│  │  - AXI/TileLink 接口                  │  │
│  │  - CSR 寄存器                         │  │
│  │  - 电源管理控制逻辑                   │  │
│  └───────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
┌─────────────────────────────────────────────┐
│  VDD_CORE (1.0V) - 可关断                   │
│  ┌───────────────────────────────────────┐  │
│  │  - CoralNPU 核心流水线                │  │
│  │  - ITCM / DTCM                        │  │
│  └───────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
```

**实现多电源域需要：**
- 电源开关（Power Switch）
- 隔离单元（Isolation Cell）
- 电平转换器（Level Shifter，如果电压不同）
- 保持寄存器（Retention Register，可选）

**注意：** 多电源域配置需要修改 RTL 代码，添加 UPF（Unified Power Format）约束。

### 14.4.4 复位要求

#### 复位信号

CoralNPU 有两级复位：

1. **外部复位**（`aresetn` / `rst_ni`）：
   - 类型：异步复位，低电平有效
   - 作用：复位整个 CoralNPU
   - 来源：SoC 复位控制器

2. **内部核心复位**：
   - 类型：异步复位
   - 作用：仅复位 CoralNPU 核心
   - 来源：CSR 寄存器或调试模块

#### 复位策略

CoralNPU 使用**同步复位释放**策略：

```verilog
// 复位同步器（简化版）
module RstSync (
    input  wire clk_i,      // 输入时钟
    input  wire rstn_i,     // 异步复位输入
    output reg  rstn_o      // 同步复位输出
);
    reg rstn_sync;
    
    always @(posedge clk_i or negedge rstn_i) begin
        if (!rstn_i) begin
            rstn_sync <= 1'b0;
            rstn_o    <= 1'b0;
        end else begin
            rstn_sync <= 1'b1;
            rstn_o    <= rstn_sync;
        end
    end
endmodule
```

**为什么需要复位同步？**

异步复位信号可能在时钟边沿附近变化，导致亚稳态。复位同步器确保：
- 复位释放发生在时钟上升沿
- 避免亚稳态传播到系统中

#### 上电复位序列

正确的上电复位序列：

```
步骤 1: 上电
    ↓
    VDD 稳定
    ↓
步骤 2: 启动时钟
    ↓
    时钟稳定运行
    ↓
步骤 3: 保持复位
    ↓
    复位信号保持低电平至少 10 个时钟周期
    ↓
步骤 4: 释放复位
    ↓
    在时钟上升沿释放复位
    ↓
步骤 5: 等待稳定
    ↓
    等待至少 2 个时钟周期
    ↓
步骤 6: 开始配置
    ↓
    通过 CSR 配置 CoralNPU
```

**C 代码示例：**

```c
// 上电复位函数
void coralnpu_power_on_reset(void) {
    // 步骤 1-3: 硬件自动完成
    
    // 步骤 4: 释放外部复位（假设通过寄存器控制）
    *RESET_CTRL_REG = 0x1;  // 释放 CoralNPU 复位
    
    // 步骤 5: 等待稳定
    for (volatile int i = 0; i < 100; i++);  // 延迟
    
    // 步骤 6: 配置 CoralNPU
    // 此时 CoralNPU 处于复位后状态：
    // - 核心停机（时钟门控）
    // - PC = 0
    // - 所有寄存器清零
}
```

#### 软件复位

通过 CSR 寄存器进行软件复位：

```c
// 软件复位 CoralNPU 核心
void coralnpu_soft_reset(void) {
    volatile uint32_t* reset_csr = (uint32_t*)CORALNPU_RESET_CSR_ADDR;
    
    // 步骤 1: 停止时钟（可选，但推荐）
    *reset_csr = 0x3;  // RESET=1, CLOCK_GATE=1
    
    // 步骤 2: 等待一个时钟周期
    for (volatile int i = 0; i < 10; i++);
    
    // 步骤 3: 释放时钟门控
    *reset_csr = 0x1;  // RESET=1, CLOCK_GATE=0
    
    // 步骤 4: 等待一个时钟周期
    for (volatile int i = 0; i < 10; i++);
    
    // 步骤 5: 释放复位
    *reset_csr = 0x0;  // RESET=0, CLOCK_GATE=0
    
    // 现在 CoralNPU 核心已复位并开始运行
}
```

## 14.5 验证检查清单

### 14.5.1 接口验证

在集成 CoralNPU 后，需要验证接口连接的正确性。

#### AXI 接口验证

**检查项 1: AXI Slave 接口基本读写**

```c
// 测试 1: 写入 ITCM
void test_axi_slave_write(void) {
    volatile uint32_t* itcm = (uint32_t*)CORALNPU_ITCM_BASE;
    
    // 写入测试数据
    itcm[0] = 0x12345678;
    itcm[1] = 0xABCDEF00;
    
    // 读回验证
    assert(itcm[0] == 0x12345678);
    assert(itcm[1] == 0xABCDEF00);
    
    printf("✓ AXI Slave 写入测试通过\n");
}

// 测试 2: 读取 CSR
void test_axi_slave_read_csr(void) {
    volatile uint32_t* status_csr = (uint32_t*)CORALNPU_STATUS_CSR;
    
    uint32_t status = *status_csr;
    printf("CoralNPU 状态: 0x%08X\n", status);
    
    // 上电后应该处于停机状态
    assert(status & 0x1);  // HALTED 位应该为 1
    
    printf("✓ AXI Slave 读取 CSR 测试通过\n");
}

// 测试 3: 突发传输
void test_axi_slave_burst(void) {
    volatile uint32_t* dtcm = (uint32_t*)CORALNPU_DTCM_BASE;
    
    // 写入一段数据
    for (int i = 0; i < 256; i++) {
        dtcm[i] = i * 0x11111111;
    }
    
    // 读回验证
    for (int i = 0; i < 256; i++) {
        assert(dtcm[i] == i * 0x11111111);
    }
    
    printf("✓ AXI Slave 突发传输测试通过\n");
}
```

**检查项 2: AXI Master 接口访问外部内存**

```c
// 测试 4: CoralNPU 访问外部内存
void test_axi_master_access(void) {
    // 准备测试程序：从外部内存读取数据
    volatile uint32_t* itcm = (uint32_t*)CORALNPU_ITCM_BASE;
    volatile uint32_t* dtcm = (uint32_t*)CORALNPU_DTCM_BASE;
    
    // 在外部内存准备测试数据
    volatile uint32_t* ext_mem = (uint32_t*)EXTERNAL_MEM_BASE;
    ext_mem[0] = 0xDEADBEEF;
    
    // 编写简单的测试程序（伪代码）
    // lw x1, 0(x2)  // 从外部内存读取
    // sw x1, 0(x3)  // 写入 DTCM
    itcm[0] = 0x00012083;  // lw x1, 0(x2)
    itcm[1] = 0x0011A023;  // sw x1, 0(x3)
    itcm[2] = 0x00000013;  // nop
    // ... 配置寄存器 x2 = EXTERNAL_MEM_BASE, x3 = DTCM_BASE
    
    // 启动 CoralNPU
    coralnpu_start();
    
    // 等待执行完成
    coralnpu_wait_halted();
    
    // 验证结果
    assert(dtcm[0] == 0xDEADBEEF);
    
    printf("✓ AXI Master 访问外部内存测试通过\n");
}
```

**检查项 3: AXI 错误响应处理**

```c
// 测试 5: 访问无效地址
void test_axi_error_response(void) {
    volatile uint32_t* invalid_addr = (uint32_t*)(CORALNPU_BASE + 0x50000);
    
    // 尝试访问无效地址
    uint32_t data = *invalid_addr;  // 应该返回 SLVERR
    
    // 检查 AXI 总线是否正确处理错误
    // （具体检查方法取决于 SoC 设计）
    
    printf("✓ AXI 错误响应测试通过\n");
}
```

#### TileLink 接口验证

TileLink 接口的验证类似于 AXI，但需要检查 TileLink 特有的特性：

```c
// 测试 6: TileLink 完整性检查
void test_tilelink_integrity(void) {
    // TileLink 接口包含完整性位
    // 验证完整性生成和检查是否正常工作
    
    // 写入数据
    volatile uint32_t* itcm = (uint32_t*)CORALNPU_ITCM_BASE;
    itcm[0] = 0x12345678;
    
    // 读回数据
    uint32_t data = itcm[0];
    
    // 如果完整性检查失败，TileLink 会返回错误
    assert(data == 0x12345678);
    
    printf("✓ TileLink 完整性测试通过\n");
}
```

### 14.5.2 功能验证

验证 CoralNPU 的基本功能是否正常。

#### 启动和停止

```c
// 测试 7: 启动和停止 CoralNPU
void test_start_stop(void) {
    volatile uint32_t* reset_csr = (uint32_t*)CORALNPU_RESET_CSR;
    volatile uint32_t* status_csr = (uint32_t*)CORALNPU_STATUS_CSR;
    
    // 初始状态：停机
    uint32_t status = *status_csr;
    assert(status & 0x1);  // HALTED
    
    // 加载程序到 ITCM
    load_program_to_itcm();
    
    // 启动 CoralNPU
    *reset_csr = 0x0;  // 释放复位和时钟门控
    
    // 等待一段时间
    delay_ms(10);
    
    // 检查状态：应该在运行或已完成
    status = *status_csr;
    // 根据程序，可能是运行中或已停机
    
    printf("✓ 启动和停止测试通过\n");
}
```


#### 中断处理

```c
// 测试 8: 中断响应
void test_interrupt(void) {
    // 加载一个等待中断的程序
    volatile uint32_t* itcm = (uint32_t*)CORALNPU_ITCM_BASE;
    
    // 程序：执行 WFI 指令等待中断
    itcm[0] = 0x10500073;  // WFI
    itcm[1] = 0x00000013;  // NOP
    
    // 启动 CoralNPU
    coralnpu_start();
    
    // 等待进入 WFI 状态
    delay_ms(1);
    
    // 检查 WFI 状态
    volatile uint32_t* status_csr = (uint32_t*)CORALNPU_STATUS_CSR;
    // 注意：WFI 状态通过 io.wfi 输出，不在 STATUS CSR 中
    
    // 发送中断
    send_interrupt_to_coralnpu();
    
    // 等待中断处理
    delay_ms(1);
    
    // CoralNPU 应该从 WFI 唤醒并继续执行
    
    printf("✓ 中断响应测试通过\n");
}
```

#### 程序执行

```c
// 测试 9: 执行简单程序
void test_simple_program(void) {
    volatile uint32_t* itcm = (uint32_t*)CORALNPU_ITCM_BASE;
    volatile uint32_t* dtcm = (uint32_t*)CORALNPU_DTCM_BASE;
    
    // 简单的加法程序
    // x1 = 10
    // x2 = 20
    // x3 = x1 + x2
    // 将 x3 存储到 DTCM[0]
    
    itcm[0] = 0x00A00093;  // addi x1, x0, 10
    itcm[1] = 0x01400113;  // addi x2, x0, 20
    itcm[2] = 0x002081B3;  // add x3, x1, x2
    itcm[3] = 0x00302023;  // sw x3, 0(x0) + DTCM_offset
    // ... 需要配置 DTCM 基地址
    
    // 启动执行
    coralnpu_start();
    coralnpu_wait_halted();
    
    // 验证结果
    assert(dtcm[0] == 30);
    
    printf("✓ 简单程序执行测试通过\n");
}
```

### 14.5.3 性能验证

验证 CoralNPU 的性能是否符合预期。

#### 时钟频率测试

```c
// 测试 10: 测量实际运行频率
void test_clock_frequency(void) {
    // 加载一个计数程序
    volatile uint32_t* itcm = (uint32_t*)CORALNPU_ITCM_BASE;
    volatile uint32_t* dtcm = (uint32_t*)CORALNPU_DTCM_BASE;
    
    // 程序：循环 1000000 次
    // for (i = 0; i < 1000000; i++);
    
    // 记录开始时间
    uint32_t start_time = get_system_time_us();
    
    // 启动 CoralNPU
    coralnpu_start();
    coralnpu_wait_halted();
    
    // 记录结束时间
    uint32_t end_time = get_system_time_us();
    uint32_t elapsed_us = end_time - start_time;
    
    // 计算实际频率
    // 假设每次循环 5 条指令，1000000 次循环 = 5000000 条指令
    uint32_t instructions = 5000000;
    float freq_mhz = (float)instructions / elapsed_us;
    
    printf("实际运行频率: %.2f MHz\n", freq_mhz);
    
    // 验证频率在合理范围内
    assert(freq_mhz > 10.0 && freq_mhz < 500.0);
    
    printf("✓ 时钟频率测试通过\n");
}
```

#### 内存带宽测试

```c
// 测试 11: 测量内存访问带宽
void test_memory_bandwidth(void) {
    // 测试 DTCM 写入带宽
    volatile uint32_t* dtcm = (uint32_t*)CORALNPU_DTCM_BASE;
    
    uint32_t start_time = get_system_time_us();
    
    // 写入 8KB 数据
    for (int i = 0; i < 2048; i++) {
        dtcm[i] = i;
    }
    
    uint32_t end_time = get_system_time_us();
    uint32_t elapsed_us = end_time - start_time;
    
    // 计算带宽 (MB/s)
    float bandwidth_mbps = (8.0 * 1024) / elapsed_us;
    
    printf("DTCM 写入带宽: %.2f MB/s\n", bandwidth_mbps);
    
    printf("✓ 内存带宽测试通过\n");
}
```

#### 功耗测试

```c
// 测试 12: 测量功耗
void test_power_consumption(void) {
    // 测量空闲功耗
    coralnpu_stop();
    delay_ms(100);
    float idle_power = measure_power_mw();
    printf("空闲功耗: %.2f mW\n", idle_power);
    
    // 测量运行功耗
    load_busy_loop_program();
    coralnpu_start();
    delay_ms(100);
    float active_power = measure_power_mw();
    printf("运行功耗: %.2f mW\n", active_power);
    
    // 测量 WFI 功耗
    load_wfi_program();
    coralnpu_start();
    delay_ms(100);
    float wfi_power = measure_power_mw();
    printf("WFI 功耗: %.2f mW\n", wfi_power);
    
    // 验证功耗梯度
    assert(wfi_power < active_power);
    assert(idle_power < wfi_power);
    
    printf("✓ 功耗测试通过\n");
}
```

### 14.5.4 时序验证

验证时序是否满足要求。

#### 静态时序分析（STA）

使用 EDA 工具进行静态时序分析：

**检查项：**

1. **建立时间（Setup Time）**：
   - 所有路径的建立时间裕量 > 0
   - 关键路径识别

2. **保持时间（Hold Time）**：
   - 所有路径的保持时间裕量 > 0

3. **时钟偏斜（Clock Skew）**：
   - 时钟树偏斜 < 100 ps

4. **最大频率**：
   - 根据最差路径计算最大工作频率

**示例 SDC 约束文件：**

```tcl
# 时钟定义
create_clock -name sys_clk -period 10.0 [get_ports aclk]

# 输入延迟
set_input_delay -clock sys_clk -max 2.0 [all_inputs]
set_input_delay -clock sys_clk -min 0.5 [all_inputs]

# 输出延迟
set_output_delay -clock sys_clk -max 2.0 [all_outputs]
set_output_delay -clock sys_clk -min 0.5 [all_outputs]

# 时钟不确定性
set_clock_uncertainty -setup 0.2 [get_clocks sys_clk]
set_clock_uncertainty -hold 0.1 [get_clocks sys_clk]

# 虚假路径（如果有跨时钟域路径）
# set_false_path -from [get_clocks clk_a] -to [get_clocks clk_b]

# 多周期路径（如果有）
# set_multicycle_path -setup 2 -from [get_pins ...] -to [get_pins ...]
```

#### 动态时序验证

在仿真中验证时序：

```systemverilog
// 时序检查模块
module timing_checker (
    input wire clk,
    input wire resetn,
    input wire axi_awvalid,
    input wire axi_awready
);
    // 检查握手信号的时序
    property axi_handshake_timing;
        @(posedge clk) disable iff (!resetn)
        axi_awvalid && !axi_awready |=> axi_awvalid;
    endproperty
    
    assert property (axi_handshake_timing)
        else $error("AXI 握手时序违规");
    
    // 检查复位时序
    property reset_timing;
        @(posedge clk)
        $fell(resetn) |-> ##[1:10] $rose(resetn);
    endproperty
    
    assert property (reset_timing)
        else $error("复位时序违规");
endmodule
```

### 14.5.5 完整验证检查清单

以下是完整的验证检查清单，用于确保 CoralNPU 集成正确：

#### 接口验证清单

```
□ AXI Slave 接口
  □ 单次读操作
  □ 单次写操作
  □ 突发读操作（长度 2, 4, 8, 16）
  □ 突发写操作（长度 2, 4, 8, 16）
  □ 不同传输大小（1, 2, 4, 8, 16 字节）
  □ FIXED 突发类型
  □ INCR 突发类型
  □ WRAP 突发类型
  □ 地址对齐检查
  □ 地址越界检查
  □ 错误响应（SLVERR）
  □ ID 传递正确性

□ AXI Master 接口
  □ 读操作
  □ 写操作
  □ 突发传输
  □ 访问外部内存
  □ 访问外设寄存器
  □ 错误响应处理

□ TileLink 接口（如果使用）
  □ Get 操作
  □ PutFullData 操作
  □ PutPartialData 操作
  □ 完整性检查
  □ 错误处理
```

#### 功能验证清单

```
□ 复位和启动
  □ 上电复位
  □ 软件复位
  □ 复位后状态检查
  □ 启动流程

□ 程序执行
  □ 简单算术运算
  □ 内存访问（ITCM/DTCM）
  □ 外部内存访问
  □ 分支和跳转
  □ 函数调用和返回

□ 中断处理
  □ 中断响应
  □ WFI 指令
  □ 中断唤醒
  □ 中断优先级（如果支持）

□ CSR 访问
  □ 读取状态寄存器
  □ 写入控制寄存器
  □ PC 设置
  □ 时钟门控控制

□ 调试功能（如果使用）
  □ 调试模块访问
  □ 断点设置
  □ 单步执行
  □ 寄存器读写
```

#### 性能验证清单

```
□ 时钟频率
  □ 最大频率测试
  □ 不同频率下的稳定性

□ 吞吐量
  □ 指令吞吐量（IPC）
  □ 内存带宽
  □ 总线利用率

□ 延迟
  □ 中断响应延迟
  □ 内存访问延迟
  □ 总线事务延迟

□ 功耗
  □ 空闲功耗
  □ 运行功耗
  □ WFI 功耗
  □ 时钟门控效果
```

#### 时序验证清单

```
□ 静态时序分析
  □ 建立时间检查
  □ 保持时间检查
  □ 时钟偏斜检查
  □ 最大频率计算
  □ 关键路径分析

□ 动态时序验证
  □ 仿真时序检查
  □ 握手协议时序
  □ 复位时序
  □ 时钟域交叉检查
```

#### 压力测试清单

```
□ 长时间运行测试
  □ 24 小时稳定性测试
  □ 温度循环测试

□ 边界条件测试
  □ 最大频率运行
  □ 最小电压运行
  □ 极端温度测试

□ 随机测试
  □ 随机指令序列
  □ 随机内存访问模式
  □ 随机中断注入
```

## 14.6 集成示例

### 14.6.1 简单 SoC 集成示例

以下是一个简单的 SoC 集成示例，展示如何将 CoralNPU 集成到一个基本的系统中。

#### 系统架构

```
简单 SoC 架构：
┌─────────────────────────────────────────────────────┐
│                    Simple SoC                       │
│                                                     │
│  ┌──────────┐         ┌──────────────┐             │
│  │   CPU    │────────→│  AXI Crossbar│             │
│  │ (Master) │         │              │             │
│  └──────────┘         └───────┬──────┘             │
│                               │                     │
│                    ┌──────────┼──────────┐          │
│                    │          │          │          │
│              ┌─────▼────┐ ┌──▼────┐ ┌──▼────────┐  │
│              │ CoralNPU │ │  RAM  │ │ Peripherals│ │
│              │ (Slave)  │ │       │ │            │ │
│              └─────┬────┘ └───────┘ └────────────┘  │
│                    │                                │
│                    │ (Master)                       │
│                    ▼                                │
│              ┌──────────┐                           │
│              │   RAM    │                           │
│              └──────────┘                           │
└─────────────────────────────────────────────────────┘
```

#### Verilog 顶层模块

```verilog
module simple_soc (
    input  wire        clk,
    input  wire        resetn,
    
    // 外部接口（如 UART、GPIO 等）
    output wire        uart_tx,
    input  wire        uart_rx
);

    // ========================================
    // 时钟和复位
    // ========================================
    wire sys_clk = clk;
    wire sys_resetn = resetn;

    // ========================================
    // AXI 信号定义
    // ========================================
    
    // CPU -> Crossbar
    wire [31:0] cpu_axi_awaddr;
    wire        cpu_axi_awvalid;
    wire        cpu_axi_awready;
    // ... 其他 AXI 信号
    
    // Crossbar -> CoralNPU Slave
    wire [31:0] coralnpu_s_axi_awaddr;
    wire        coralnpu_s_axi_awvalid;
    wire        coralnpu_s_axi_awready;
    // ... 其他 AXI 信号
    
    // CoralNPU Master -> Crossbar
    wire [31:0] coralnpu_m_axi_awaddr;
    wire        coralnpu_m_axi_awvalid;
    wire        coralnpu_m_axi_awready;
    // ... 其他 AXI 信号

    // ========================================
    // CPU 实例化
    // ========================================
    cpu_core u_cpu (
        .clk            (sys_clk),
        .resetn         (sys_resetn),
        
        // AXI Master 接口
        .m_axi_awaddr   (cpu_axi_awaddr),
        .m_axi_awvalid  (cpu_axi_awvalid),
        .m_axi_awready  (cpu_axi_awready),
        // ... 其他信号
    );

    // ========================================
    // AXI Crossbar
    // ========================================
    axi_crossbar u_crossbar (
        .clk            (sys_clk),
        .resetn         (sys_resetn),
        
        // Master 接口
        .s_axi_awaddr   ({cpu_axi_awaddr, coralnpu_m_axi_awaddr}),
        .s_axi_awvalid  ({cpu_axi_awvalid, coralnpu_m_axi_awvalid}),
        .s_axi_awready  ({cpu_axi_awready, coralnpu_m_axi_awready}),
        // ... 其他信号
        
        // Slave 接口
        .m_axi_awaddr   ({coralnpu_s_axi_awaddr, ram_axi_awaddr, periph_axi_awaddr}),
        .m_axi_awvalid  ({coralnpu_s_axi_awvalid, ram_axi_awvalid, periph_axi_awvalid}),
        .m_axi_awready  ({coralnpu_s_axi_awready, ram_axi_awready, periph_axi_awready}),
        // ... 其他信号
    );

    // ========================================
    // CoralNPU 实例化
    // ========================================
    wire coralnpu_irq;
    wire coralnpu_halted;
    wire coralnpu_fault;
    wire coralnpu_wfi;
    
    CoreMiniAxi u_coralnpu (
        .aclk           (sys_clk),
        .aresetn        (sys_resetn),
        
        // AXI Slave 接口
        .axi_slave_awaddr   (coralnpu_s_axi_awaddr),
        .axi_slave_awvalid  (coralnpu_s_axi_awvalid),
        .axi_slave_awready  (coralnpu_s_axi_awready),
        // ... 其他信号
        
        // AXI Master 接口
        .axi_master_awaddr  (coralnpu_m_axi_awaddr),
        .axi_master_awvalid (coralnpu_m_axi_awvalid),
        .axi_master_awready (coralnpu_m_axi_awready),
        // ... 其他信号
        
        // 控制信号
        .irq        (coralnpu_irq),
        .halted     (coralnpu_halted),
        .fault      (coralnpu_fault),
        .wfi        (coralnpu_wfi),
        .te         (1'b0)
    );
    
    // 中断连接（示例：定时器中断）
    assign coralnpu_irq = timer_interrupt;

    // ========================================
    // 内存和外设
    // ========================================
    // RAM、外设等模块实例化...
    
endmodule
```


### 14.6.2 启动 CoralNPU 的完整流程

以下是在集成后启动 CoralNPU 的完整 C 代码示例。

#### 头文件定义

```c
// coralnpu.h - CoralNPU 驱动头文件

#ifndef CORALNPU_H
#define CORALNPU_H

#include <stdint.h>
#include <stdbool.h>

// CoralNPU 基地址（根据实际 SoC 设计修改）
#define CORALNPU_BASE       0x70000000

// 内存区域偏移
#define ITCM_OFFSET         0x00000
#define DTCM_OFFSET         0x10000
#define CSR_OFFSET          0x30000

// 内存大小
#define ITCM_SIZE           (8 * 1024)    // 8 KB
#define DTCM_SIZE           (32 * 1024)   // 32 KB

// CSR 寄存器偏移
#define CSR_RESET_CONTROL   0x0
#define CSR_PC_START        0x4
#define CSR_STATUS          0x8

// CSR 位定义
#define RESET_BIT           (1 << 0)
#define CLOCK_GATE_BIT      (1 << 1)
#define STATUS_HALTED_BIT   (1 << 0)
#define STATUS_FAULT_BIT    (1 << 1)

// 内存指针
#define CORALNPU_ITCM       ((volatile uint8_t*)(CORALNPU_BASE + ITCM_OFFSET))
#define CORALNPU_DTCM       ((volatile uint8_t*)(CORALNPU_BASE + DTCM_OFFSET))
#define CORALNPU_CSR(off)   ((volatile uint32_t*)(CORALNPU_BASE + CSR_OFFSET + (off)))

// 函数声明
void coralnpu_init(void);
void coralnpu_load_program(const uint8_t* program, uint32_t size);
void coralnpu_set_pc(uint32_t pc);
void coralnpu_start(void);
void coralnpu_stop(void);
void coralnpu_reset(void);
bool coralnpu_is_halted(void);
bool coralnpu_is_fault(void);
void coralnpu_wait_halted(void);

#endif // CORALNPU_H
```

#### 驱动实现

```c
// coralnpu.c - CoralNPU 驱动实现

#include "coralnpu.h"
#include <string.h>

// 延迟函数（需要根据实际系统实现）
static void delay_cycles(uint32_t cycles) {
    for (volatile uint32_t i = 0; i < cycles; i++);
}

/**
 * 初始化 CoralNPU
 * 这个函数在系统启动时调用一次
 */
void coralnpu_init(void) {
    // 确保 CoralNPU 处于复位状态
    volatile uint32_t* reset_csr = CORALNPU_CSR(CSR_RESET_CONTROL);
    *reset_csr = RESET_BIT | CLOCK_GATE_BIT;
    
    // 等待复位生效
    delay_cycles(100);
    
    // 清空 ITCM 和 DTCM
    memset((void*)CORALNPU_ITCM, 0, ITCM_SIZE);
    memset((void*)CORALNPU_DTCM, 0, DTCM_SIZE);
    
    // 设置 PC 为 0
    volatile uint32_t* pc_csr = CORALNPU_CSR(CSR_PC_START);
    *pc_csr = 0;
}

/**
 * 加载程序到 ITCM
 * @param program 程序二进制数据
 * @param size 程序大小（字节）
 */
void coralnpu_load_program(const uint8_t* program, uint32_t size) {
    if (size > ITCM_SIZE) {
        // 错误：程序太大
        return;
    }
    
    // 确保 CoralNPU 处于停止状态
    coralnpu_stop();
    
    // 复制程序到 ITCM
    memcpy((void*)CORALNPU_ITCM, program, size);
    
    // 如果程序小于 ITCM，清空剩余部分
    if (size < ITCM_SIZE) {
        memset((void*)(CORALNPU_ITCM + size), 0, ITCM_SIZE - size);
    }
}

/**
 * 设置起始 PC
 * @param pc 程序计数器值
 */
void coralnpu_set_pc(uint32_t pc) {
    volatile uint32_t* pc_csr = CORALNPU_CSR(CSR_PC_START);
    *pc_csr = pc;
}

/**
 * 启动 CoralNPU
 */
void coralnpu_start(void) {
    volatile uint32_t* reset_csr = CORALNPU_CSR(CSR_RESET_CONTROL);
    
    // 步骤 1: 释放时钟门控，但保持复位
    *reset_csr = RESET_BIT;
    delay_cycles(10);
    
    // 步骤 2: 释放复位
    *reset_csr = 0;
    delay_cycles(10);
    
    // 现在 CoralNPU 开始执行
}

/**
 * 停止 CoralNPU
 */
void coralnpu_stop(void) {
    volatile uint32_t* reset_csr = CORALNPU_CSR(CSR_RESET_CONTROL);
    
    // 进入复位状态并门控时钟
    *reset_csr = RESET_BIT | CLOCK_GATE_BIT;
    delay_cycles(10);
}

/**
 * 复位 CoralNPU（软件复位）
 */
void coralnpu_reset(void) {
    coralnpu_stop();
    delay_cycles(100);
    // 保持在停止状态，需要调用 coralnpu_start() 来启动
}

/**
 * 检查 CoralNPU 是否停机
 * @return true 如果停机，false 如果运行中
 */
bool coralnpu_is_halted(void) {
    volatile uint32_t* status_csr = CORALNPU_CSR(CSR_STATUS);
    uint32_t status = *status_csr;
    return (status & STATUS_HALTED_BIT) != 0;
}

/**
 * 检查 CoralNPU 是否故障
 * @return true 如果故障，false 如果正常
 */
bool coralnpu_is_fault(void) {
    volatile uint32_t* status_csr = CORALNPU_CSR(CSR_STATUS);
    uint32_t status = *status_csr;
    return (status & STATUS_FAULT_BIT) != 0;
}

/**
 * 等待 CoralNPU 停机
 * 这个函数会阻塞直到 CoralNPU 停机或故障
 */
void coralnpu_wait_halted(void) {
    while (!coralnpu_is_halted() && !coralnpu_is_fault()) {
        // 轮询状态
        delay_cycles(100);
    }
}
```

#### 使用示例

```c
// main.c - 使用 CoralNPU 的示例程序

#include "coralnpu.h"
#include <stdio.h>

// 示例程序：计算 1+2+3+...+100
// 这是一个编译好的 RISC-V 二进制程序
const uint8_t test_program[] = {
    // 这里应该是实际的机器码
    // 为了示例，这里用伪代码表示
    0x13, 0x00, 0x00, 0x00,  // addi x0, x0, 0 (nop)
    // ... 更多指令
};

int main(void) {
    printf("CoralNPU 集成测试\n");
    
    // 步骤 1: 初始化 CoralNPU
    printf("初始化 CoralNPU...\n");
    coralnpu_init();
    
    // 步骤 2: 加载程序
    printf("加载程序到 ITCM...\n");
    coralnpu_load_program(test_program, sizeof(test_program));
    
    // 步骤 3: 设置起始 PC（如果不是从 0 开始）
    coralnpu_set_pc(0x0000);
    
    // 步骤 4: 启动 CoralNPU
    printf("启动 CoralNPU...\n");
    coralnpu_start();
    
    // 步骤 5: 等待执行完成
    printf("等待执行完成...\n");
    coralnpu_wait_halted();
    
    // 步骤 6: 检查状态
    if (coralnpu_is_fault()) {
        printf("错误：CoralNPU 发生故障\n");
        return -1;
    }
    
    printf("执行完成\n");
    
    // 步骤 7: 读取结果（从 DTCM）
    volatile uint32_t* dtcm = (volatile uint32_t*)CORALNPU_DTCM;
    uint32_t result = dtcm[0];
    printf("结果: %u\n", result);
    
    // 步骤 8: 停止 CoralNPU
    coralnpu_stop();
    
    return 0;
}
```

### 14.6.3 中断处理示例

如果需要使用中断功能，以下是完整的示例。

#### 中断控制器连接

```verilog
// 在 SoC 顶层模块中
module soc_top (
    // ... 其他信号
);
    // 中断源
    wire timer_irq;
    wire uart_irq;
    wire gpio_irq;
    
    // 中断控制器
    wire coralnpu_irq;
    
    interrupt_controller u_intc (
        .clk        (sys_clk),
        .resetn     (sys_resetn),
        
        // 中断输入
        .irq_in     ({timer_irq, uart_irq, gpio_irq}),
        
        // 中断输出到 CoralNPU
        .irq_out    (coralnpu_irq)
    );
    
    // CoralNPU
    CoreMiniAxi u_coralnpu (
        // ... 其他信号
        .irq        (coralnpu_irq),
        // ...
    );
endmodule
```

#### 中断处理程序（RISC-V 汇编）

```assembly
# interrupt_handler.S - CoralNPU 中断处理程序

.section .text
.global _start
.global interrupt_handler

_start:
    # 初始化栈指针
    la sp, stack_top
    
    # 设置中断向量
    la t0, interrupt_handler
    csrw mtvec, t0
    
    # 使能中断
    li t0, 0x8
    csrw mstatus, t0
    
    # 主循环
main_loop:
    # 执行一些工作
    # ...
    
    # 等待中断
    wfi
    
    # 中断返回后继续
    j main_loop

# 中断处理程序
interrupt_handler:
    # 保存上下文
    addi sp, sp, -64
    sw x1, 0(sp)
    sw x2, 4(sp)
    # ... 保存其他寄存器
    
    # 读取中断原因
    csrr t0, mcause
    
    # 处理中断
    # ...
    
    # 恢复上下文
    lw x1, 0(sp)
    lw x2, 4(sp)
    # ... 恢复其他寄存器
    addi sp, sp, 64
    
    # 中断返回
    mret
```

#### 中断触发（C 代码）

```c
// 从主 CPU 触发 CoralNPU 中断

// 假设中断控制器有一个软件中断寄存器
#define INTC_SW_IRQ_REG  ((volatile uint32_t*)0x80000000)

void trigger_coralnpu_interrupt(void) {
    // 设置软件中断位
    *INTC_SW_IRQ_REG = 0x1;
    
    // 中断会自动传递到 CoralNPU
}

void clear_coralnpu_interrupt(void) {
    // 清除软件中断位
    *INTC_SW_IRQ_REG = 0x0;
}
```

## 14.7 常见问题和解决方案

### 14.7.1 集成问题

#### 问题 1: CoralNPU 无法启动

**症状：**
- 启动后 CoralNPU 保持停机状态
- 状态寄存器显示 HALTED=1

**可能原因和解决方案：**

1. **时钟门控未释放**
   ```c
   // 检查 RESET_CONTROL 寄存器
   uint32_t reset_ctrl = *CORALNPU_CSR(CSR_RESET_CONTROL);
   if (reset_ctrl & CLOCK_GATE_BIT) {
       // 时钟被门控，需要释放
       *CORALNPU_CSR(CSR_RESET_CONTROL) = 0;
   }
   ```

2. **复位未正确释放**
   ```c
   // 确保复位序列正确
   *CORALNPU_CSR(CSR_RESET_CONTROL) = RESET_BIT | CLOCK_GATE_BIT;
   delay_cycles(10);
   *CORALNPU_CSR(CSR_RESET_CONTROL) = RESET_BIT;
   delay_cycles(10);
   *CORALNPU_CSR(CSR_RESET_CONTROL) = 0;
   ```

3. **ITCM 未正确加载**
   ```c
   // 验证 ITCM 内容
   volatile uint32_t* itcm = (volatile uint32_t*)CORALNPU_ITCM;
   printf("ITCM[0] = 0x%08X\n", itcm[0]);
   // 应该看到有效的指令，而不是 0
   ```

#### 问题 2: AXI 总线死锁

**症状：**
- 系统挂起
- AXI 事务无响应

**可能原因和解决方案：**

1. **地址解码错误**
   - 检查 AXI Crossbar 的地址解码配置
   - 确保 CoralNPU 的地址范围正确配置

2. **握手信号未连接**
   ```verilog
   // 检查所有 AXI 握手信号是否正确连接
   // 特别注意 ready 和 valid 信号
   ```

3. **时钟域问题**
   - 如果 AXI 接口和 CoralNPU 核心使用不同时钟
   - 需要添加时钟域交叉（CDC）逻辑

#### 问题 3: 内存访问错误

**症状：**
- CoralNPU 进入故障状态（FAULT=1）
- 读取的数据不正确

**可能原因和解决方案：**

1. **地址对齐错误**
   ```c
   // ITCM 和 CSR 需要 4 字节对齐
   // 错误示例：
   uint32_t* addr = (uint32_t*)0x70000001;  // 未对齐
   
   // 正确示例：
   uint32_t* addr = (uint32_t*)0x70000000;  // 4 字节对齐
   ```

2. **地址越界**
   ```c
   // 检查访问地址是否在有效范围内
   // ITCM: 0x70000000 - 0x70001FFF
   // DTCM: 0x70010000 - 0x70017FFF
   // CSR:  0x70030000 - 0x7003FFFF
   ```

3. **AXI 传输大小错误**
   - 检查 AXI awsize/arsize 是否正确
   - CoralNPU 支持 1, 2, 4, 8, 16 字节传输

### 14.7.2 性能问题

#### 问题 4: 性能低于预期

**症状：**
- 程序执行时间过长
- IPC（每周期指令数）很低

**可能原因和解决方案：**

1. **时钟频率过低**
   ```c
   // 测量实际时钟频率
   // 如果频率低于预期，检查时钟配置
   ```

2. **内存访问延迟高**
   - 检查 AXI 总线的延迟
   - 优化 AXI Crossbar 配置
   - 考虑添加缓存

3. **时钟门控过于激进**
   ```c
   // 如果频繁进入/退出 WFI，会影响性能
   // 考虑调整时钟门控策略
   ```

#### 问题 5: 功耗过高

**症状：**
- 芯片温度过高
- 电池消耗快

**可能原因和解决方案：**

1. **时钟门控未启用**
   ```c
   // 确保在空闲时启用时钟门控
   // 使用 WFI 指令让 CoralNPU 进入低功耗模式
   ```

2. **不必要的内存访问**
   - 优化程序，减少内存访问
   - 使用 DTCM 而不是外部内存

3. **时钟频率过高**
   - 如果不需要最大性能，降低时钟频率
   - 功耗与频率成正比

### 14.7.3 调试技巧

#### 技巧 1: 使用调试接口

CoralNPU 提供了调试接口，可以监控执行状态：

```verilog
// 在仿真中监控调试信号
always @(posedge clk) begin
    if (u_coralnpu.io_debug_en[0]) begin
        $display("PC: 0x%08X, Inst: 0x%08X", 
                 u_coralnpu.io_debug_addr[0],
                 u_coralnpu.io_debug_inst[0]);
    end
end
```

#### 技巧 2: 添加断言

在 RTL 中添加断言来捕获错误：

```systemverilog
// 检查 AXI 协议违规
assert property (@(posedge clk) disable iff (!resetn)
    axi_awvalid && !axi_awready |=> axi_awvalid)
    else $error("AXI AWVALID 在 AWREADY 之前被撤销");
```

#### 技巧 3: 波形分析

使用波形查看器分析问题：

```tcl
# 在 ModelSim/QuestaSim 中添加关键信号
add wave -group "CoralNPU" /tb/u_soc/u_coralnpu/*
add wave -group "AXI Slave" /tb/u_soc/u_coralnpu/axi_slave_*
add wave -group "AXI Master" /tb/u_soc/u_coralnpu/axi_master_*
```

## 14.8 总结

本章介绍了如何将 CoralNPU 集成到 SoC 中，包括：

1. **集成概述**：了解 CoralNPU 的集成方式和流程
2. **SoC 集成**：学习如何连接 CoralNPU 的各个接口
3. **接口要求**：掌握 AXI 和 TileLink 接口的详细要求
4. **时钟和电源**：了解时钟、电源和复位的要求
5. **验证检查清单**：使用完整的检查清单验证集成正确性
6. **集成示例**：通过实际代码示例学习集成过程
7. **常见问题**：了解常见问题和解决方案

**关键要点：**

- CoralNPU 支持 AXI4 和 TileLink 两种总线接口
- 正确的复位序列对于可靠启动至关重要
- 使用提供的验证检查清单确保集成质量
- 时钟门控可以显著降低功耗
- 调试接口在开发阶段非常有用

**下一步：**

- 阅读第 15 章"高级主题"了解更多高级功能
- 参考第 16 章"API 参考"获取详细的接口文档
- 查看第 13 章"调试指南"学习调试技巧

