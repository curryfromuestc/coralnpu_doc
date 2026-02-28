# 第十六章：API 参考

本章提供 CoralNPU 的完整 API 参考，包括硬件接口、软件 API、CSR 寄存器和外设寄存器映射。

## 16.1 硬件 API

### 16.1.1 顶层模块接口

CoralNPU 的顶层模块提供以下主要接口：

#### SCore 标量核心接口

标量核心（SCore）是 CoralNPU 的主要处理单元，提供以下 I/O 接口：

| 接口名称 | 方向 | 位宽 | 描述 |
|---------|------|------|------|
| `csr` | I/O | - | CSR 输入输出接口 |
| `halted` | Output | 1 | 核心停止信号 |
| `fault` | Output | 1 | 故障信号 |
| `wfi` | Output | 1 | 等待中断信号 |
| `irq` | Input | 1 | 中断请求信号 |
| `ibus` | Master | - | 指令总线接口 |
| `dbus` | Master | - | 数据总线接口 |
| `ebus` | Master | - | 扩展总线接口 |
| `iflush` | Output | - | 指令缓存刷新接口 |
| `dflush` | Output | - | 数据缓存刷新接口 |
| `slog` | Output | - | 系统日志接口 |
| `debug` | Output | - | 调试接口 |

#### 指令总线接口 (IBusIO)

指令总线用于从内存获取指令：

| 信号名称 | 方向 | 位宽 | 描述 |
|---------|------|------|------|
| `valid` | Output | 1 | 请求有效信号 |
| `ready` | Input | 1 | 准备就绪信号 |
| `addr` | Output | 32 | 取指地址 |
| `rdata` | Input | 256 | 读取的指令数据 |
| `fault` | Input | - | 故障信息 |

**说明**：
- `fetchDataBits` 默认为 256 位，可以同时获取 8 条 32 位指令
- 当 `valid` 为高时，`addr` 必须保持不变直到 `ready` 信号触发
- `rdata` 在 `ready` 信号触发后的下一个周期有效

#### 数据总线接口 (DBusIO)

数据总线用于加载和存储操作：

| 信号名称 | 方向 | 位宽 | 描述 |
|---------|------|------|------|
| `valid` | Output | 1 | 请求有效信号 |
| `ready` | Input | 1 | 准备就绪信号 |
| `write` | Output | 1 | 写操作标志 |
| `pc` | Output | 32 | 程序计数器 |
| `addr` | Output | 32 | 访问地址 |
| `adrx` | Output | 32 | 扩展地址 |
| `size` | Output | 4 | 传输大小（字节数的对数） |
| `wdata` | Output | 256 | 写数据 |
| `wmask` | Output | 32 | 写掩码（字节粒度） |
| `rdata` | Input | 256 | 读数据 |

**说明**：
- `lsuDataBits` 默认为 256 位，支持最大 32 字节的传输
- `size` 编码：0=1字节，1=2字节，2=4字节，3=8字节，4=16字节，5=32字节
- `wmask` 的每一位对应一个字节，1 表示该字节需要写入

#### CSR 输入输出接口 (CsrInOutIO)

CSR 接口用于外部访问和控制核心状态：

| 信号名称 | 方向 | 位宽 | 描述 |
|---------|------|------|------|
| `in.value[0..12]` | Input | 32×13 | CSR 输入值数组 |
| `out.value[0..8]` | Output | 32×9 | CSR 输出值数组 |

**CSR 输出映射**：

| 索引 | CSR 寄存器 | 描述 |
|------|-----------|------|
| 0 | - | 输入值回传 |
| 1 | `mepc` | 机器异常程序计数器 |
| 2 | `mtval` | 机器陷阱值 |
| 3 | `mcause` | 机器异常原因 |
| 4 | `mcycle[31:0]` | 周期计数器低 32 位 |
| 5 | `mcycle[63:32]` | 周期计数器高 32 位 |
| 6 | `minstret[31:0]` | 指令退休计数器低 32 位 |
| 7 | `minstret[63:32]` | 指令退休计数器高 32 位 |
| 8 | `mcontext0` | 机器上下文寄存器 0 |

### 16.1.2 端口定义

#### 取指单元端口 (FetchIO)

取指单元为每个指令通道提供解耦的指令流：

```scala
class FetchInstruction {
  addr: UInt(32.W)      // 指令地址
  inst: UInt(32.W)      // 指令编码
  brchFwd: Bool         // 分支前向标志
}
```

| 端口 | 类型 | 描述 |
|------|------|------|
| `lanes[0..3]` | Decoupled[FetchInstruction] | 4 个指令通道 |

**说明**：
- CoralNPU 支持 4 个指令通道（`instructionLanes = 4`）
- 每个通道使用 Decoupled 接口，支持反压
- `brchFwd` 用于分支预测和前向传播

#### 寄存器文件端口 (RegfileIO)

标量寄存器文件提供多端口读写访问：

**读端口**：
- 8 个读端口，每个端口包含：
  - `addr`: UInt(5.W) - 寄存器地址（0-31）
  - `data`: UInt(32.W) - 读取的数据

**写端口**：
- 6 个写端口（4 个指令通道 + 2 个额外端口）
- 每个端口包含：
  - `valid`: Bool - 写使能
  - `addr`: UInt(5.W) - 寄存器地址
  - `data`: UInt(32.W) - 写入的数据

**说明**：
- 寄存器 x0 硬连线为 0，写入无效
- 支持写后读转发（write-after-read forwarding）
- 多个写端口同时写入同一寄存器会触发断言错误

#### 浮点寄存器文件端口 (FRegfileIO)

浮点寄存器文件（当 `enableFloat = true` 时）：

**读端口**：
- 4 个读端口，每个端口包含：
  - `valid`: Bool - 读使能
  - `addr`: UInt(5.W) - 寄存器地址（0-31）
  - `data`: Fp32 - 读取的浮点数据

**写端口**：
- 2 个写端口，每个端口包含：
  - `valid`: Bool - 写使能
  - `addr`: UInt(5.W) - 寄存器地址
  - `data`: Fp32 - 写入的浮点数据

**说明**：
- 浮点寄存器为 32 位单精度（IEEE 754）
- 支持 RV32F 指令集扩展

### 16.1.3 参数配置

CoralNPU 的硬件参数通过 `Parameters` 类配置：

#### 核心参数

| 参数名称 | 类型 | 默认值 | 描述 |
|---------|------|--------|------|
| `programCounterBits` | Int | 32 | 程序计数器位宽 |
| `instructionBits` | Int | 32 | 指令位宽 |
| `instructionLanes` | Int | 4 | 指令通道数 |
| `hartId` | Int | 0 | 硬件线程 ID |

