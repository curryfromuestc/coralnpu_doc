# 总线系统

## 总线系统概述

CoralNPU 采用了混合总线架构，结合了 AXI4 和 TileLink-UL 两种总线协议。这种设计既保证了与外部 IP 的兼容性（通过 AXI4），又利用了 TileLink 协议的简洁性和灵活性来实现片上互连。

### 总线架构层次

CoralNPU 的总线系统分为三个层次：



- **核心总线层**：CPU 核心内部使用专用的 IBus（指令总线）和 DBus（数据总线）接口
- **片上互连层**：使用 TileLink-UL 协议连接各个 IP 模块和外设
- **外部接口层**：通过 AXI4 协议与外部存储器和外设通信



### 总线系统拓扑

```
                    CoralNPU 总线系统架构
                    
    +-------------+          +-------------+
    |  取指单元   |          |     LSU     |
    |  (IFetch)   |          | (Load/Store)|
    +------+------+          +------+------+
           |                        |
           | IBus                   | DBus
           |                        |
    +------v------+          +------v------+
    | IBus2Axi    |          | DBus2Axi    |
    +------+------+          +------+------+
           |                        |
           | AXI4-Read              | AXI4-Full
           |                        |
           +------------+-----------+
                        |
                   +----v----+
                   | Fabric  |  (互连结构)
                   +----+----+
                        |
           +------------+------------+
           |            |            |
      +----v----+  +----v----+  +----v----+
      |  SRAM   |  |  UART   |  |  GPIO   |
      | (Memory)|  |(外设1)  |  |(外设2)  |
      +---------+  +---------+  +---------+
```

### 总线协议选择原因

**为什么使用 AXI4？**


- AXI4 是业界标准的片上总线协议，广泛应用于 ARM 生态系统
- 大量现成的 IP 核支持 AXI4 接口，便于集成第三方 IP
- 支持高性能的突发传输和乱序执行



**为什么使用 TileLink-UL？**


- TileLink 是由 SiFive 开发的开源总线协议，特别适合 RISC-V 生态
- TileLink-UL（Uncached Lightweight）是简化版本，适合不需要缓存一致性的场景
- 协议简单，硬件开销小，适合资源受限的嵌入式系统
- 支持完整性检查（integrity checking），提高系统可靠性



## AXI4 接口

### AXI4 协议简介

AXI4（Advanced eXtensible Interface 4）是 ARM 公司定义的高性能片上总线协议。它是 AMBA（Advanced Microcontroller Bus Architecture）协议族的一部分。

\paragraph{AXI4 的基本特点}



- **独立的读写通道**：读操作和写操作使用完全独立的通道，可以并行执行
- **基于握手的传输**：每个通道使用 valid/ready 握手信号
- **支持突发传输**：一次地址传输可以对应多次数据传输
- **支持乱序执行**：通过事务 ID 标识不同的传输
- **支持多主多从**：可以连接多个主设备和从设备



\paragraph{AXI4 的五个通道}

AXI4 协议定义了五个独立的通道：



**通道名称** & **方向** & **用途** \\

AW (Write Address) & 主→从 & 传输写地址和控制信息 \\

W (Write Data) & 主→从 & 传输写数据 \\

B (Write Response) & 从→主 & 返回写操作的响应 \\

AR (Read Address) & 主→从 & 传输读地址和控制信息 \\

R (Read Data) & 从→主 & 返回读数据和响应 \\



\paragraph{AXI4 握手机制}

AXI4 的每个通道都使用 valid/ready 握手协议：



- **valid**：由发送方驱动，表示数据有效
- **ready**：由接收方驱动，表示准备接收数据
- **传输发生**：当 valid 和 ready 同时为高时，数据传输完成



```
AXI4 握手时序图

时钟:    __|‾‾|__|‾‾|__|‾‾|__|‾‾|__|‾‾|__|‾‾|__|‾‾|__

valid:   ________|‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾|______

ready:   __________________|‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾|______

data:    --------< 无效 >--< 有效数据 >--------

传输:                      ^
                           |
                      传输发生点
                      
说明：
- 第2个时钟周期：valid 变高，但 ready 还是低，传输未发生
- 第3个时钟周期：valid 保持高，ready 也变高，传输发生
- 第4个时钟周期：valid 和 ready 都变低，传输结束
```

\paragraph{AXI4 读操作时序}

```
AXI4 读操作时序图

时钟:     __|‾‾|__|‾‾|__|‾‾|__|‾‾|__|‾‾|__|‾‾|__|‾‾|__|‾‾|__

AR通道:
  arvalid: ______|‾‾‾‾‾‾|________________________________
  arready: __________|‾‾‾‾‾‾|____________________________
  araddr:  ------< 地址A >------------------------------

R通道:
  rvalid:  __________________________|‾‾‾‾‾‾|____________
  rready:  ____________________________|‾‾‾‾‾‾|__________
  rdata:   --------------------------< 数据D >----------

说明：
1. 主设备在 AR 通道发送读地址
2. 从设备接收地址后，准备数据
3. 从设备在 R 通道返回读数据
4. 地址传输和数据传输之间可能有延迟
```

\paragraph{AXI4 写操作时序}

```
AXI4 写操作时序图

时钟:     __|‾‾|__|‾‾|__|‾‾|__|‾‾|__|‾‾|__|‾‾|__|‾‾|__|‾‾|__

AW通道:
  awvalid: ______|‾‾‾‾‾‾|________________________________
  awready: __________|‾‾‾‾‾‾|____________________________
  awaddr:  ------< 地址A >------------------------------

W通道:
  wvalid:  ______|‾‾‾‾‾‾|________________________________
  wready:  __________|‾‾‾‾‾‾|____________________________
  wdata:   ------< 数据D >------------------------------

B通道:
  bvalid:  __________________________|‾‾‾‾‾‾|____________
  bready:  ____________________________|‾‾‾‾‾‾|__________
  bresp:   --------------------------< 响应 >-----------

说明：
1. 主设备在 AW 通道发送写地址
2. 主设备在 W 通道发送写数据
3. AW 和 W 可以同时发送，也可以有先后顺序
4. 从设备完成写操作后，在 B 通道返回响应
```

### CoralNPU 中的 AXI4 使用

\paragraph{AXI4 信号定义}

在 CoralNPU 中，AXI4 接口的信号定义在 `Axi.scala` 文件中：

```
// 地址通道（AR 和 AW 共用的结构）
class AxiAddress(addrWidthBits: Int, dataWidthBits: Int, idBits: Int) {
  val addr   = UInt(addrWidthBits.W)    // 地址
  val prot   = UInt(3.W)                 // 保护类型
  val id     = UInt(idBits.W)            // 事务 ID
  val len    = UInt(8.W)                 // 突发长度
  val size   = UInt(3.W)                 // 传输大小
  val burst  = UInt(2.W)                 // 突发类型
  val lock   = UInt(1.W)                 // 锁定类型
  val cache  = UInt(4.W)                 // 缓存属性
  val qos    = UInt(4.W)                 // 服务质量
  val region = UInt(4.W)                 // 区域标识
}

// 写数据通道
class AxiWriteData(dataWidthBits: Int, idBits: Int) {
  val data = UInt(dataWidthBits.W)      // 写数据
  val last = Bool()                      // 最后一次传输
  val strb = UInt((dataWidthBits/8).W)  // 字节选通（哪些字节有效）
}

// 写响应通道
class AxiWriteResponse(idBits: Int) {
  val id   = UInt(idBits.W)             // 事务 ID
  val resp = UInt(2.W)                  // 响应类型
}

// 读数据通道
class AxiReadData(dataWidthBits: Int, idBits: Int) {
  val data = UInt(dataWidthBits.W)      // 读数据
  val id   = UInt(idBits.W)             // 事务 ID
  val resp = UInt(2.W)                  // 响应类型
  val last = Bool()                      // 最后一次传输
}
```

\paragraph{AXI4 响应类型}

AXI4 定义了四种响应类型：

```
object AxiResponseType {
  val OKAY   = 0  // 正常访问成功
  val EXOKAY = 1  // 独占访问成功
  val SLVERR = 2  // 从设备错误
  val DECERR = 3  // 解码错误（地址无效）
}
```

\paragraph{AXI4 突发类型}

```
object AxiBurstType {
  val FIXED = 0  // 固定地址（地址不变）
  val INCR  = 1  // 递增地址（最常用）
  val WRAP  = 2  // 回卷地址（用于缓存行）
}
```

### AXI4 主设备接口

CoralNPU 定义了完整的 AXI4 主设备接口：

```
class AxiMasterIO(addrWidthBits: Int, dataWidthBits: Int, idBits: Int) {
  val write = new AxiMasterWriteIO(...)  // 写通道组
  val read  = new AxiMasterReadIO(...)   // 读通道组
}
```

**写通道组**包含：


- `write.addr`：写地址通道（AW）
- `write.data`：写数据通道（W）
- `write.resp`：写响应通道（B）



**读通道组**包含：


- `read.addr`：读地址通道（AR）
- `read.data`：读数据通道（R）




## TileLink-UL 总线

### TileLink 协议简介

TileLink 是由 SiFive 公司开发的开源片上互连协议，专为 RISC-V 生态系统设计。TileLink 有多个版本，CoralNPU 使用的是 **TileLink-UL（Uncached Lightweight）**，这是一个简化版本，适合不需要缓存一致性的场景。

\paragraph{TileLink 的基本特点}



- **简洁的协议**：相比 AXI4，TileLink 的信号更少，逻辑更简单
- **双向通道**：只有两个通道（A 通道和 D 通道），而不是五个
- **统一的操作码**：使用操作码区分读写操作
- **完整性检查**：支持端到端的数据完整性校验（ECC）
- **灵活的数据宽度**：支持不同宽度的数据传输



\paragraph{TileLink-UL 的两个通道}



**通道名称** & **方向** & **用途** \\

A (Request) & 主→从 & 发送请求（读或写） \\

D (Response) & 从→主 & 返回响应（数据或确认） \\



\paragraph{TileLink-UL 操作类型}

**A 通道操作码**（请求）：

```
object TLULOpcodesA {
  val PutFullData    = 0  // 写入完整数据（所有字节）
  val PutPartialData = 1  // 写入部分数据（部分字节）
  val Get            = 4  // 读取数据
}
```

**D 通道操作码**（响应）：

```
object TLULOpcodesD {
  val AccessAck     = 0  // 写操作确认（无数据）
  val AccessAckData = 1  // 读操作确认（带数据）
}
```

### TileLink-UL 信号定义

\paragraph{A 通道（请求通道）}

```
class TileLink_A_Channel(p: TLULParameters) {
  val opcode  = UInt(3.W)        // 操作码（Get/Put）
  val param   = UInt(3.W)        // 参数（通常为 0）
  val size    = UInt(z.W)        // 传输大小（log2 字节数）
  val source  = UInt(o.W)        // 源 ID（类似 AXI 的 ID）
  val address = UInt(a.W)        // 地址
  val mask    = UInt(w.W)        // 字节掩码（哪些字节有效）
  val data    = UInt((8*w).W)    // 数据（写操作时使用）
  val user    = ...              // 用户自定义字段
}
```

**字段说明**：



- **opcode**：指定操作类型（读/写）
- **size**：传输大小的对数值。例如：
  

  - size = 0 表示 1 字节（$2^0$）
  - size = 1 表示 2 字节（$2^1$）
  - size = 2 表示 4 字节（$2^2$）
  - size = 3 表示 8 字节（$2^3$）
  