#### 功能使能参数

| 参数名称 | 类型 | 默认值 | 描述 |
|---------|------|--------|------|
| `enableVerification` | Boolean | false | 使能验证逻辑 |
| `enableRvv` | Boolean | false | 使能 RISC-V 向量扩展 |
| `enableFloat` | Boolean | false | 使能浮点运算 |
| `enableDebug` | Boolean | false | 使能调试模块 |
| `enableFetchL0` | Boolean | true | 使能 L0 指令缓存 |

#### 向量扩展参数

| 参数名称 | 类型 | 默认值 | 描述 |
|---------|------|--------|------|
| `rvvVlen` | Int | 128 | 向量寄存器位宽 |
| `rvvVlenb` | Int | 16 | 向量寄存器字节数（VLEN/8） |
| `rvvRegCount` | Int | 32 | 向量寄存器数量 |

#### 缓存参数

| 参数名称 | 类型 | 默认值 | 描述 |
|---------|------|--------|------|
| `fetchCacheBytes` | Int | 1024 | L0 指令缓存大小（字节） |
| `l1islots` | Int | 256 | L1 指令缓存槽数 |
| `l1iassoc` | Int | 4 | L1 指令缓存关联度 |
| `l1dslots` | Int | 256 | L1 数据缓存槽数（×2 banks） |

#### 总线参数

| 参数名称 | 类型 | 默认值 | 描述 |
|---------|------|--------|------|
| `fetchAddrBits` | Int | 32 | 取指地址位宽 |
| `fetchDataBits` | Int | 256 | 取指数据位宽 |
| `lsuAddrBits` | Int | 32 | LSU 地址位宽 |
| `lsuDataBits` | Int | 256 | LSU 数据位宽 |

#### AXI 接口参数

| 参数名称 | 类型 | 默认值 | 描述 |
|---------|------|--------|------|
| `axi0IdBits` | Int | 4 | AXI0（指令）ID 位宽 |
| `axi0AddrBits` | Int | 32 | AXI0 地址位宽 |
| `axi0DataBits` | Int | 256 | AXI0 数据位宽 |
| `axi1IdBits` | Int | 4 | AXI1（数据）ID 位宽 |
| `axi1AddrBits` | Int | 32 | AXI1 地址位宽 |
| `axi1DataBits` | Int | 256 | AXI1 数据位宽 |
| `axi2IdBits` | Int | 6 | AXI2（TCM）ID 位宽 |
| `axi2AddrBits` | Int | 32 | AXI2 地址位宽 |
| `axi2DataBits` | Int | 256 | AXI2 数据位宽 |

#### 内存区域参数

| 参数名称 | 类型 | 默认值 | 描述 |
|---------|------|--------|------|
| `itcmSizeKBytes` | Int | 8 | 指令紧耦合内存大小（KB） |
| `dtcmSizeKBytes` | Int | 32 | 数据紧耦合内存大小（KB） |

**内存映射（默认配置）**：

| 区域 | 起始地址 | 大小 | 类型 |
|------|---------|------|------|
| ITCM | 0x0000000 | 8 KB | 指令内存 |
| DTCM | 0x0010000 | 32 KB | 数据内存 |
| CSR | 0x0030000 | 4 KB | 外设寄存器 |

**内存映射（highmem 配置）**：

| 区域 | 起始地址 | 大小 | 类型 |
|------|---------|------|------|
| ITCM | 0x00000000 | 最大 1 MB | 指令内存 |
| DTCM | 0x00100000 | 最大 1 MB | 数据内存 |
| CSR | 0x00200000 | 4 KB | 外设寄存器 |

#### 退休缓冲区参数

| 参数名称 | 类型 | 默认值 | 描述 |
|---------|------|--------|------|
| `retirementBufferSize` | Int | 8 | 退休缓冲区大小 |
| `floatRegfileBaseAddr` | Int | 32 | 浮点寄存器基地址 |
| `rvvRegfileBaseAddr` | Int | 64 | 向量寄存器基地址 |


## 16.2 软件 API

### 16.2.1 系统调用

CoralNPU 支持标准的 RISC-V 系统调用接口。系统调用通过 `ecall` 指令触发。

#### 系统调用约定

| 寄存器 | 用途 |
|--------|------|
| `a7` (x17) | 系统调用号 |
| `a0` (x10) | 参数 1 / 返回值 |
| `a1` (x11) | 参数 2 / 错误码 |
| `a2` (x12) | 参数 3 |
| `a3` (x13) | 参数 4 |
| `a4` (x14) | 参数 5 |
| `a5` (x15) | 参数 6 |

#### 常用系统调用

| 系统调用号 | 名称 | 参数 | 返回值 | 描述 |
|-----------|------|------|--------|------|
| 93 | `exit` | a0: 退出码 | - | 终止程序执行 |
| 64 | `write` | a0: fd, a1: buf, a2: count | 写入字节数 | 写入文件描述符 |
| 63 | `read` | a0: fd, a1: buf, a2: count | 读取字节数 | 从文件描述符读取 |
| 214 | `brk` | a0: addr | 新的 break 地址 | 调整数据段大小 |
| 160 | `uname` | a0: buf | 0 成功，-1 失败 | 获取系统信息 |

**示例代码**：

```c
// 退出程序
static inline void sys_exit(int code) {
    register long a0 asm("a0") = code;
    register long a7 asm("a7") = 93;
    asm volatile("ecall" : : "r"(a0), "r"(a7));
}

// 写入数据
static inline long sys_write(int fd, const void *buf, size_t count) {
    register long a0 asm("a0") = fd;
    register long a1 asm("a1") = (long)buf;
    register long a2 asm("a2") = count;
    register long a7 asm("a7") = 64;
    register long ret asm("a0");
    asm volatile("ecall" : "=r"(ret) : "r"(a0), "r"(a1), "r"(a2), "r"(a7));
    return ret;
}
```

### 16.2.2 库函数

#### CSR 访问函数

CoralNPU 提供内联函数用于访问 CSR 寄存器：

```c
// 读取 CSR
#define csr_read(csr)                                   \
({                                                      \
    unsigned long __v;                                  \
    asm volatile ("csrr %0, " #csr : "=r" (__v));      \
    __v;                                                \
})

// 写入 CSR
#define csr_write(csr, val)                             \
({                                                      \
    unsigned long __v = (unsigned long)(val);           \
    asm volatile ("csrw " #csr ", %0" :: "r" (__v));   \
})

// 设置 CSR 位
#define csr_set(csr, val)                               \
({                                                      \
    unsigned long __v = (unsigned long)(val);           \
    asm volatile ("csrs " #csr ", %0" :: "r" (__v));   \
})

// 清除 CSR 位
#define csr_clear(csr, val)                             \
({                                                      \
    unsigned long __v = (unsigned long)(val);           \
    asm volatile ("csrc " #csr ", %0" :: "r" (__v));   \
})
```