- **source**：事务标识符，用于匹配请求和响应
- **mask**：字节掩码，指示哪些字节需要写入（仅用于写操作）
- **data**：写数据（仅用于写操作）



\paragraph{D 通道（响应通道）}

```
class TileLink_D_Channel(p: TLULParameters) {
  val opcode = UInt(3.W)         // 操作码（AccessAck/AccessAckData）
  val param  = UInt(3.W)         // 参数（通常为 0）
  val size   = UInt(z.W)         // 传输大小
  val source = UInt(o.W)         // 源 ID（与请求匹配）
  val sink   = UInt(i.W)         // 汇 ID（通常为 0）
  val data   = UInt((8*w).W)     // 数据（读操作时使用）
  val user   = ...               // 用户自定义字段
  val error  = Bool()            // 错误标志
}
```

**字段说明**：



- **opcode**：响应类型（带数据或不带数据）
- **source**：必须与请求的 source 字段匹配
- **data**：读数据（仅用于读操作）
- **error**：指示操作是否出错



### TileLink-UL 操作时序

\paragraph{读操作时序}

```
TileLink-UL 读操作时序图

时钟:     __|‾‾|__|‾‾|__|‾‾|__|‾‾|__|‾‾|__|‾‾|__|‾‾|__|‾‾|__

A通道 (请求):
  a_valid:  ______|‾‾‾‾‾‾|________________________________
  a_ready:  __________|‾‾‾‾‾‾|____________________________
  a_opcode: ------< Get >--------------------------------
  a_addr:   ------< 地址 >-------------------------------
  a_source: ------< ID >---------------------------------

D通道 (响应):
  d_valid:  __________________________|‾‾‾‾‾‾|____________
  d_ready:  ____________________________|‾‾‾‾‾‾|__________
  d_opcode: --------------------------< AckData >--------
  d_data:   --------------------------< 数据 >-----------
  d_source: --------------------------< ID >-------------

说明：
1. 主设备在 A 通道发送 Get 请求
2. 从设备接收请求后，准备数据
3. 从设备在 D 通道返回 AccessAckData 响应
4. source ID 在请求和响应中保持一致
```

\paragraph{写操作时序}

```
TileLink-UL 写操作时序图

时钟:     __|‾‾|__|‾‾|__|‾‾|__|‾‾|__|‾‾|__|‾‾|__|‾‾|__|‾‾|__

A通道 (请求):
  a_valid:  ______|‾‾‾‾‾‾|________________________________
  a_ready:  __________|‾‾‾‾‾‾|____________________________
  a_opcode: ------< Put >--------------------------------
  a_addr:   ------< 地址 >-------------------------------
  a_data:   ------< 数据 >-------------------------------
  a_mask:   ------< 掩码 >-------------------------------
  a_source: ------< ID >---------------------------------

D通道 (响应):
  d_valid:  __________________________|‾‾‾‾‾‾|____________
  d_ready:  ____________________________|‾‾‾‾‾‾|__________
  d_opcode: --------------------------< Ack >------------
  d_source: --------------------------< ID >-------------

说明：
1. 主设备在 A 通道发送 Put 请求，同时携带数据
2. 从设备接收请求和数据，执行写操作
3. 从设备在 D 通道返回 AccessAck 响应
4. 写操作的响应不包含数据
```

### OpenTitan TileLink 扩展

CoralNPU 使用了 OpenTitan 项目定义的 TileLink 扩展，增加了完整性检查功能：

```
class OpenTitanTileLink_A_User {
  val rsvd        = UInt(5.W)   // 保留字段
  val instr_type  = UInt(4.W)   // 指令类型（mubi4_t）
  val cmd_intg    = UInt(7.W)   // 命令完整性校验码
  val data_intg   = UInt(7.W)   // 数据完整性校验码
}

class OpenTitanTileLink_D_User {
  val rsp_intg    = UInt(7.W)   // 响应完整性校验码
  val data_intg   = UInt(7.W)   // 数据完整性校验码
}
```

**完整性检查的作用**：


- 检测传输过程中的数据损坏
- 提高系统的可靠性和安全性
- 符合功能安全标准的要求



### TileLink 与 AXI4 的对比



**特性** & **TileLink-UL** & **AXI4** \\

通道数量 & 2（A, D） & 5（AW, W, B, AR, R） \\

读写分离 & 否（共用 A 通道） & 是（独立的读写通道） \\

协议复杂度 & 简单 & 复杂 \\

突发传输 & 不支持 & 支持 \\

完整性检查 & 内置支持 & 需要额外扩展 \\

生态系统 & RISC-V & ARM \\

适用场景 & 片上互连 & 片上互连、外部接口 \\



## 指令总线 (IBus)

### IBus 接口定义

指令总线（IBus）连接取指单元（IFetch）和指令缓存/内存。它是一个简化的只读接口，专门用于取指令。

```
class IBusIO(p: Parameters) {
  val valid = Output(Bool())              // 请求有效
  val addr  = Output(UInt(32.W))          // 指令地址
  val ready = Input(Bool())               // 数据就绪
  val rdata = Input(UInt(p.lsuDataBits.W)) // 读取的指令数据
}
```

**信号说明**：



- **valid**：取指单元发出，表示需要取指令
- **addr**：指令地址（32 位）
- **ready**：内存返回，表示指令数据已准备好
- **rdata**：返回的指令数据



### IBus 操作流程

```
IBus 取指时序图

时钟:    __|‾‾|__|‾‾|__|‾‾|__|‾‾|__|‾‾|__|‾‾|__|‾‾|__|‾‾|__

valid:   ______|‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾|______

addr:    ------< PC1 >< PC2 >< PC3 >------------------

ready:   __________________|‾‾‾‾‾‾|__|‾‾‾‾‾‾|__|‾‾‾‾‾‾|__

rdata:   ------------------< I1 >--< I2 >--< I3 >------

说明：
1. 取指单元在 valid 为高时发出地址
2. 地址可以连续变化（流水线取指）
3. 内存在 ready 为高时返回指令数据
4. 可能存在延迟（ready 不一定立即响应）
```

### IBus 到 AXI4 的转换

`IBus2Axi` 模块负责将 IBus 接口转换为 AXI4 读接口：

```
class IBus2Axi(p: Parameters) {
  val io = IO(new Bundle {
    val ibus = Flipped(new IBusIO(p))      // IBus 从接口
    val axi  = new AxiMasterReadIO(...)    // AXI4 主接口
  })
}
```

**转换逻辑**：



- **地址对齐**：将 IBus 地址对齐到缓存行边界
```
val linebit = log2Ceil(p.lsuDataBits / 8)
val saddr = Cat(io.ibus.addr(31, linebit), 0.U(linebit.W))
```

- **状态机**：使用寄存器跟踪 AXI 传输状态
```
val sraddrActive = RegInit(false.B)  // 地址已发送
val sdata = RegInit(0.U(...))        // 缓存的数据
```

- **AXI 读地址通道**：
```
io.axi.addr.valid := io.ibus.valid && !sraddrActive
io.axi.addr.bits.addr := saddr
io.axi.addr.bits.prot := 2.U  // 指令访问
```

- **AXI 读数据通道**：
```
io.axi.data.ready := true.B
when (io.axi.data.valid && io.axi.data.ready) {
  sdata := io.axi.data.bits.data
}
```

- **IBus 响应**：
```
io.ibus.ready := io.axi.data.valid && sraddrActive
io.ibus.rdata := sdata
```



### IBus 特点



- **只读接口**：只支持读操作，不支持写操作
- **简单协议**：没有复杂的握手和状态
- **低延迟**：针对取指优化，减少延迟
- **地址对齐**：自动对齐到缓存行边界




## 数据总线 (DBus)

### DBus 接口定义

数据总线（DBus）连接加载存储单元（LSU）和数据缓存/内存。它支持读写操作，并且可以处理不同大小的数据传输。

```
class DBusIO(p: Parameters) {
  val valid = Output(Bool())               // 请求有效
  val write = Output(Bool())               // 写操作标志
  val addr  = Output(UInt(32.W))           // 访问地址
  val size  = Output(UInt(4.W))            // 访问大小（字节掩码）
  val wdata = Output(UInt(p.axi2DataBits.W)) // 写数据
  val wmask = Output(UInt((p.axi2DataBits/8).W)) // 写掩码
  val pc    = Output(UInt(p.programCounterBits.W)) // 程序计数器
  val ready = Input(Bool())                // 操作完成
  val rdata = Input(UInt(p.axi2DataBits.W)) // 读数据
}
```

**信号说明**：



- **valid**：LSU 发出，表示有访存请求
- **write**：指示是写操作（true）还是读操作（false）
- **addr**：访问地址
- **size**：访问大小，使用独热编码（one-hot）：
  

  - 0001 = 1 字节
  - 0010 = 2 字节
  - 0100 = 4 字节
  - 1000 = 8 字节
  

- **wdata**：写数据（仅写操作使用）
- **wmask**：字节写掩码（仅写操作使用）
- **pc**：当前指令的 PC，用于异常处理
- **ready**：内存返回，表示操作完成
- **rdata**：读数据（仅读操作使用）



### DBus 操作时序

\paragraph{读操作时序}

```
DBus 读操作时序图

时钟:    __|‾‾|__|‾‾|__|‾‾|__|‾‾|__|‾‾|__|‾‾|__|‾‾|__|‾‾|__

valid:   ______|‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾|____________

write:   ______________________________________________|__

addr:    ------< 地址 >--------------------------------

size:    ------< 大小 >--------------------------------

ready:   __________________________|‾‾‾‾‾‾|____________

rdata:   --------------------------< 数据 >-----------

说明：
1. LSU 发出读请求（valid=1, write=0）
2. 提供地址和访问大小
3. 内存准备数据后，ready 变高
4. LSU 在 ready 为高时读取 rdata
```

\paragraph{写操作时序}

```
DBus 写操作时序图

时钟:    __|‾‾|__|‾‾|__|‾‾|__|‾‾|__|‾‾|__|‾‾|__|‾‾|__|‾‾|__

valid:   ______|‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾|____________

write:   ______|‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾|____________

addr:    ------< 地址 >--------------------------------

size:    ------< 大小 >--------------------------------

wdata:   ------< 数据 >--------------------------------

wmask:   ------< 掩码 >--------------------------------

ready:   __________________________|‾‾‾‾‾‾|____________

说明：
1. LSU 发出写请求（valid=1, write=1）
2. 提供地址、数据、大小和掩码
3. 内存完成写操作后，ready 变高
4. LSU 在 ready 为高时知道写操作完成
```

### DBus 到 AXI4 的转换

`DBus2Axi` 模块负责将 DBus 接口转换为完整的 AXI4 接口（读+写）：

```
class DBus2Axi(p: Parameters) {
  val io = IO(new Bundle {
    val dbus  = Flipped(new DBusIO(p))     // DBus 从接口
    val axi   = new AxiMasterIO(...)       // AXI4 主接口
    val fault = Valid(new FaultInfo(p))    // 故障信息
  })
}
```

\paragraph{写路径实现}

写操作需要协调三个 AXI 通道：AW（写地址）、W（写数据）、B（写响应）。