**使用示例**：

```c
// 读取周期计数器
uint64_t get_cycles(void) {
    uint32_t lo = csr_read(mcycle);
    uint32_t hi = csr_read(mcycleh);
    return ((uint64_t)hi << 32) | lo;
}

// 设置中断使能
void enable_interrupts(void) {
    csr_set(mstatus, 0x8);  // 设置 MIE 位
}

// 设置异常处理入口
void set_trap_vector(void *handler) {
    csr_write(mtvec, (unsigned long)handler);
}
```

#### 内存屏障函数

```c
// 内存屏障
static inline void memory_barrier(void) {
    asm volatile("fence" ::: "memory");
}

// 指令屏障
static inline void instruction_barrier(void) {
    asm volatile("fence.i" ::: "memory");
}

// 数据同步屏障
static inline void data_sync_barrier(void) {
    asm volatile("fence rw, rw" ::: "memory");
}
```

#### 原子操作函数

```c
// 原子加载
static inline uint32_t atomic_load(volatile uint32_t *addr) {
    uint32_t val;
    asm volatile("lr.w %0, (%1)" : "=r"(val) : "r"(addr) : "memory");
    return val;
}

// 原子存储
static inline int atomic_store(volatile uint32_t *addr, uint32_t val) {
    int result;
    asm volatile("sc.w %0, %2, (%1)" 
                 : "=r"(result) : "r"(addr), "r"(val) : "memory");
    return result;
}

// 原子交换
static inline uint32_t atomic_swap(volatile uint32_t *addr, uint32_t val) {
    uint32_t old;
    asm volatile("amoswap.w %0, %2, (%1)" 
                 : "=r"(old) : "r"(addr), "r"(val) : "memory");
    return old;
}

// 原子加法
static inline uint32_t atomic_add(volatile uint32_t *addr, uint32_t val) {
    uint32_t old;
    asm volatile("amoadd.w %0, %2, (%1)" 
                 : "=r"(old) : "r"(addr), "r"(val) : "memory");
    return old;
}
```

### 16.2.3 向量内置函数

当 `enableRvv = true` 时，CoralNPU 支持 RISC-V 向量扩展（RVV 1.0）。

#### 向量配置函数

```c
// 设置向量长度和类型
// vl: 向量长度（元素数量）
// vtype: 向量类型配置
// 返回实际设置的向量长度
static inline size_t vsetvl(size_t vl, size_t vtype) {
    size_t result;
    asm volatile("vsetvl %0, %1, %2" 
                 : "=r"(result) : "r"(vl), "r"(vtype));
    return result;
}

// 设置向量长度（使用立即数类型）
static inline size_t vsetvli(size_t vl, size_t vtypei) {
    size_t result;
    asm volatile("vsetvli %0, %1, %2" 
                 : "=r"(result) : "r"(vl), "i"(vtypei));
    return result;
}
```

**向量类型编码（vtype）**：

| 位域 | 名称 | 描述 |
|------|------|------|
| [2:0] | `vsew` | 元素宽度：0=8位，1=16位，2=32位，3=64位 |
| [5:3] | `vlmul` | 长度倍数：0=1，1=2，2=4，3=8，5=1/8，6=1/4，7=1/2 |
| [6] | `vta` | 尾部不可知：0=保留，1=不可知 |
| [7] | `vma` | 掩码不可知：0=保留，1=不可知 |

**常用向量类型宏**：

```c
#define VTYPE_SEW8   0  // 8位元素
#define VTYPE_SEW16  1  // 16位元素
#define VTYPE_SEW32  2  // 32位元素
#define VTYPE_SEW64  3  // 64位元素

#define VTYPE_LMUL1  (0 << 3)  // LMUL=1
#define VTYPE_LMUL2  (1 << 3)  // LMUL=2
#define VTYPE_LMUL4  (2 << 3)  // LMUL=4
#define VTYPE_LMUL8  (3 << 3)  // LMUL=8

#define VTYPE_VTA    (1 << 6)  // 尾部不可知
#define VTYPE_VMA    (1 << 7)  // 掩码不可知
```

#### 向量加载/存储函数

```c
// 向量单位步长加载（8位）
static inline void vle8_v(void *vd, const uint8_t *rs1, size_t vl) {
    asm volatile("vle8.v %0, (%1)" :: "vr"(vd), "r"(rs1));
}

// 向量单位步长加载（32位）
static inline void vle32_v(void *vd, const uint32_t *rs1, size_t vl) {
    asm volatile("vle32.v %0, (%1)" :: "vr"(vd), "r"(rs1));
}

// 向量单位步长存储（8位）
static inline void vse8_v(uint8_t *rs1, const void *vs3, size_t vl) {
    asm volatile("vse8.v %1, (%0)" :: "r"(rs1), "vr"(vs3));
}

// 向量单位步长存储（32位）
static inline void vse32_v(uint32_t *rs1, const void *vs3, size_t vl) {
    asm volatile("vse32.v %1, (%0)" :: "r"(rs1), "vr"(vs3));
}

// 向量跨步加载（32位）
static inline void vlse32_v(void *vd, const uint32_t *rs1, long rs2, size_t vl) {
    asm volatile("vlse32.v %0, (%1), %2" :: "vr"(vd), "r"(rs1), "r"(rs2));
}

// 向量跨步存储（32位）
static inline void vsse32_v(uint32_t *rs1, long rs2, const void *vs3, size_t vl) {
    asm volatile("vsse32.v %2, (%0), %1" :: "r"(rs1), "r"(rs2), "vr"(vs3));
}
```

#### 向量算术函数