```
// 写地址通道
val waddrFired = RegInit(false.B)
io.axi.write.addr.valid := !waddrFired && io.dbus.valid && io.dbus.write
io.axi.write.addr.bits.addr := io.dbus.addr
io.axi.write.addr.bits.size := Ctz(io.dbus.size)  // 转换为 log2 大小

// 写数据通道（使用队列缓冲）
val wdataQueue = Module(new Queue(new AxiWriteData(...), 2))
wdataQueue.io.enq.valid := !wdataFired && io.dbus.valid && io.dbus.write
wdataQueue.io.enq.bits.data := io.dbus.wdata
wdataQueue.io.enq.bits.strb := io.dbus.wmask

// 写响应通道
val wrespReceived = RegInit(false.B)
io.axi.write.resp.ready := !wrespReceived && io.dbus.valid && io.dbus.write

// 写操作完成条件
val writeFinished = (io.axi.write.addr.fire || waddrFired) &&
                    (wdataQueue.io.enq.fire || wdataFired) &&
                    (io.axi.write.resp.fire || wrespReceived)
```

**状态跟踪**：


- `waddrFired`：写地址已发送
- `wdataFired`：写数据已发送
- `wrespReceived`：写响应已接收
- 只有三个都完成，写操作才算完成



\paragraph{读路径实现}

读操作需要协调两个 AXI 通道：AR（读地址）、R（读数据）。

```
// 读地址通道
val raddrFired = RegInit(false.B)
io.axi.read.addr.valid := !raddrFired && io.dbus.valid && !io.dbus.write
io.axi.read.addr.bits.addr := io.dbus.addr
io.axi.read.addr.bits.size := Ctz(io.dbus.size)

// 读数据通道
val rdataReceived = RegInit(MakeInvalid(UInt(p.axi2DataBits.W)))
io.axi.read.data.ready := !rdataReceived.valid && io.dbus.valid && !io.dbus.write

// 读操作完成条件
val readFinished = (io.axi.read.addr.fire || raddrFired) &&
                   (io.axi.read.data.fire || rdataReceived.valid)

// 插入延迟寄存器以匹配 DBus 接口预期
val readNext = RegInit(0.U(p.axi2DataBits.W))
readNext := Mux(readFinished, 
                Mux(io.axi.read.data.fire, io.axi.read.data.bits.data, 
                    rdataReceived.bits),
                readNext)
io.dbus.rdata := readNext
```

\paragraph{故障处理}

DBus2Axi 模块还负责检测和报告访存故障：

```
io.fault.valid := io.dbus.valid && Mux(
  io.dbus.write,
  // 写故障：检查写响应
  io.axi.write.resp.valid && 
    (io.axi.write.resp.bits.resp =/= AxiResponseType.OKAY),
  // 读故障：检查读响应
  io.axi.read.data.valid && 
    (io.axi.read.data.bits.resp =/= AxiResponseType.OKAY)
)

io.fault.bits.write := io.dbus.write
io.fault.bits.addr  := io.dbus.addr
io.fault.bits.epc   := io.dbus.pc  // 用于异常处理
```

**故障类型**：


- **SLVERR**：从设备错误（例如：写只读内存）
- **DECERR**：解码错误（例如：访问不存在的地址）



### DBus 特点



- **读写支持**：支持完整的读写操作
- **灵活的大小**：支持 1/2/4/8 字节访问
- **字节掩码**：支持部分字节写入
- **故障检测**：检测并报告访存错误
- **PC 跟踪**：记录 PC 用于异常处理



## 互连结构 (Fabric)

### Fabric 概述

Fabric 是 CoralNPU 的片上互连结构，负责连接多个主设备（如 CPU 核心）和多个从设备（如 SRAM、外设）。它实现了仲裁、路由和地址映射功能。

### Fabric 接口定义

Fabric 使用自定义的 `FabricIO` 接口，这是一个简化的内存访问接口：

```
class FabricIO(p: Parameters) {
  val readDataAddr  = Valid(UInt(p.axi2AddrBits.W))    // 读地址
  val writeDataAddr = Valid(UInt(p.axi2AddrBits.W))    // 写地址
  val writeDataBits = UInt(p.axi2DataBits.W)           // 写数据
  val writeDataStrb = UInt((p.axi2DataBits/8).W)       // 写掩码
  val readData      = Flipped(Valid(UInt(p.axi2DataBits.W))) // 读数据
  val writeResp     = Input(Bool())                    // 写响应
}
```

**设计特点**：


- 使用 `Valid` 接口（只有 valid 和 bits，没有 ready）
- 读写地址分离，可以同时发起读写请求
- 简化的握手协议，适合片上互连



### Fabric 仲裁器 (FabricArbiter)

当多个主设备需要访问同一个从设备时，需要仲裁器来决定优先级。

```
class FabricArbiter(p: Parameters, n: Int = 2) {
  val io = IO(new Bundle {
    val source = Vec(n, Flipped(new FabricIO(p)))  // n 个输入端口
    val fabricBusy = Output(Vec(n, Bool()))        // 反压信号
    val port = new FabricIO(p)                     // 1 个输出端口
  })
}
```

\paragraph{仲裁策略}

FabricArbiter 使用**固定优先级仲裁**：

```
val sourceValid = io.source.map(x => 
  x.readDataAddr.valid || x.writeDataAddr.valid)

// fabricBusy(i) 为真，如果有更高优先级的源（索引 < i）正在使用
val busySignals = sourceValid.scanLeft(false.B)(_ || _).dropRight(1)
io.fabricBusy := VecInit(busySignals)
```

**优先级规则**：


- 索引小的端口优先级高
- 如果端口 0 有请求，端口 1 和 2 都会被阻塞
- 如果端口 0 空闲，端口 1 有请求，端口 2 会被阻塞



\paragraph{请求选择}