```c
// 向量加法（向量-向量）
static inline void vadd_vv(void *vd, const void *vs2, const void *vs1) {
    asm volatile("vadd.vv %0, %1, %2" :: "vr"(vd), "vr"(vs2), "vr"(vs1));
}

// 向量加法（向量-标量）
static inline void vadd_vx(void *vd, const void *vs2, long rs1) {
    asm volatile("vadd.vx %0, %1, %2" :: "vr"(vd), "vr"(vs2), "r"(rs1));
}

// 向量减法（向量-向量）
static inline void vsub_vv(void *vd, const void *vs2, const void *vs1) {
    asm volatile("vsub.vv %0, %1, %2" :: "vr"(vd), "vr"(vs2), "vr"(vs1));
}

// 向量乘法（向量-向量）
static inline void vmul_vv(void *vd, const void *vs2, const void *vs1) {
    asm volatile("vmul.vv %0, %1, %2" :: "vr"(vd), "vr"(vs2), "vr"(vs1));
}

// 向量乘法（向量-标量）
static inline void vmul_vx(void *vd, const void *vs2, long rs1) {
    asm volatile("vmul.vx %0, %1, %2" :: "vr"(vd), "vr"(vs2), "r"(rs1));
}

// 向量除法（向量-向量）
static inline void vdiv_vv(void *vd, const void *vs2, const void *vs1) {
    asm volatile("vdiv.vv %0, %1, %2" :: "vr"(vd), "vr"(vs2), "vr"(vs1));
}
```

#### 向量归约函数

```c
// 向量求和归约
static inline void vredsum_vs(void *vd, const void *vs2, const void *vs1) {
    asm volatile("vredsum.vs %0, %1, %2" :: "vr"(vd), "vr"(vs2), "vr"(vs1));
}

// 向量最大值归约
static inline void vredmax_vs(void *vd, const void *vs2, const void *vs1) {
    asm volatile("vredmax.vs %0, %1, %2" :: "vr"(vd), "vr"(vs2), "vr"(vs1));
}

// 向量最小值归约
static inline void vredmin_vs(void *vd, const void *vs2, const void *vs1) {
    asm volatile("vredmin.vs %0, %1, %2" :: "vr"(vd), "vr"(vs2), "vr"(vs1));
}
```

#### 向量掩码函数

```c
// 向量比较（等于）
static inline void vmseq_vv(void *vd, const void *vs2, const void *vs1) {
    asm volatile("vmseq.vv %0, %1, %2" :: "vr"(vd), "vr"(vs2), "vr"(vs1));
}

// 向量比较（不等于）
static inline void vmsne_vv(void *vd, const void *vs2, const void *vs1) {
    asm volatile("vmsne.vv %0, %1, %2" :: "vr"(vd), "vr"(vs2), "vr"(vs1));
}

// 向量比较（小于）
static inline void vmslt_vv(void *vd, const void *vs2, const void *vs1) {
    asm volatile("vmslt.vv %0, %1, %2" :: "vr"(vd), "vr"(vs2), "vr"(vs1));
}

// 向量比较（小于等于）
static inline void vmsle_vv(void *vd, const void *vs2, const void *vs1) {
    asm volatile("vmsle.vv %0, %1, %2" :: "vr"(vd), "vr"(vs2), "vr"(vs1));
}
```

**向量操作示例**：

```c
// 向量加法示例
void vector_add(uint32_t *dst, const uint32_t *src1, 
                const uint32_t *src2, size_t n) {
    size_t vl;
    // 配置向量：32位元素，LMUL=1
    vsetvli(0, VTYPE_SEW32 | VTYPE_LMUL1);
    
    for (size_t i = 0; i < n; ) {
        vl = vsetvli(n - i, VTYPE_SEW32 | VTYPE_LMUL1);
        vle32_v(v0, &src1[i], vl);
        vle32_v(v1, &src2[i], vl);
        vadd_vv(v2, v0, v1);
        vse32_v(&dst[i], v2, vl);
        i += vl;
    }
}
```


## 16.3 CSR 参考

### 16.3.1 CSR 寄存器列表

CoralNPU 实现了以下 CSR 寄存器：

#### 浮点 CSR（当 enableFloat = true）

| 地址 | 名称 | 权限 | 描述 |
|------|------|------|------|
| 0x001 | `fflags` | RW | 浮点异常标志 |
| 0x002 | `frm` | RW | 浮点舍入模式 |
| 0x003 | `fcsr` | RW | 浮点控制和状态寄存器 |

#### 向量 CSR（当 enableRvv = true）

| 地址 | 名称 | 权限 | 描述 |
|------|------|------|------|
| 0x008 | `vstart` | RW | 向量起始索引 |
| 0x009 | `vxsat` | RW | 向量定点饱和标志 |
| 0x00A | `vxrm` | RW | 向量定点舍入模式 |
| 0xC20 | `vl` | RO | 向量长度 |
| 0xC21 | `vtype` | RO | 向量类型 |
| 0xC22 | `vlenb` | RO | 向量长度（字节）= 16 |

#### 机器模式 CSR

| 地址 | 名称 | 权限 | 描述 |
|------|------|------|------|
| 0x300 | `mstatus` | RW | 机器状态寄存器 |
| 0x301 | `misa` | RO | ISA 和扩展 |
| 0x304 | `mie` | RW | 机器中断使能 |
| 0x305 | `mtvec` | RW | 机器陷阱向量基地址 |
| 0x340 | `mscratch` | RW | 机器模式临时寄存器 |
| 0x341 | `mepc` | RW | 机器异常程序计数器 |
| 0x342 | `mcause` | RW | 机器异常原因 |
| 0x343 | `mtval` | RW | 机器陷阱值 |

#### 性能计数器 CSR

| 地址 | 名称 | 权限 | 描述 |
|------|------|------|------|
| 0xB00 | `mcycle` | RW | 机器周期计数器（低 32 位） |
| 0xB02 | `minstret` | RW | 机器指令退休计数器（低 32 位） |
| 0xB80 | `mcycleh` | RW | 机器周期计数器（高 32 位） |
| 0xB82 | `minstreth` | RW | 机器指令退休计数器（高 32 位） |

#### 机器信息 CSR

| 地址 | 名称 | 权限 | 描述 |
|------|------|------|------|
| 0xF11 | `mvendorid` | RO | 厂商 ID = 0x426（Google） |
| 0xF12 | `marchid` | RO | 架构 ID = 0 |
| 0xF13 | `mimpid` | RO | 实现 ID = 0 |
| 0xF14 | `mhartid` | RO | 硬件线程 ID |

#### 调试 CSR（当 enableDebug = true）

| 地址 | 名称 | 权限 | 描述 |
|------|------|------|------|
| 0x7A0 | `tselect` | RW | 触发器选择 |
| 0x7A1 | `tdata1` | RW | 触发器数据 1 |
| 0x7A2 | `tdata2` | RW | 触发器数据 2 |
| 0x7A4 | `tinfo` | RO | 触发器信息 = 0x01000040 |
| 0x7B0 | `dcsr` | RW | 调试控制和状态 |
| 0x7B1 | `dpc` | RW | 调试程序计数器 |
| 0x7B2 | `dscratch0` | RW | 调试临时寄存器 0 |
| 0x7B3 | `dscratch1` | RW | 调试临时寄存器 1 |