```
io.port.readDataAddr := MuxCase(MakeInvalid(UInt(p.axi2AddrBits.W)),
  (0 until n).map(x => (sourceValid(x) -> io.source(x).readDataAddr))
)

io.port.writeDataAddr := MuxCase(MakeInvalid(UInt(p.axi2AddrBits.W)),
  (0 until n).map(x => (sourceValid(x) -> io.source(x).writeDataAddr))
)
```

**选择逻辑**：


- 使用 `MuxCase` 选择第一个有效的请求
- 读写请求独立选择
- 默认输出无效信号



\paragraph{响应广播}

```
for (i <- 0 until n) {
  io.source(i).readData := io.port.readData
  io.source(i).writeResp := io.port.writeResp
}
```

**广播策略**：


- 所有源端口都接收相同的响应
- 每个源端口根据自己的请求状态决定是否使用响应



### Fabric 多路复用器 (FabricMux)

当一个主设备需要访问多个从设备时，需要多路复用器来路由请求。

```
class FabricMux(p: Parameters, regions: Seq[MemoryRegion]) {
  val portCount = regions.length
  val io = IO(new Bundle {
    val source = Flipped(new FabricIO(p))       // 1 个输入端口
    val fabricBusy = Output(Bool())             // 反压信号
    val ports = Vec(portCount, new FabricIO(p)) // n 个输出端口
    val periBusy = Vec(portCount, Input(Bool())) // 外设忙信号
  })
}
```

\paragraph{地址解码}

```
val addr = MuxUpTo1H(0.U, Seq(
  io.source.readDataAddr.valid -> io.source.readDataAddr.bits,
  io.source.writeDataAddr.valid -> io.source.writeDataAddr.bits,
))

val selected = MuxCase(MakeInvalid(portIdxType), (0 until portCount).map(
  x => (sourceValid && regions(x).contains(addr)) ->
      MakeValid(true.B, x.U(portIdxBits.W))
))
```

**解码逻辑**：


- 根据访问地址确定目标端口
- 使用 `MemoryRegion` 定义的地址范围
- 如果地址不在任何范围内，`selected.valid` 为假



\paragraph{地址重映射}

```
for (i <- 0 until portCount) {
  val readAddr = io.source.readDataAddr.bits & 
                 ~regions(i).memStart.U(...)
  val writeAddr = io.source.writeDataAddr.bits & 
                  ~regions(i).memStart.U(...)
  
  io.ports(i).readDataAddr.bits := Mux(portSelected(i), readAddr, 0.U)
  io.ports(i).writeDataAddr.bits := Mux(portSelected(i), writeAddr, 0.U)
}
```

**重映射原理**：


- 将全局地址转换为局部地址
- 例如：全局地址 0x8000\_0100 访问起始地址为 0x8000\_0000 的 SRAM
- 局部地址 = 0x8000\_0100 \& \textasciitilde 0x8000\_0000 = 0x0100



\paragraph{响应路由}

```
val lastReadSelected = RegInit(MakeInvalid(portIdxType))
lastReadSelected := MuxUpTo1H(MakeInvalid(portIdxType), 
  (0 until portCount).map(
    i => (portSelected(i) && io.source.readDataAddr.valid) ->
        MakeValid(true.B, i.U(portIdxBits.W))
))

io.source.readData := MuxUpTo1H(MakeInvalid(UInt(p.axi2DataBits.W)),
  (0 until portCount).map(i =>
    (lastReadSelected.valid && (lastReadSelected.bits === i.U)) ->
        io.ports(i).readData
  )
)
```

**响应选择**：


- 使用寄存器记住上一次选择的端口
- 延迟一个周期以匹配外设的响应延迟
- 根据记录的端口选择正确的响应数据




### Fabric 连接示例

```
Fabric 多主多从连接示例

        主设备                    从设备
        
    +----------+              +----------+
    |  CPU 0   |              |  SRAM    |
    | (IFetch) |              | 0x0000   |
    +----+-----+              +-----+----+
         |                          ^
         |                          |
    +----v-----+              +-----+----+
    |  CPU 1   |              |  UART    |
    |  (LSU)   |              | 0x1000   |
    +----+-----+              +-----+----+
         |                          ^
         |                          |
         +------+         +---------+
                |         |
           +----v---------v----+
           |  FabricArbiter   |  (仲裁)
           +--------+---------+
                    |
           +--------v---------+
           |   FabricMux      |  (路由)
           +--------+---------+
                    |
         +----------+----------+
         |                     |
    +----v----+          +-----v----+
    |  GPIO   |          |   I2C    |
    | 0x2000  |          |  0x3000  |
    +---------+          +----------+
```

### 内存映射配置

Fabric 使用 `MemoryRegion` 定义内存映射：

```
case class MemoryRegion(
  memStart: Long,   // 起始地址
  memSize: Long     // 大小（字节）
) {
  def contains(addr: UInt): Bool = {
    val start = memStart.U
    val end = (memStart + memSize).U
    (addr >= start) && (addr < end)
  }
}
```

**典型的内存映射**：



**设备** & **起始地址** & **大小** & **说明** \\

SRAM & 0x0000\_0000 & 64KB & 片上 SRAM \\

UART & 0x1000\_0000 & 4KB & 串口外设 \\

GPIO & 0x2000\_0000 & 4KB & GPIO 外设 \\

I2C  & 0x3000\_0000 & 4KB & I2C 外设 \\

SPI  & 0x4000\_0000 & 4KB & SPI 外设 \\



## TileLink 互连组件

除了 Fabric，CoralNPU 还使用了基于 TileLink 的互连组件，提供更标准化的片上互连。

### TlulSocketM1（多主到单从）

`TlulSocketM1` 将多个 TileLink 主设备连接到一个从设备：