#### 自定义 CSR

| 地址 | 名称 | 权限 | 描述 |
|------|------|------|------|
| 0x7C0 | `mcontext0` | RW | 机器上下文寄存器 0 |
| 0x7C1 | `mcontext1` | RW | 机器上下文寄存器 1 |
| 0x7C2 | `mcontext2` | RW | 机器上下文寄存器 2 |
| 0x7C3 | `mcontext3` | RW | 机器上下文寄存器 3 |
| 0x7C4 | `mcontext4` | RW | 机器上下文寄存器 4 |
| 0x7C5 | `mcontext5` | RW | 机器上下文寄存器 5 |
| 0x7C6 | `mcontext6` | RW | 机器上下文寄存器 6 |
| 0x7C7 | `mcontext7` | RW | 机器上下文寄存器 7 |
| 0x7E0 | `mpc` | RW | 机器程序计数器 |
| 0x7E1 | `msp` | RW | 机器栈指针 |
| 0xFC0 | `kisa` | RW | CoralNPU ISA 扩展 |
| 0xFC4 | `kscm0` | RO | SCM 版本信息 [31:0] |
| 0xFC8 | `kscm1` | RO | SCM 版本信息 [63:32] |
| 0xFCC | `kscm2` | RO | SCM 版本信息 [95:64] |
| 0xFD0 | `kscm3` | RO | SCM 版本信息 [127:96] |
| 0xFD4 | `kscm4` | RO | SCM 版本信息 [159:128] |

### 16.3.2 CSR 位域定义

#### mstatus（0x300）- 机器状态寄存器

| 位 | 名称 | 访问 | 复位值 | 描述 |
|----|------|------|--------|------|
| [31:15] | - | RO | 0 | 保留 |
| [14:13] | `fs` | RO | 1（浮点）/0 | 浮点状态：0=Off, 1=Initial, 2=Clean, 3=Dirty |
| [12:11] | `mpp` | RW | 0 | 机器先前特权模式 |
| [10:9] | `vs` | RO | 1（向量）/0 | 向量状态：0=Off, 1=Initial, 2=Clean, 3=Dirty |
| [8:0] | - | RO | 0 | 保留 |

**说明**：
- `fs` 和 `vs` 字段指示浮点和向量单元的状态
- `mpp` 保存陷阱前的特权模式，用于 `mret` 指令恢复

#### misa（0x301）- ISA 和扩展

| 位 | 名称 | 访问 | 值 | 描述 |
|----|------|------|-----|------|
| [31:30] | `MXL` | RO | 1 | XLEN：1=32位，2=64位 |
| [29:26] | - | RO | 0 | 保留 |
| [25] | `Z` | RO | 0 | 保留 |
| [24] | `Y` | RO | 0 | 保留 |
| [23] | `X` | RO | 1 | 非标准扩展 |
| [22] | `W` | RO | 0 | 保留 |
| [21] | `V` | RO | enableRvv | 向量扩展 |
| [20] | `U` | RO | 0 | 用户模式 |
| [19:13] | - | RO | 0 | 保留 |
| [12] | `M` | RO | 1 | 整数乘除法扩展 |
| [11:9] | - | RO | 0 | 保留 |
| [8] | `I` | RO | 1 | RV32I 基础整数指令集 |
| [7:6] | - | RO | 0 | 保留 |
| [5] | `F` | RO | enableFloat | 单精度浮点扩展 |
| [4:0] | - | RO | 0 | 保留 |

**默认值**：
- 无浮点无向量：0x40001100
- 有浮点：0x40001120
- 有向量：0x40201100
- 有浮点和向量：0x40201120

#### mie（0x304）- 机器中断使能

| 位 | 名称 | 访问 | 复位值 | 描述 |
|----|------|------|--------|------|
| [31:1] | - | RO | 0 | 保留 |
| [0] | `MIE` | RW | 0 | 机器模式中断使能 |

#### mtvec（0x305）- 机器陷阱向量基地址

| 位 | 名称 | 访问 | 复位值 | 描述 |
|----|------|------|--------|------|
| [31:2] | `BASE` | RW | 0 | 陷阱处理程序基地址（4字节对齐） |
| [1:0] | `MODE` | RW | 0 | 向量模式：0=Direct, 1=Vectored |

**说明**：
- Direct 模式：所有陷阱跳转到 BASE
- Vectored 模式：异步中断跳转到 BASE + 4×cause

#### mepc（0x341）- 机器异常程序计数器

| 位 | 名称 | 访问 | 复位值 | 描述 |
|----|------|------|--------|------|
| [31:0] | `MEPC` | RW | 0 | 异常返回地址 |

**说明**：
- 保存发生异常时的 PC 值
- `mret` 指令将 PC 恢复为 `mepc` 的值

#### mcause（0x342）- 机器异常原因

| 位 | 名称 | 访问 | 复位值 | 描述 |
|----|------|------|--------|------|
| [31] | `Interrupt` | RW | 0 | 1=中断，0=异常 |
| [30:0] | `Exception Code` | RW | 0 | 异常/中断编码 |

**异常编码**：

| 编码 | 名称 | 描述 |
|------|------|------|
| 0 | Instruction address misaligned | 指令地址未对齐 |
| 1 | Instruction access fault | 指令访问故障 |
| 2 | Illegal instruction | 非法指令 |
| 3 | Breakpoint | 断点 |
| 4 | Load address misaligned | 加载地址未对齐 |
| 5 | Load access fault | 加载访问故障 |
| 6 | Store/AMO address misaligned | 存储地址未对齐 |
| 7 | Store/AMO access fault | 存储访问故障 |
| 8 | Environment call from U-mode | 用户模式环境调用 |
| 11 | Environment call from M-mode | 机器模式环境调用 |

**中断编码**（Interrupt=1）：

| 编码 | 名称 | 描述 |
|------|------|------|
| 3 | Machine software interrupt | 机器软件中断 |
| 7 | Machine timer interrupt | 机器定时器中断 |
| 11 | Machine external interrupt | 机器外部中断 |

#### mtval（0x343）- 机器陷阱值

| 位 | 名称 | 访问 | 复位值 | 描述 |
|----|------|------|--------|------|
| [31:0] | `MTVAL` | RW | 0 | 异常相关信息 |

**说明**：
- 地址异常：保存出错的地址
- 非法指令：保存指令编码
- 其他异常：通常为 0

#### fflags（0x001）- 浮点异常标志

| 位 | 名称 | 访问 | 复位值 | 描述 |
|----|------|------|--------|------|
| [31:5] | - | RO | 0 | 保留 |
| [4] | `NV` | RW | 0 | 无效操作 |
| [3] | `DZ` | RW | 0 | 除零 |
| [2] | `OF` | RW | 0 | 溢出 |
| [1] | `UF` | RW | 0 | 下溢 |
| [0] | `NX` | RW | 0 | 不精确 |

#### frm（0x002）- 浮点舍入模式

| 位 | 名称 | 访问 | 复位值 | 描述 |
|----|------|------|--------|------|
| [31:3] | - | RO | 0 | 保留 |
| [2:0] | `FRM` | RW | 0 | 舍入模式 |

**舍入模式编码**：

| 值 | 名称 | 描述 |
|----|------|------|
| 0 | RNE | 舍入到最近偶数 |
| 1 | RTZ | 向零舍入 |
| 2 | RDN | 向下舍入（向负无穷） |
| 3 | RUP | 向上舍入（向正无穷） |
| 4 | RMM | 舍入到最近最大值 |

#### fcsr（0x003）- 浮点控制和状态

| 位 | 名称 | 访问 | 复位值 | 描述 |
|----|------|------|--------|------|
| [31:8] | - | RO | 0 | 保留 |
| [7:5] | `frm` | RW | 0 | 舍入模式 |
| [4:0] | `fflags` | RW | 0 | 异常标志 |

**说明**：`fcsr` 是 `frm` 和 `fflags` 的组合视图

#### vstart（0x008）- 向量起始索引

| 位 | 名称 | 访问 | 复位值 | 描述 |
|----|------|------|--------|------|
| [31:7] | - | RO | 0 | 保留 |
| [6:0] | `VSTART` | RW | 0 | 向量操作起始元素索引 |

**说明**：
- 用于向量指令的可恢复异常
- 正常情况下为 0
- 向量指令完成后自动清零

#### vxsat（0x009）- 向量定点饱和标志

| 位 | 名称 | 访问 | 复位值 | 描述 |
|----|------|------|--------|------|
| [31:1] | - | RO | 0 | 保留 |
| [0] | `VXSAT` | RW | 0 | 定点饱和标志 |

#### vxrm（0x00A）- 向量定点舍入模式

| 位 | 名称 | 访问 | 复位值 | 描述 |
|----|------|------|--------|------|
| [31:2] | - | RO | 0 | 保留 |
| [1:0] | `VXRM` | RW | 0 | 舍入模式 |

**舍入模式编码**：

| 值 | 名称 | 描述 |
|----|------|------|
| 0 | RNU | 向上舍入（向正无穷） |
| 1 | RNE | 舍入到最近偶数 |
| 2 | RDN | 向下舍入（向零） |
| 3 | ROD | 舍入到奇数 |

#### vl（0xC20）- 向量长度

| 位 | 名称 | 访问 | 复位值 | 描述 |
|----|------|------|--------|------|
| [31:7] | - | RO | 0 | 保留 |
| [6:0] | `VL` | RO | 0 | 当前向量长度（元素数量） |

**说明**：
- 只读寄存器，由 `vsetvl` 指令设置
- 最大值取决于 VLEN 和 LMUL

#### vtype（0xC21）- 向量类型

| 位 | 名称 | 访问 | 复位值 | 描述 |
|----|------|------|--------|------|
| [31] | `vill` | RO | 0 | 非法配置标志 |
| [30:8] | - | RO | 0 | 保留 |
| [7] | `vma` | RO | 0 | 掩码不可知 |
| [6] | `vta` | RO | 0 | 尾部不可知 |
| [5:3] | `vsew` | RO | 0 | 元素宽度 |
| [2:0] | `vlmul` | RO | 0 | 长度倍数 |

#### dcsr（0x7B0）- 调试控制和状态

| 位 | 名称 | 访问 | 复位值 | 描述 |
|----|------|------|--------|------|
| [31:28] | `xdebugver` | RO | 4 | 调试规范版本 |
| [27:16] | - | RO | 0 | 保留 |
| [15] | `ebreakm` | RW | 0 | M模式 ebreak 进入调试 |
| [14] | - | RO | 0 | 保留 |
| [13] | `ebreaks` | RW | 0 | S模式 ebreak 进入调试 |
| [12] | `ebreaku` | RW | 0 | U模式 ebreak 进入调试 |
| [11] | `stepie` | RW | 0 | 单步时中断使能 |
| [10] | `stopcount` | RW | 0 | 调试时停止计数器 |
| [9] | `stoptime` | RW | 0 | 调试时停止定时器 |
| [8:6] | `cause` | RO | 0 | 进入调试原因 |
| [5] | - | RO | 0 | 保留 |
| [4] | `mprven` | RW | 0 | 使用 mstatus.mprv |
| [3] | `nmip` | RO | 0 | 不可屏蔽中断挂起 |
| [2] | `step` | RW | 0 | 单步模式 |
| [1:0] | `prv` | RW | 3 | 调试前特权模式 |

**调试原因编码**：

| 值 | 名称 | 描述 |
|----|------|------|
| 1 | ebreak | ebreak 指令 |
| 2 | trigger | 触发器匹配 |
| 3 | haltreq | 外部调试请求 |
| 4 | step | 单步执行 |


## 16.4 寄存器映射

### 16.4.1 核心控制寄存器

核心控制寄存器通过 AXI 接口访问，基地址为外设区域起始地址。

#### 基本控制寄存器

| 偏移 | 名称 | 访问 | 复位值 | 描述 |
|------|------|------|--------|------|
| 0x000 | `RESET_REG` | RW | 0x3 | 复位和时钟门控控制 |
| 0x004 | `PC_START_REG` | RW | 0x0 | 启动程序计数器 |
| 0x008 | `STATUS_REG` | RO | 0x0 | 核心状态 |

#### RESET_REG（0x000）- 复位和时钟门控

| 位 | 名称 | 访问 | 复位值 | 描述 |
|----|------|------|--------|------|
| [31:2] | - | RO | 0 | 保留 |
| [1] | `CG` | RW | 1 | 时钟门控：1=门控，0=运行 |
| [0] | `RESET` | RW | 1 | 复位：1=复位，0=正常 |

**使用流程**：
1. 写入 0x3：核心处于复位和时钟门控状态
2. 写入 0x2：解除复位，保持时钟门控
3. 写入 0x0：解除时钟门控，核心开始运行

#### PC_START_REG（0x004）- 启动程序计数器

| 位 | 名称 | 访问 | 复位值 | 描述 |
|----|------|------|--------|------|
| [31:0] | `PC_START` | RW | 0 | 核心启动时的 PC 值 |