```
class TlulSocketM1(
  p: TLULParameters,
  M: Int = 4,                    // 主设备数量
  HReqDepth: Seq[Int] = Nil,     // 主侧请求 FIFO 深度
  HRspDepth: Seq[Int] = Nil,     // 主侧响应 FIFO 深度
  DReqDepth: Int = 1,            // 从侧请求 FIFO 深度
  DRspDepth: Int = 1             // 从侧响应 FIFO 深度
)
```

\paragraph{工作原理}

```
TlulSocketM1 结构

主设备 0 ──┐
           ├─> [FIFO] ──┐
主设备 1 ──┤            │
           ├─> [FIFO] ──┤
主设备 2 ──┤            ├─> [仲裁器] ─> [FIFO] ─> 从设备
           ├─> [FIFO] ──┤
主设备 3 ──┘            │
           └─> [FIFO] ──┘
```

**关键特性**：



- **Source ID 扩展**：
```
// 将主设备索引添加到 source ID 的低位
hreq_fifo_i.source := Cat(io.tl_h(i).a.bits.source, i.U(StIdW.W))
```
这样可以在响应时识别请求来自哪个主设备。

- **轮询仲裁**：
```
val arb = Module(new CoralNPURRArbiter(
  new OpenTitanTileLink.A_Channel(p), M))
```
使用轮询（Round-Robin）仲裁器，保证公平性。

- **响应路由**：
```
val rsp_arb_grant = Mux(io.tl_d.d.valid, 
                        UIntToOH(io.tl_d.d.bits.source(StIdW - 1, 0)), 
                        0.U(M.W))
```
根据 source ID 的低位将响应路由到正确的主设备。



### TlulSocket1N（单主到多从）

`TlulSocket1N` 将一个 TileLink 主设备连接到多个从设备：

```
class TlulSocket1N(
  p: TLULParameters,
  N: Int = 4,                    // 从设备数量
  DReqDepth: Seq[Int] = Nil,     // 从侧请求 FIFO 深度
  DRspDepth: Seq[Int] = Nil,     // 从侧响应 FIFO 深度
  ExplicitErrs: Boolean = true   // 是否显式处理错误
)
```

\paragraph{工作原理}

```
TlulSocket1N 结构

                    +------------------+
                    | 地址解码 & 路由  |
                    +------------------+
                            |
主设备 ─> [FIFO] ─> +-------+-------+-------+
                    |       |       |       |
                 [FIFO]  [FIFO]  [FIFO]  [错误响应器]
                    |       |       |       |
                    v       v       v       v
                 从设备0  从设备1  从设备2  (无效地址)
```

**关键特性**：



- **外部地址选择**：
```
val dev_select_i = Input(UInt(NWD.W))  // 由外部逻辑提供
```
地址解码在外部完成，提高灵活性。

- **未完成请求跟踪**：
```
val num_req_outstanding = RegInit(0.U(outstandingW.W))
val dev_select_outstanding = RegInit(0.U(NWD.W))
```
跟踪未完成的请求，确保响应路由到正确的从设备。

- **请求阻塞**：
```
val hold_all_requests = 
  (num_req_outstanding =/= 0.U) && 
  (dev_select_t =/= dev_select_outstanding)
```
如果有未完成的请求，且新请求的目标不同，则阻塞新请求。

- **错误响应器**：
```
if (ExplicitErrs && (1 << NWD) > N) {
  val err_resp = Module(new TlulErrorResponder(p))
  // 处理访问无效地址的请求
}
```
自动为无效地址生成错误响应。



### TileLink FIFO (TlulFifoSync)

`TlulFifoSync` 是一个同步 FIFO，用于缓冲 TileLink 请求和响应：

```
class TlulFifoSync(
  p: TLULParameters,
  reqDepth: Int,      // 请求 FIFO 深度
  rspDepth: Int,      // 响应 FIFO 深度
  reqPass: Boolean,   // 请求是否直通（深度为 0）
  rspPass: Boolean    // 响应是否直通（深度为 0）
)
```

**用途**：


- **时序优化**：打断长组合逻辑路径
- **缓冲**：吸收突发流量
- **解耦**：允许主从设备以不同速率工作



### TileLink 到 AXI4 转换 (TLUL2Axi)

`TLUL2Axi` 模块将 TileLink-UL 接口转换为 AXI4 接口：

```
class TLUL2Axi(p_tl: Parameters, p_axi: Parameters) {
  val io = IO(new Bundle {
    val tl_a = Flipped(Decoupled(new TileLink_A_Channel(...)))
    val tl_d = Decoupled(new TileLink_D_Channel(...))
    val axi  = new AxiMasterIO(...)
  })
}
```

\paragraph{转换逻辑}

**请求转换**：

```
val is_get = tl_a.bits.opcode === TLULOpcodesA.Get.asUInt
val is_put = tl_a.bits.opcode === TLULOpcodesA.PutFullData.asUInt ||
             tl_a.bits.opcode === TLULOpcodesA.PutPartialData.asUInt

// Get -> AXI 读
io.axi.read.addr.valid := tl_a.valid && is_get
io.axi.read.addr.bits.addr := tl_a.bits.address
io.axi.read.addr.bits.size := tl_a.bits.size

// Put -> AXI 写
io.axi.write.addr.valid := tl_a.valid && is_put
io.axi.write.addr.bits.addr := tl_a.bits.address
io.axi.write.data.bits.data := tl_a.bits.data
io.axi.write.data.bits.strb := tl_a.bits.mask
```

**响应转换**：

```
// AXI 读响应 -> TileLink AccessAckData
read_resp.opcode := TLULOpcodesD.AccessAckData.asUInt
read_resp.data := io.axi.read.data.bits.data
read_resp.error := io.axi.read.data.bits.resp =/= 0.U

// AXI 写响应 -> TileLink AccessAck
write_resp.opcode := TLULOpcodesD.AccessAck.asUInt
write_resp.error := io.axi.write.resp.bits.resp =/= 0.U
```