**说明**：
- 必须在解除复位前设置
- 通常设置为程序入口地址（如 0x0）

#### STATUS_REG（0x008）- 核心状态

| 位 | 名称 | 访问 | 复位值 | 描述 |
|----|------|------|--------|------|
| [31:2] | - | RO | 0 | 保留 |
| [1] | `FAULT` | RO | 0 | 故障状态：1=故障 |
| [0] | `HALTED` | RO | 0 | 停止状态：1=停止 |

### 16.4.2 CSR 镜像寄存器

CoralNPU 的内部 CSR 可以通过外设寄存器访问（只读）：

| 偏移 | CSR | 描述 |
|------|-----|------|
| 0x100 | - | 输入值回传 |
| 0x104 | `mepc` | 机器异常程序计数器 |
| 0x108 | `mtval` | 机器陷阱值 |
| 0x10C | `mcause` | 机器异常原因 |
| 0x110 | `mcycle[31:0]` | 周期计数器低 32 位 |
| 0x114 | `mcycle[63:32]` | 周期计数器高 32 位 |
| 0x118 | `minstret[31:0]` | 指令退休计数器低 32 位 |
| 0x11C | `minstret[63:32]` | 指令退休计数器高 32 位 |
| 0x120 | `mcontext0` | 机器上下文寄存器 0 |

**说明**：
- 这些寄存器提供对核心内部 CSR 的只读访问
- 用于外部监控和调试
- 不影响核心内部的 CSR 访问

### 16.4.3 调试模块寄存器（当 enableDebug = true）

调试模块提供外部调试接口：

| 偏移 | 名称 | 访问 | 描述 |
|------|------|------|------|
| 0x800 | `DBG_REQ_ADDR` | RW | 调试请求地址 |
| 0x804 | `DBG_REQ_DATA` | RW | 调试请求数据 |
| 0x808 | `DBG_REQ_OP` | RW | 调试请求操作 |
| 0x80C | `DBG_RSP_DATA` | RO | 调试响应数据 |
| 0x810 | `DBG_RSP_OP` | RO | 调试响应操作 |
| 0x814 | `DBG_STATUS` | RW | 调试状态 |

#### DBG_REQ_ADDR（0x800）- 调试请求地址

| 位 | 名称 | 访问 | 描述 |
|----|------|------|------|
| [31:0] | `ADDR` | RW | 寄存器地址或内存地址 |

#### DBG_REQ_DATA（0x804）- 调试请求数据

| 位 | 名称 | 访问 | 描述 |
|----|------|------|------|
| [31:0] | `DATA` | RW | 写入的数据 |

#### DBG_REQ_OP（0x808）- 调试请求操作

| 位 | 名称 | 访问 | 描述 |
|----|------|------|------|
| [31:0] | `OP` | RW | 操作类型 |

**操作类型编码**：

| 值 | 名称 | 描述 |
|----|------|------|
| 0 | NOP | 无操作 |
| 1 | CSR_READ | 读取 CSR |
| 2 | CSR_WRITE | 写入 CSR |
| 3 | REG_READ | 读取标量寄存器 |
| 4 | REG_WRITE | 写入标量寄存器 |
| 5 | FREG_READ | 读取浮点寄存器 |
| 6 | FREG_WRITE | 写入浮点寄存器 |

**说明**：
- 写入 `DBG_REQ_OP` 触发调试操作
- 操作完成后，结果在 `DBG_RSP_DATA` 中

#### DBG_RSP_DATA（0x80C）- 调试响应数据

| 位 | 名称 | 访问 | 描述 |
|----|------|------|------|
| [31:0] | `DATA` | RO | 读取的数据 |

#### DBG_RSP_OP（0x810）- 调试响应操作

| 位 | 名称 | 访问 | 描述 |
|----|------|------|------|
| [31:0] | `OP` | RO | 响应的操作类型 |

#### DBG_STATUS（0x814）- 调试状态

| 位 | 名称 | 访问 | 描述 |
|----|------|------|------|
| [31:2] | - | RO | 保留 |
| [1] | `RSP_VALID` | RO | 响应有效：1=有数据 |
| [0] | `REQ_READY` | RO | 请求就绪：1=可接受新请求 |

**使用流程**：
1. 等待 `REQ_READY = 1`
2. 写入 `DBG_REQ_ADDR` 和 `DBG_REQ_DATA`（如需要）
3. 写入 `DBG_REQ_OP` 触发操作
4. 等待 `RSP_VALID = 1`
5. 读取 `DBG_RSP_DATA` 和 `DBG_RSP_OP`
6. 写入 `DBG_STATUS` 清除响应

### 16.4.4 内存映射总览

#### 默认内存映射

```
0x00000000 ┌─────────────────────┐
           │                     │
           │   ITCM (8 KB)       │  指令紧耦合内存
           │                     │
0x00002000 ├─────────────────────┤
           │                     │
           │   (保留)            │
           │                     │
0x00010000 ├─────────────────────┤
           │                     │
           │   DTCM (32 KB)      │  数据紧耦合内存
           │                     │
0x00018000 ├─────────────────────┤
           │                     │
           │   (保留)            │
           │                     │
0x00030000 ├─────────────────────┤
           │                     │
           │   外设寄存器 (4 KB) │  CSR 和控制寄存器
           │                     │
0x00031000 └─────────────────────┘
```

#### Highmem 内存映射

```
0x00000000 ┌─────────────────────┐
           │                     │
           │   ITCM              │  指令紧耦合内存
           │   (最大 1 MB)       │  可配置大小
           │                     │
0x00100000 ├─────────────────────┤
           │                     │
           │   DTCM              │  数据紧耦合内存
           │   (最大 1 MB)       │  可配置大小
           │                     │
0x00200000 ├─────────────────────┤
           │                     │
           │   外设寄存器 (4 KB) │  CSR 和控制寄存器
           │                     │
0x00201000 └─────────────────────┘
```

### 16.4.5 访问方法

#### C 语言访问示例