**ID 重映射**：

如果 TileLink source ID 宽度大于 AXI ID 宽度，使用 `TlulIdRemapper` 进行转换：

```
if (idWidthMismatch) {
  id_remapper = Module(new TlulIdRemapper(...))
  // 限制同时进行的事务数量
  // 将宽 ID 映射到窄 ID
}
```

## 总线系统设计要点

### 性能优化



- **流水线化**：
   

   - 使用 FIFO 打断长组合逻辑路径
   - 允许请求和响应重叠
   


- **并行性**：
   

   - 读写通道独立（AXI4）
   - 多个主设备可以同时访问不同的从设备
   


- **缓冲**：
   

   - 在关键路径上添加队列
   - 吸收突发流量，提高吞吐量
   




### 可靠性设计



- **完整性检查**：
   

   - TileLink 支持端到端的 ECC 校验
   - 检测传输过程中的数据损坏
   


- **错误处理**：
   

   - 检测并报告访存错误
   - 自动生成错误响应（无效地址）
   


- **死锁避免**：
   

   - 使用 FIFO 避免循环依赖
   - 限制未完成事务的数量
   




### 可扩展性



- **参数化设计**：
   

   - 数据宽度、地址宽度、ID 宽度都可配置
   - FIFO 深度可调整
   


- **模块化**：
   

   - 仲裁器、多路复用器、转换器都是独立模块
   - 可以灵活组合
   


- **协议转换**：
   

   - 支持 TileLink 和 AXI4 互转
   - 便于集成不同协议的 IP
   




### 调试支持



- **事务跟踪**：
   

   - 使用 ID 跟踪每个事务
   - 记录 PC 用于异常定位
   


- **断言检查**：
   

   - 检测协议违规
   - 检测非法状态
   


- **性能计数器**：
   

   - 统计总线利用率
   - 识别性能瓶颈
   




## 总线使用示例

### 示例 1：CPU 取指令

```
1. IFetch 发出取指请求
   IBus.valid = 1
   IBus.addr = 0x0000_1000

2. IBus2Axi 转换为 AXI 读请求
   AXI.read.addr.valid = 1
   AXI.read.addr.bits.addr = 0x0000_1000
   AXI.read.addr.bits.prot = 2 (指令访问)

3. Fabric 路由到 SRAM
   根据地址 0x0000_1000 选择 SRAM 端口

4. SRAM 返回指令数据
   AXI.read.data.valid = 1
   AXI.read.data.bits.data = 0x00000013 (nop 指令)

5. IBus2Axi 返回给 IFetch
   IBus.ready = 1
   IBus.rdata = 0x00000013
```

### 示例 2：CPU 写内存

```
1. LSU 发出写请求
   DBus.valid = 1
   DBus.write = 1
   DBus.addr = 0x0000_2000
   DBus.wdata = 0x12345678
   DBus.wmask = 0xF (4 字节全写)

2. DBus2Axi 转换为 AXI 写请求
   AXI.write.addr.valid = 1
   AXI.write.addr.bits.addr = 0x0000_2000
   AXI.write.data.valid = 1
   AXI.write.data.bits.data = 0x12345678
   AXI.write.data.bits.strb = 0xF

3. Fabric 路由到 SRAM
   根据地址 0x0000_2000 选择 SRAM 端口

4. SRAM 完成写操作
   AXI.write.resp.valid = 1
   AXI.write.resp.bits.resp = 0 (OKAY)

5. DBus2Axi 返回给 LSU
   DBus.ready = 1
```

### 示例 3：访问外设

```
1. LSU 发出读请求（读 UART 状态）
   DBus.valid = 1
   DBus.write = 0
   DBus.addr = 0x1000_0000

2. DBus2Axi 转换为 AXI 读请求
   AXI.read.addr.valid = 1
   AXI.read.addr.bits.addr = 0x1000_0000

3. Fabric 路由到 UART
   根据地址 0x1000_0000 选择 UART 端口
   地址重映射：0x1000_0000 -> 0x0000（UART 本地地址）

4. UART 返回状态寄存器
   AXI.read.data.valid = 1
   AXI.read.data.bits.data = 0x00000001 (TX ready)

5. DBus2Axi 返回给 LSU
   DBus.ready = 1
   DBus.rdata = 0x00000001
```

## 总结

CoralNPU 的总线系统采用了混合架构，结合了 AXI4 和 TileLink-UL 两种协议的优势：



- **AXI4** 用于外部接口和高性能传输，保证了与业界标准的兼容性
- **TileLink-UL** 用于片上互连，提供了简洁高效的内部通信
- **Fabric** 提供了灵活的互连结构，支持多主多从连接
- **转换器**（IBus2Axi、DBus2Axi、TLUL2Axi）实现了不同协议之间的无缝转换



这种设计既保证了性能和可靠性，又提供了良好的可扩展性和灵活性，适合嵌入式 SoC 的需求。

### 关键概念回顾

**AXI4 协议**：


- 五个独立通道（AW, W, B, AR, R）
- 基于 valid/ready 握手
- 支持突发传输和乱序执行



**TileLink-UL 协议**：


- 两个通道（A, D）
- 使用操作码区分读写
- 支持完整性检查



**IBus**：


- 只读接口，专门用于取指令
- 简单的 valid/ready 握手
- 自动地址对齐



**DBus**：


- 读写接口，用于数据访存
- 支持不同大小的访问
- 提供故障检测



**Fabric**：


- 片上互连结构
- 支持仲裁和路由
- 灵活的地址映射



### 进一步学习



- 阅读 AXI4 规范：ARM IHI 0022E
- 阅读 TileLink 规范：SiFive TileLink Specification
- 研究 OpenTitan 项目的总线实现
- 分析 CoralNPU 的仿真波形，观察总线时序