```c
// 寄存器基地址
#define CORALNPU_BASE    0x00030000  // 默认配置
// #define CORALNPU_BASE 0x00200000  // Highmem 配置

// 寄存器偏移
#define REG_RESET        0x000
#define REG_PC_START     0x004
#define REG_STATUS       0x008
#define REG_CSR_BASE     0x100

// 寄存器访问宏
#define REG32(addr)      (*(volatile uint32_t *)(addr))
#define CORALNPU_REG(offset) REG32(CORALNPU_BASE + (offset))

// 启动核心
void coralnpu_start(uint32_t pc_start) {
    // 设置启动 PC
    CORALNPU_REG(REG_PC_START) = pc_start;
    
    // 解除复位，保持时钟门控
    CORALNPU_REG(REG_RESET) = 0x2;
    
    // 解除时钟门控，开始运行
    CORALNPU_REG(REG_RESET) = 0x0;
}

// 停止核心
void coralnpu_stop(void) {
    // 时钟门控
    CORALNPU_REG(REG_RESET) = 0x2;
    
    // 复位
    CORALNPU_REG(REG_RESET) = 0x3;
}

// 检查核心状态
int coralnpu_is_halted(void) {
    return CORALNPU_REG(REG_STATUS) & 0x1;
}

int coralnpu_is_fault(void) {
    return (CORALNPU_REG(REG_STATUS) >> 1) & 0x1;
}

// 读取性能计数器
uint64_t coralnpu_get_cycles(void) {
    uint32_t lo = CORALNPU_REG(REG_CSR_BASE + 0x10);
    uint32_t hi = CORALNPU_REG(REG_CSR_BASE + 0x14);
    return ((uint64_t)hi << 32) | lo;
}

uint64_t coralnpu_get_instret(void) {
    uint32_t lo = CORALNPU_REG(REG_CSR_BASE + 0x18);
    uint32_t hi = CORALNPU_REG(REG_CSR_BASE + 0x1C);
    return ((uint64_t)hi << 32) | lo;
}
```

#### 调试接口访问示例

```c
// 调试寄存器偏移
#define REG_DBG_REQ_ADDR  0x800
#define REG_DBG_REQ_DATA  0x804
#define REG_DBG_REQ_OP    0x808
#define REG_DBG_RSP_DATA  0x80C
#define REG_DBG_RSP_OP    0x810
#define REG_DBG_STATUS    0x814

// 调试操作类型
#define DBG_OP_NOP        0
#define DBG_OP_CSR_READ   1
#define DBG_OP_CSR_WRITE  2
#define DBG_OP_REG_READ   3
#define DBG_OP_REG_WRITE  4

// 等待调试请求就绪
void debug_wait_ready(void) {
    while (!(CORALNPU_REG(REG_DBG_STATUS) & 0x1));
}

// 等待调试响应
void debug_wait_response(void) {
    while (!(CORALNPU_REG(REG_DBG_STATUS) & 0x2));
}

// 清除调试响应
void debug_clear_response(void) {
    CORALNPU_REG(REG_DBG_STATUS) = 0;
}

// 读取 CSR
uint32_t debug_read_csr(uint32_t csr_addr) {
    debug_wait_ready();
    CORALNPU_REG(REG_DBG_REQ_ADDR) = csr_addr;
    CORALNPU_REG(REG_DBG_REQ_OP) = DBG_OP_CSR_READ;
    debug_wait_response();
    uint32_t data = CORALNPU_REG(REG_DBG_RSP_DATA);
    debug_clear_response();
    return data;
}

// 写入 CSR
void debug_write_csr(uint32_t csr_addr, uint32_t value) {
    debug_wait_ready();
    CORALNPU_REG(REG_DBG_REQ_ADDR) = csr_addr;
    CORALNPU_REG(REG_DBG_REQ_DATA) = value;
    CORALNPU_REG(REG_DBG_REQ_OP) = DBG_OP_CSR_WRITE;
    debug_wait_response();
    debug_clear_response();
}

// 读取标量寄存器
uint32_t debug_read_reg(uint32_t reg_idx) {
    debug_wait_ready();
    CORALNPU_REG(REG_DBG_REQ_ADDR) = reg_idx;
    CORALNPU_REG(REG_DBG_REQ_OP) = DBG_OP_REG_READ;
    debug_wait_response();
    uint32_t data = CORALNPU_REG(REG_DBG_RSP_DATA);
    debug_clear_response();
    return data;
}

// 写入标量寄存器
void debug_write_reg(uint32_t reg_idx, uint32_t value) {
    debug_wait_ready();
    CORALNPU_REG(REG_DBG_REQ_ADDR) = reg_idx;
    CORALNPU_REG(REG_DBG_REQ_DATA) = value;
    CORALNPU_REG(REG_DBG_REQ_OP) = DBG_OP_REG_WRITE;
    debug_wait_response();
    debug_clear_response();
}
```

## 16.5 参数头文件生成

CoralNPU 提供 `EmitParametersHeader` 函数，可以将硬件参数导出为 C 头文件：

```c
// 自动生成的 coralnpu_parameters.h
#ifndef CORALNPU_PARAMETERS_H_
#define CORALNPU_PARAMETERS_H_

#include <stdbool.h>

// 核心参数
#define KP_programCounterBits 32
#define KP_instructionBits 32
#define KP_instructionLanes 4
#define KP_hartId 0

// 功能使能
#define KP_enableVerification false
#define KP_enableRvv false
#define KP_enableFloat false
#define KP_enableDebug false
#define KP_enableFetchL0 true

// 向量参数
#define KP_rvvVlen 128

// 缓存参数
#define KP_fetchCacheBytes 1024
#define KP_l1islots 256
#define KP_l1iassoc 4
#define KP_l1dslots 256

// 总线参数
#define KP_fetchAddrBits 32
#define KP_fetchDataBits 256
#define KP_lsuAddrBits 32
#define KP_lsuDataBits 256

// AXI 参数
#define KP_axi0IdBits 4
#define KP_axi0AddrBits 32
#define KP_axi1IdBits 4
#define KP_axi1AddrBits 32
#define KP_axi2IdBits 6
#define KP_axi2AddrBits 32

// 内存参数
#define KP_itcmSizeKBytes 8
#define KP_dtcmSizeKBytes 32

// 退休缓冲区参数
#define KP_retirementBufferSize 8
#define KP_floatRegfileBaseAddr 32
#define KP_rvvRegfileBaseAddr 64

// 动态参数
#define KP_dbusSize 6
#define KP_useRetirementBuffer false
#define KP_retirementBufferIdxWidth 6
#define KP_useDebugModule false

#endif
```

**使用方法**：

```scala
// 在 Chisel 代码中生成头文件
val params = new Parameters()
params.enableRvv = true
params.enableFloat = true

val header = EmitParametersHeader(params)
// 将 header 写入文件
```

## 16.6 总结

本章提供了 CoralNPU 的完整 API 参考，包括：

1. **硬件 API**：顶层模块接口、端口定义和参数配置
2. **软件 API**：系统调用、库函数和向量内置函数
3. **CSR 参考**：所有 CSR 寄存器的详细位域定义
4. **寄存器映射**：外设寄存器地址和访问方法

这些 API 为 CoralNPU 的硬件集成和软件开发提供了完整的参考。

