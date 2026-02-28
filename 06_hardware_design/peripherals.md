# 6.3 外设

## 6.3.1 外设概述

CoralNPU 集成了多种常用的外设接口，方便与外部设备进行通信和控制。这些外设通过 TileLink-UL 总线与处理器核心连接，提供了统一的访问接口。

### 支持的外设

CoralNPU 目前支持以下外设：

- **GPIO (通用输入输出)**：用于控制和读取数字信号
- **UART (通用异步收发器)**：用于串口通信
- **SPI Master (串行外设接口主机)**：用于与 SPI 设备通信
- **I2C Master (I²C 总线主机)**：用于与 I2C 设备通信

### 外设地址映射

所有外设都映射到内存地址空间中，通过读写特定地址的寄存器来控制外设。以下是外设的基地址：

| 外设 | 基地址 | 地址范围 | 说明 |
|------|--------|----------|------|
| UART0 | 0x40000000 | 4KB | 串口 0 |
| UART1 | 0x40010000 | 4KB | 串口 1 |
| SPI Master | 0x40020000 | 4KB | SPI 主机控制器 |
| GPIO | 0x40030000 | 4KB | 通用 I/O |
| I2C Master | 0x40040000 | 4KB | I2C 主机控制器 |

### TileLink-UL 总线接口

TileLink-UL (Uncached Lightweight) 是一种轻量级的片上总线协议，所有外设都通过这个总线与 CPU 通信。

**TileLink-UL 的特点：**
- 简单的请求-响应模型
- 支持 32 位和 128 位数据宽度
- 无缓存一致性协议（适合外设访问）
- 低延迟

**访问方式：**
在 C 代码中，可以通过指针直接访问外设寄存器：

```c
// 定义寄存器访问宏
#define REG32(addr) (*(volatile uint32_t*)(addr))

// 读取寄存器
uint32_t value = REG32(0x40030000);

// 写入寄存器
REG32(0x40030000) = 0x12345678;
```

**volatile 关键字的作用：**
- `volatile` 告诉编译器这个变量可能会被外部因素改变（比如硬件）
- 防止编译器优化掉对这个变量的读写操作
- 确保每次访问都真实地读写内存

## 6.3.2 GPIO (通用输入输出)

### GPIO 简介

GPIO (General Purpose Input/Output) 是最基本的数字接口，可以配置为输入或输出模式。

**GPIO 的用途：**
- 控制 LED 灯
- 读取按钮状态
- 控制继电器
- 与其他数字设备通信

### GPIO 寄存器映射

GPIO 基地址：`0x40030000`

| 偏移地址 | 寄存器名 | 访问权限 | 说明 |
|----------|----------|----------|------|
| 0x00 | DATA_IN | 只读 | GPIO 输入数据寄存器 |
| 0x04 | DATA_OUT | 读写 | GPIO 输出数据寄存器 |
| 0x08 | OUT_EN | 读写 | GPIO 输出使能寄存器 |

**寄存器说明：**

1. **DATA_IN (0x00)**
   - 读取 GPIO 引脚的当前状态
   - 每个位对应一个 GPIO 引脚
   - 位值为 1 表示高电平，0 表示低电平

2. **DATA_OUT (0x04)**
   - 设置 GPIO 输出值
   - 只有当对应的 OUT_EN 位为 1 时才有效
   - 写入 1 输出高电平，写入 0 输出低电平

3. **OUT_EN (0x08)**
   - 配置 GPIO 方向（输入/输出）
   - 位值为 1 表示输出模式，0 表示输入模式
   - 复位后默认为 0（输入模式）

### GPIO 使用示例

#### 示例 1：控制 LED

```c
#include <stdint.h>

#define GPIO_BASE 0x40030000
#define GPIO_DATA_IN  (GPIO_BASE + 0x00)
#define GPIO_DATA_OUT (GPIO_BASE + 0x04)
#define GPIO_OUT_EN   (GPIO_BASE + 0x08)

#define REG32(addr) (*(volatile uint32_t*)(addr))

// 初始化 GPIO，将引脚 0 配置为输出
void gpio_init(void) {
    // 设置引脚 0 为输出模式
    REG32(GPIO_OUT_EN) = 0x00000001;  // 第 0 位设为 1
}

// 点亮 LED（连接到 GPIO 引脚 0）
void led_on(void) {
    REG32(GPIO_DATA_OUT) = 0x00000001;  // 引脚 0 输出高电平
}

// 熄灭 LED
void led_off(void) {
    REG32(GPIO_DATA_OUT) = 0x00000000;  // 引脚 0 输出低电平
}

// LED 闪烁
void led_blink(void) {
    gpio_init();
    
    while (1) {
        led_on();
        // 延时（简单的循环延时）
        for (volatile int i = 0; i < 1000000; i++);
        
        led_off();
        for (volatile int i = 0; i < 1000000; i++);
    }
}
```

#### 示例 2：读取按钮状态

```c
// 读取按钮状态（连接到 GPIO 引脚 1）
int button_is_pressed(void) {
    // 确保引脚 1 配置为输入（OUT_EN 的第 1 位为 0）
    uint32_t out_en = REG32(GPIO_OUT_EN);
    out_en &= ~(1 << 1);  // 清除第 1 位
    REG32(GPIO_OUT_EN) = out_en;
    
    // 读取引脚状态
    uint32_t data_in = REG32(GPIO_DATA_IN);
    return (data_in & (1 << 1)) ? 1 : 0;  // 检查第 1 位
}

// 按钮控制 LED
void button_control_led(void) {
    // 引脚 0 为输出（LED），引脚 1 为输入（按钮）
    REG32(GPIO_OUT_EN) = 0x00000001;
    
    while (1) {
        if (button_is_pressed()) {
            led_on();
        } else {
            led_off();
        }
    }
}
```

#### 示例 3：操作多个 GPIO 引脚

```c
// 设置多个引脚为输出
void gpio_set_output_pins(uint32_t pin_mask) {
    uint32_t out_en = REG32(GPIO_OUT_EN);
    out_en |= pin_mask;  // 将指定的位设为 1
    REG32(GPIO_OUT_EN) = out_en;
}

// 设置多个引脚的输出值
void gpio_write_pins(uint32_t pin_mask, uint32_t values) {
    uint32_t data_out = REG32(GPIO_DATA_OUT);
    data_out &= ~pin_mask;  // 清除要修改的位
    data_out |= (values & pin_mask);  // 设置新值
    REG32(GPIO_DATA_OUT) = data_out;
}

// 示例：控制 8 个 LED（引脚 0-7）
void led_pattern_demo(void) {
    // 将引脚 0-7 设为输出
    gpio_set_output_pins(0x000000FF);
    
    // 显示不同的 LED 模式
    while (1) {
        // 模式 1：全亮
        gpio_write_pins(0xFF, 0xFF);
        for (volatile int i = 0; i < 1000000; i++);
        
        // 模式 2：全灭
        gpio_write_pins(0xFF, 0x00);
        for (volatile int i = 0; i < 1000000; i++);
        
        // 模式 3：交替闪烁
        gpio_write_pins(0xFF, 0xAA);  // 10101010
        for (volatile int i = 0; i < 1000000; i++);
        
        gpio_write_pins(0xFF, 0x55);  // 01010101
        for (volatile int i = 0; i < 1000000; i++);
    }
}
```

**位操作说明：**
- `|=` (按位或赋值)：用于设置某些位为 1
- `&=` (按位与赋值)：用于清除某些位为 0
- `~` (按位取反)：将 0 变 1，1 变 0
- `<<` (左移)：将位向左移动，例如 `1 << 3` 得到 `0b00001000`


## 6.3.3 UART (通用异步收发器)

### UART 简介

UART (Universal Asynchronous Receiver/Transmitter) 是一种串行通信协议，广泛用于设备间的数据传输。

**UART 的特点：**
- 异步通信（不需要时钟信号）
- 全双工（可以同时发送和接收）
- 简单可靠
- 常用于调试输出、传感器通信等

**UART 通信参数：**
- 波特率 (Baud Rate)：每秒传输的位数，常见值有 9600、115200 等
- 数据位：通常为 8 位
- 停止位：通常为 1 位
- 校验位：通常无校验

### UART 寄存器映射

UART 基地址：
- UART0: `0x40000000`
- UART1: `0x40010000`

| 偏移地址 | 寄存器名 | 访问权限 | 说明 |
|----------|----------|----------|------|
| 0x10 | CTRL | 读写 | 控制寄存器 |
| 0x14 | STATUS | 只读 | 状态寄存器 |
| 0x1C | WDATA | 只写 | 发送数据寄存器 |

**CTRL 寄存器 (0x10)：**

| 位域 | 名称 | 说明 |
|------|------|------|
| [31:16] | NCO | 波特率控制值 |
| [1] | RX_EN | 接收使能（1=使能，0=禁用）|
| [0] | TX_EN | 发送使能（1=使能，0=禁用）|

**NCO (Numerically Controlled Oscillator) 计算公式：**
```
NCO = (波特率 << 20) / 时钟频率
```

例如，如果系统时钟为 50 MHz，波特率为 115200：
```
NCO = (115200 << 20) / 50000000 = 2417
```

**STATUS 寄存器 (0x14)：**

| 位 | 名称 | 说明 |
|----|------|------|
| [0] | TX_FULL | 发送缓冲区满标志（1=满，0=未满）|

**WDATA 寄存器 (0x1C)：**
- 写入要发送的字节数据（低 8 位有效）

### UART 使用示例

#### 示例 1：UART 初始化和发送字符

```c
#include <stdint.h>

#define UART1_BASE 0x40010000
#define UART_CTRL_OFFSET   0x10
#define UART_STATUS_OFFSET 0x14
#define UART_WDATA_OFFSET  0x1C

#define REG32(addr) (*(volatile uint32_t*)(addr))

// 初始化 UART
// clock_frequency_mhz: 系统时钟频率（单位：MHz）
void uart_init(uint32_t clock_frequency_mhz) {
    // 计算 NCO 值，波特率设为 115200
    // NCO = (115200 << 20) / (clock_frequency_mhz * 1000000)
    uint64_t nco = ((uint64_t)115200 << 20) / (clock_frequency_mhz * 1000000);
    
    // 设置控制寄存器：NCO 值 + 使能发送和接收
    // 位 [31:16] = NCO, 位 [1:0] = 0b11 (TX_EN=1, RX_EN=1)
    uint32_t ctrl_value = ((uint32_t)nco << 16) | 0x3;
    REG32(UART1_BASE + UART_CTRL_OFFSET) = ctrl_value;
}

// 发送一个字符
void uart_putc(char c) {
    volatile uint32_t* uart_status = 
        (volatile uint32_t*)(UART1_BASE + UART_STATUS_OFFSET);
    volatile uint32_t* uart_wdata = 
        (volatile uint32_t*)(UART1_BASE + UART_WDATA_OFFSET);
    
    // 等待发送缓冲区不满
    while (*uart_status & 1) {
        // 空操作，防止编译器优化
        asm volatile("nop");
    }
    
    // 写入数据
    *uart_wdata = (uint32_t)c;
}

// 发送字符串
void uart_puts(const char* s) {
    while (*s) {
        uart_putc(*s++);
    }
}
```

**代码说明：**
- `uint64_t`：64 位无符号整数，用于避免计算溢出
- `<< 20`：左移 20 位，相当于乘以 2^20 (1048576)
- `asm volatile("nop")`：插入一条空操作指令，防止编译器优化掉循环

#### 示例 2：发送十六进制数据

```c
// 发送 8 位十六进制数（例如：0xAB 显示为 "AB"）
void uart_puthex8(uint8_t v) {
    const char* hex = "0123456789ABCDEF";
    uart_putc(hex[(v >> 4) & 0xF]);  // 高 4 位
    uart_putc(hex[v & 0xF]);         // 低 4 位
}

// 发送 32 位十六进制数（例如：0x12345678 显示为 "12345678"）
void uart_puthex32(uint32_t v) {
    const char* hex = "0123456789ABCDEF";
    for (int i = 7; i >= 0; i--) {
        uart_putc(hex[(v >> (i * 4)) & 0xF]);
    }
}

// 使用示例
void uart_demo(void) {
    uart_init(50);  // 假设系统时钟为 50 MHz
    
    uart_puts("CoralNPU UART Test\n");
    uart_puts("Value: 0x");
    uart_puthex32(0x12345678);
    uart_puts("\n");
}
```

#### 示例 3：格式化输出

```c
// 发送十进制数
void uart_putdec(uint32_t v) {
    char buffer[11];  // 最多 10 位数字 + 结束符
    int i = 0;
    
    // 特殊情况：0
    if (v == 0) {
        uart_putc('0');
        return;
    }
    
    // 将数字转换为字符串（逆序）
    while (v > 0) {
        buffer[i++] = '0' + (v % 10);
        v /= 10;
    }
    
    // 反向输出
    while (i > 0) {
        uart_putc(buffer[--i]);
    }
}

// 简单的 printf 风格函数（仅支持 %d, %x, %s）
void uart_printf(const char* fmt, ...) {
    // 注意：这是一个简化版本，实际使用需要 stdarg.h
    // 这里仅作示例说明
    
    uart_puts("Temperature: ");
    uart_putdec(25);
    uart_puts(" C\n");
    
    uart_puts("Sensor ID: 0x");
    uart_puthex32(0xABCD1234);
    uart_puts("\n");
}
```

#### 示例 4：调试输出宏

```c
// 定义调试输出宏
#define DEBUG_PRINT(msg) uart_puts("[DEBUG] " msg "\n")
#define DEBUG_VALUE(name, val) do { \
    uart_puts("[DEBUG] " name " = 0x"); \
    uart_puthex32(val); \
    uart_puts("\n"); \
} while(0)

// 使用示例
void debug_example(void) {
    uint32_t register_value = 0x12345678;
    
    DEBUG_PRINT("Starting initialization...");
    DEBUG_VALUE("Register", register_value);
    DEBUG_PRINT("Initialization complete");
}
```

**do-while(0) 宏技巧：**
- 这是 C 语言中常用的宏定义技巧
- 确保宏可以像普通语句一样使用（后面加分号）
- 避免在 if-else 语句中出现问题


## 6.3.4 SPI (串行外设接口)

### SPI 简介

SPI (Serial Peripheral Interface) 是一种高速的同步串行通信协议，常用于与传感器、存储器、显示屏等设备通信。

**SPI 的特点：**
- 同步通信（有时钟信号）
- 全双工（同时发送和接收）
- 高速（可达数十 MHz）
- 主从模式（Master-Slave）

**SPI 信号线：**
- **SCLK (Serial Clock)**：时钟信号，由主机产生
- **MOSI (Master Out Slave In)**：主机发送数据
- **MISO (Master In Slave Out)**：主机接收数据
- **CS/CSB (Chip Select)**：片选信号，低电平有效

**SPI 模式：**
SPI 有 4 种模式，由 CPOL 和 CPHA 两个参数决定：

| 模式 | CPOL | CPHA | 说明 |
|------|------|------|------|
| 0 | 0 | 0 | 空闲时 SCLK 为低，第一个边沿采样 |
| 1 | 0 | 1 | 空闲时 SCLK 为低，第二个边沿采样 |
| 2 | 1 | 0 | 空闲时 SCLK 为高，第一个边沿采样 |
| 3 | 1 | 1 | 空闲时 SCLK 为高，第二个边沿采样 |

- **CPOL (Clock Polarity)**：时钟极性，决定空闲时 SCLK 的电平
- **CPHA (Clock Phase)**：时钟相位，决定在哪个边沿采样数据

### SPI 寄存器映射

SPI Master 基地址：`0x40020000`

| 偏移地址 | 寄存器名 | 访问权限 | 说明 |
|----------|----------|----------|------|
| 0x00 | STATUS | 只读 | 状态寄存器 |
| 0x04 | CONTROL | 读写 | 控制寄存器 |
| 0x08 | TXDATA | 只写 | 发送数据寄存器 |
| 0x0C | RXDATA | 只读 | 接收数据寄存器 |
| 0x10 | CSID | 读写 | 片选 ID 寄存器 |
| 0x14 | CSMODE | 读写 | 片选模式寄存器 |

**STATUS 寄存器 (0x00)：**

| 位 | 名称 | 说明 |
|----|------|------|
| [2] | TX_FULL | 发送 FIFO 满（1=满，0=未满）|
| [1] | RX_EMPTY | 接收 FIFO 空（1=空，0=非空）|
| [0] | BUSY | SPI 忙碌标志（1=忙，0=空闲）|

**CONTROL 寄存器 (0x04)：**

| 位域 | 名称 | 说明 |
|------|------|------|
| [15:8] | DIV | 时钟分频系数 |
| [2] | CPHA | 时钟相位 |
| [1] | CPOL | 时钟极性 |
| [0] | ENABLE | SPI 使能（1=使能，0=禁用）|

**时钟分频计算：**
```
SPI_CLK = SYS_CLK / (2 * (DIV + 1))
```

例如，系统时钟 50 MHz，想要 1 MHz 的 SPI 时钟：
```
DIV = (50 MHz / (2 * 1 MHz)) - 1 = 24
```

**TXDATA 寄存器 (0x08)：**
- 写入要发送的字节数据（低 8 位有效）
- 数据会进入发送 FIFO（深度为 4）

**RXDATA 寄存器 (0x0C)：**
- 读取接收到的字节数据（低 8 位有效）
- 数据来自接收 FIFO（深度为 4）

**CSID 寄存器 (0x10)：**
- 选择片选信号（位 0 对应 CS0）

**CSMODE 寄存器 (0x14)：**
- 位 0：0=自动模式，1=手动模式
- 自动模式：每次传输自动控制 CS
- 手动模式：由软件控制 CS

### SPI 使用示例

#### 示例 1：SPI 初始化

```c
#include <stdint.h>

#define SPI_BASE 0x40020000
#define SPI_STATUS  (SPI_BASE + 0x00)
#define SPI_CONTROL (SPI_BASE + 0x04)
#define SPI_TXDATA  (SPI_BASE + 0x08)
#define SPI_RXDATA  (SPI_BASE + 0x0C)
#define SPI_CSID    (SPI_BASE + 0x10)
#define SPI_CSMODE  (SPI_BASE + 0x14)

#define REG32(addr) (*(volatile uint32_t*)(addr))

// SPI 模式定义
#define SPI_MODE_0  0  // CPOL=0, CPHA=0
#define SPI_MODE_1  1  // CPOL=0, CPHA=1
#define SPI_MODE_2  2  // CPOL=1, CPHA=0
#define SPI_MODE_3  3  // CPOL=1, CPHA=1

// 初始化 SPI
// sys_clk_mhz: 系统时钟频率（MHz）
// spi_clk_mhz: 期望的 SPI 时钟频率（MHz）
// mode: SPI 模式 (0-3)
void spi_init(uint32_t sys_clk_mhz, uint32_t spi_clk_mhz, uint8_t mode) {
    // 计算分频系数
    uint32_t div = (sys_clk_mhz / (2 * spi_clk_mhz)) - 1;
    if (div > 255) div = 255;  // 限制最大值
    
    // 提取 CPOL 和 CPHA
    uint32_t cpol = (mode >> 1) & 1;
    uint32_t cpha = mode & 1;
    
    // 设置控制寄存器
    uint32_t control = (div << 8) | (cpha << 2) | (cpol << 1) | 1;
    REG32(SPI_CONTROL) = control;
    
    // 设置为自动片选模式
    REG32(SPI_CSMODE) = 0;
    
    // 选择 CS0
    REG32(SPI_CSID) = 0;
}

// 等待 SPI 空闲
void spi_wait_idle(void) {
    while (REG32(SPI_STATUS) & 0x1) {
        asm volatile("nop");
    }
}

// 等待接收 FIFO 非空
void spi_wait_rx_ready(void) {
    while (REG32(SPI_STATUS) & 0x2) {
        asm volatile("nop");
    }
}

// 等待发送 FIFO 未满
void spi_wait_tx_ready(void) {
    while (REG32(SPI_STATUS) & 0x4) {
        asm volatile("nop");
    }
}
```

#### 示例 2：SPI 数据传输

```c
// 发送并接收一个字节
uint8_t spi_transfer_byte(uint8_t data) {
    // 等待发送 FIFO 有空间
    spi_wait_tx_ready();
    
    // 发送数据
    REG32(SPI_TXDATA) = data;
    
    // 等待接收完成
    spi_wait_rx_ready();
    
    // 读取接收到的数据
    return (uint8_t)REG32(SPI_RXDATA);
}

// 发送多个字节
void spi_write_bytes(const uint8_t* data, uint32_t len) {
    for (uint32_t i = 0; i < len; i++) {
        spi_transfer_byte(data[i]);
    }
}

// 接收多个字节
void spi_read_bytes(uint8_t* buffer, uint32_t len) {
    for (uint32_t i = 0; i < len; i++) {
        buffer[i] = spi_transfer_byte(0xFF);  // 发送 dummy 字节
    }
}

// 发送后接收（先写后读）
void spi_write_then_read(const uint8_t* tx_data, uint32_t tx_len,
                         uint8_t* rx_buffer, uint32_t rx_len) {
    // 发送阶段
    for (uint32_t i = 0; i < tx_len; i++) {
        spi_transfer_byte(tx_data[i]);
    }
    
    // 接收阶段
    for (uint32_t i = 0; i < rx_len; i++) {
        rx_buffer[i] = spi_transfer_byte(0xFF);
    }
}
```

#### 示例 3：读取 SPI Flash

```c
// SPI Flash 命令定义
#define FLASH_CMD_READ_ID    0x9F  // 读取 ID
#define FLASH_CMD_READ_DATA  0x03  // 读取数据
#define FLASH_CMD_WRITE_EN   0x06  // 写使能

// 读取 SPI Flash ID
uint32_t spi_flash_read_id(void) {
    uint8_t cmd = FLASH_CMD_READ_ID;
    uint8_t id[3];
    
    // 发送命令并读取 3 字节 ID
    spi_transfer_byte(cmd);
    id[0] = spi_transfer_byte(0xFF);
    id[1] = spi_transfer_byte(0xFF);
    id[2] = spi_transfer_byte(0xFF);
    
    spi_wait_idle();
    
    // 组合成 32 位 ID
    return (id[0] << 16) | (id[1] << 8) | id[2];
}

// 从 SPI Flash 读取数据
void spi_flash_read(uint32_t address, uint8_t* buffer, uint32_t len) {
    // 发送读命令和地址
    spi_transfer_byte(FLASH_CMD_READ_DATA);
    spi_transfer_byte((address >> 16) & 0xFF);  // 地址高字节
    spi_transfer_byte((address >> 8) & 0xFF);   // 地址中字节
    spi_transfer_byte(address & 0xFF);          // 地址低字节
    
    // 读取数据
    for (uint32_t i = 0; i < len; i++) {
        buffer[i] = spi_transfer_byte(0xFF);
    }
    
    spi_wait_idle();
}

// 使用示例
void spi_flash_demo(void) {
    // 初始化 SPI：50 MHz 系统时钟，1 MHz SPI 时钟，模式 0
    spi_init(50, 1, SPI_MODE_0);
    
    // 读取 Flash ID
    uint32_t flash_id = spi_flash_read_id();
    
    // 读取 256 字节数据
    uint8_t buffer[256];
    spi_flash_read(0x00000000, buffer, 256);
}
```

#### 示例 4：手动片选控制

```c
// 启用手动片选模式
void spi_manual_cs_enable(void) {
    REG32(SPI_CSMODE) = 1;  // 手动模式
}

// 设置片选状态
void spi_set_cs(uint8_t active) {
    if (active) {
        REG32(SPI_CSID) = 0;  // CS 拉低（激活）
    } else {
        REG32(SPI_CSID) = 1;  // CS 拉高（释放）
    }
}

// 使用手动片选的传输
void spi_manual_transfer_demo(void) {
    spi_manual_cs_enable();
    
    // 激活片选
    spi_set_cs(1);
    
    // 传输数据
    spi_transfer_byte(0x12);
    spi_transfer_byte(0x34);
    
    // 释放片选
    spi_set_cs(0);
}
```


## 6.3.5 I2C (I²C 总线)

### I2C 简介

I2C (Inter-Integrated Circuit) 是一种双线串行通信协议，广泛用于连接低速外设。

**I2C 的特点：**
- 只需要两根信号线（SDA 和 SCL）
- 支持多主机、多从机
- 半双工通信
- 速度较慢（标准模式 100 kHz，快速模式 400 kHz）
- 常用于传感器、EEPROM、RTC 等设备

**I2C 信号线：**
- **SCL (Serial Clock)**：时钟线，由主机产生
- **SDA (Serial Data)**：数据线，双向传输

**I2C 地址：**
- I2C 设备有 7 位或 10 位地址
- 最常用的是 7 位地址（0x00 - 0x7F）
- 地址后跟一个读/写位（0=写，1=读）

**I2C 传输格式：**
```
起始 -> 地址+读写位 -> ACK -> 数据 -> ACK -> ... -> 停止
```

### I2C 寄存器映射

I2C Master 基地址：`0x40040000`

| 偏移地址 | 寄存器名 | 访问权限 | 说明 |
|----------|----------|----------|------|
| 0x00 | STATUS | 只读 | 状态寄存器 |
| 0x04 | CONTROL | 读写 | 控制寄存器 |
| 0x08 | PRESCALE | 读写 | 时钟预分频寄存器 |
| 0x0C | DATA | 读写 | 数据寄存器 |
| 0x10 | CMD | 只写 | 命令寄存器 |

**STATUS 寄存器 (0x00)：**

| 位 | 名称 | 说明 |
|----|------|------|
| [7] | RX_ACK | 接收到的 ACK 位（0=ACK，1=NACK）|
| [6] | BUSY | I2C 总线忙碌（1=忙，0=空闲）|
| [5] | ARB_LOST | 仲裁丢失标志 |
| [1] | TIP | 传输进行中（1=进行中，0=完成）|
| [0] | IF | 中断标志（1=有中断，0=无中断）|

**CONTROL 寄存器 (0x04)：**

| 位 | 名称 | 说明 |
|----|------|------|
| [7] | EN | I2C 核心使能（1=使能，0=禁用）|
| [6] | IEN | 中断使能（1=使能，0=禁用）|

**PRESCALE 寄存器 (0x08)：**
- 设置 SCL 时钟频率的预分频值

**时钟计算公式：**
```
SCL_freq = SYS_CLK / (5 * (PRESCALE + 1))
```

例如，系统时钟 50 MHz，想要 100 kHz 的 I2C 时钟：
```
PRESCALE = (50000000 / (5 * 100000)) - 1 = 99
```

**DATA 寄存器 (0x0C)：**
- 写入要发送的数据或地址
- 读取接收到的数据

**CMD 寄存器 (0x10)：**

| 位 | 名称 | 说明 |
|----|------|------|
| [7] | STA | 产生起始条件 |
| [6] | STO | 产生停止条件 |
| [5] | RD | 读取数据 |
| [4] | WR | 写入数据 |
| [3] | ACK | 发送 ACK（0）或 NACK（1）|
| [0] | IACK | 清除中断标志 |

### I2C 使用示例

#### 示例 1：I2C 初始化

```c
#include <stdint.h>

#define I2C_BASE 0x40040000
#define I2C_STATUS   (I2C_BASE + 0x00)
#define I2C_CONTROL  (I2C_BASE + 0x04)
#define I2C_PRESCALE (I2C_BASE + 0x08)
#define I2C_DATA     (I2C_BASE + 0x0C)
#define I2C_CMD      (I2C_BASE + 0x10)

#define REG32(addr) (*(volatile uint32_t*)(addr))

// I2C 命令位定义
#define I2C_CMD_STA  (1 << 7)  // 起始
#define I2C_CMD_STO  (1 << 6)  // 停止
#define I2C_CMD_RD   (1 << 5)  // 读
#define I2C_CMD_WR   (1 << 4)  // 写
#define I2C_CMD_ACK  (1 << 3)  // ACK
#define I2C_CMD_IACK (1 << 0)  // 清除中断

// I2C 状态位定义
#define I2C_STATUS_RX_ACK   (1 << 7)
#define I2C_STATUS_BUSY     (1 << 6)
#define I2C_STATUS_ARB_LOST (1 << 5)
#define I2C_STATUS_TIP      (1 << 1)  // Transfer In Progress
#define I2C_STATUS_IF       (1 << 0)  // Interrupt Flag

// 初始化 I2C
// sys_clk_mhz: 系统时钟频率（MHz）
// i2c_clk_khz: 期望的 I2C 时钟频率（kHz）
void i2c_init(uint32_t sys_clk_mhz, uint32_t i2c_clk_khz) {
    // 计算预分频值
    uint32_t prescale = (sys_clk_mhz * 1000) / (5 * i2c_clk_khz) - 1;
    
    // 禁用 I2C 核心以设置预分频
    REG32(I2C_CONTROL) = 0;
    
    // 设置预分频
    REG32(I2C_PRESCALE) = prescale;
    
    // 使能 I2C 核心
    REG32(I2C_CONTROL) = 0x80;  // EN=1
}

// 等待传输完成
void i2c_wait_tip(void) {
    while (REG32(I2C_STATUS) & I2C_STATUS_TIP) {
        asm volatile("nop");
    }
}

// 检查是否收到 ACK
int i2c_check_ack(void) {
    return (REG32(I2C_STATUS) & I2C_STATUS_RX_ACK) ? 0 : 1;  // 0=NACK, 1=ACK
}
```

#### 示例 2：I2C 写操作

```c
// 写一个字节到 I2C 设备
// addr: 7 位设备地址
// reg: 寄存器地址
// data: 要写入的数据
int i2c_write_byte(uint8_t addr, uint8_t reg, uint8_t data) {
    // 1. 发送起始条件和设备地址（写模式）
    REG32(I2C_DATA) = (addr << 1) | 0;  // 地址 + 写位(0)
    REG32(I2C_CMD) = I2C_CMD_STA | I2C_CMD_WR;  // 起始 + 写
    i2c_wait_tip();
    
    if (\!i2c_check_ack()) {
        // 没有收到 ACK，设备无响应
        REG32(I2C_CMD) = I2C_CMD_STO;  // 发送停止条件
        return -1;
    }
    
    // 2. 发送寄存器地址
    REG32(I2C_DATA) = reg;
    REG32(I2C_CMD) = I2C_CMD_WR;
    i2c_wait_tip();
    
    if (\!i2c_check_ack()) {
        REG32(I2C_CMD) = I2C_CMD_STO;
        return -1;
    }
    
    // 3. 发送数据
    REG32(I2C_DATA) = data;
    REG32(I2C_CMD) = I2C_CMD_WR | I2C_CMD_STO;  // 写 + 停止
    i2c_wait_tip();
    
    if (\!i2c_check_ack()) {
        return -1;
    }
    
    return 0;  // 成功
}

// 写多个字节到 I2C 设备
int i2c_write_bytes(uint8_t addr, uint8_t reg, const uint8_t* data, uint32_t len) {
    // 发送起始条件和设备地址
    REG32(I2C_DATA) = (addr << 1) | 0;
    REG32(I2C_CMD) = I2C_CMD_STA | I2C_CMD_WR;
    i2c_wait_tip();
    if (\!i2c_check_ack()) return -1;
    
    // 发送寄存器地址
    REG32(I2C_DATA) = reg;
    REG32(I2C_CMD) = I2C_CMD_WR;
    i2c_wait_tip();
    if (\!i2c_check_ack()) return -1;
    
    // 发送数据
    for (uint32_t i = 0; i < len; i++) {
        REG32(I2C_DATA) = data[i];
        
        // 最后一个字节发送停止条件
        if (i == len - 1) {
            REG32(I2C_CMD) = I2C_CMD_WR | I2C_CMD_STO;
        } else {
            REG32(I2C_CMD) = I2C_CMD_WR;
        }
        
        i2c_wait_tip();
        if (\!i2c_check_ack()) return -1;
    }
    
    return 0;
}
```

#### 示例 3：I2C 读操作

```c
// 从 I2C 设备读一个字节
// addr: 7 位设备地址
// reg: 寄存器地址
// data: 存储读取数据的指针
int i2c_read_byte(uint8_t addr, uint8_t reg, uint8_t* data) {
    // 1. 发送起始条件和设备地址（写模式）
    REG32(I2C_DATA) = (addr << 1) | 0;
    REG32(I2C_CMD) = I2C_CMD_STA | I2C_CMD_WR;
    i2c_wait_tip();
    if (\!i2c_check_ack()) {
        REG32(I2C_CMD) = I2C_CMD_STO;
        return -1;
    }
    
    // 2. 发送寄存器地址
    REG32(I2C_DATA) = reg;
    REG32(I2C_CMD) = I2C_CMD_WR;
    i2c_wait_tip();
    if (\!i2c_check_ack()) {
        REG32(I2C_CMD) = I2C_CMD_STO;
        return -1;
    }
    
    // 3. 重新发送起始条件和设备地址（读模式）
    REG32(I2C_DATA) = (addr << 1) | 1;  // 地址 + 读位(1)
    REG32(I2C_CMD) = I2C_CMD_STA | I2C_CMD_WR;
    i2c_wait_tip();
    if (\!i2c_check_ack()) {
        REG32(I2C_CMD) = I2C_CMD_STO;
        return -1;
    }
    
    // 4. 读取数据（发送 NACK 和停止条件）
    REG32(I2C_CMD) = I2C_CMD_RD | I2C_CMD_ACK | I2C_CMD_STO;
    i2c_wait_tip();
    
    // 读取数据
    *data = (uint8_t)REG32(I2C_DATA);
    
    return 0;
}

// 从 I2C 设备读多个字节
int i2c_read_bytes(uint8_t addr, uint8_t reg, uint8_t* buffer, uint32_t len) {
    // 发送起始条件和设备地址（写模式）
    REG32(I2C_DATA) = (addr << 1) | 0;
    REG32(I2C_CMD) = I2C_CMD_STA | I2C_CMD_WR;
    i2c_wait_tip();
    if (\!i2c_check_ack()) return -1;
    
    // 发送寄存器地址
    REG32(I2C_DATA) = reg;
    REG32(I2C_CMD) = I2C_CMD_WR;
    i2c_wait_tip();
    if (\!i2c_check_ack()) return -1;
    
    // 重新起始 + 设备地址（读模式）
    REG32(I2C_DATA) = (addr << 1) | 1;
    REG32(I2C_CMD) = I2C_CMD_STA | I2C_CMD_WR;
    i2c_wait_tip();
    if (\!i2c_check_ack()) return -1;
    
    // 读取数据
    for (uint32_t i = 0; i < len; i++) {
        if (i == len - 1) {
            // 最后一个字节：发送 NACK 和停止条件
            REG32(I2C_CMD) = I2C_CMD_RD | I2C_CMD_ACK | I2C_CMD_STO;
        } else {
            // 其他字节：发送 ACK
            REG32(I2C_CMD) = I2C_CMD_RD;
        }
        
        i2c_wait_tip();
        buffer[i] = (uint8_t)REG32(I2C_DATA);
    }
    
    return 0;
}
```

**I2C 读操作说明：**
- I2C 读操作需要先写入寄存器地址，然后重新起始并切换到读模式
- 读取最后一个字节时要发送 NACK，告诉从机停止发送
- ACK 位在 CMD 寄存器中为 1 时表示发送 NACK（注意这个逻辑）

#### 示例 4：读取 I2C 传感器

```c
// 示例：读取 MPU6050 加速度计/陀螺仪传感器
#define MPU6050_ADDR 0x68  // MPU6050 的 I2C 地址

// MPU6050 寄存器地址
#define MPU6050_WHO_AM_I    0x75
#define MPU6050_PWR_MGMT_1  0x6B
#define MPU6050_ACCEL_XOUT_H 0x3B

// 初始化 MPU6050
int mpu6050_init(void) {
    // 初始化 I2C：50 MHz 系统时钟，100 kHz I2C 时钟
    i2c_init(50, 100);
    
    // 读取 WHO_AM_I 寄存器验证设备
    uint8_t who_am_i;
    if (i2c_read_byte(MPU6050_ADDR, MPU6050_WHO_AM_I, &who_am_i) \!= 0) {
        return -1;  // 通信失败
    }
    
    if (who_am_i \!= 0x68) {
        return -2;  // 设备 ID 不匹配
    }
    
    // 唤醒设备（清除睡眠位）
    if (i2c_write_byte(MPU6050_ADDR, MPU6050_PWR_MGMT_1, 0x00) \!= 0) {
        return -1;
    }
    
    return 0;  // 成功
}

// 读取加速度数据
int mpu6050_read_accel(int16_t* x, int16_t* y, int16_t* z) {
    uint8_t data[6];
    
    // 从 ACCEL_XOUT_H 开始连续读取 6 个字节
    if (i2c_read_bytes(MPU6050_ADDR, MPU6050_ACCEL_XOUT_H, data, 6) \!= 0) {
        return -1;
    }
    
    // 组合高低字节（大端序）
    *x = (int16_t)((data[0] << 8) | data[1]);
    *y = (int16_t)((data[2] << 8) | data[3]);
    *z = (int16_t)((data[4] << 8) | data[5]);
    
    return 0;
}

// 使用示例
void sensor_demo(void) {
    int16_t accel_x, accel_y, accel_z;
    
    if (mpu6050_init() \!= 0) {
        // 初始化失败
        return;
    }
    
    while (1) {
        if (mpu6050_read_accel(&accel_x, &accel_y, &accel_z) == 0) {
            // 成功读取加速度数据
            // accel_x, accel_y, accel_z 包含原始加速度值
        }
        
        // 延时
        for (volatile int i = 0; i < 1000000; i++);
    }
}
```

**int16_t 说明：**
- `int16_t` 是 16 位有符号整数类型（范围 -32768 到 32767）
- 传感器数据通常是有符号的，可以表示正负方向


## 6.3.6 外设编程最佳实践

### 错误处理

在使用外设时，应该始终检查错误并进行适当的处理：

```c
// 好的做法：检查返回值
int result = i2c_write_byte(0x68, 0x6B, 0x00);
if (result != 0) {
    // 处理错误
    uart_puts("I2C write failed\n");
    return -1;
}

// 不好的做法：忽略错误
i2c_write_byte(0x68, 0x6B, 0x00);  // 如果失败怎么办？
```

### 超时保护

在等待外设状态时，应该添加超时保护，避免死循环：

```c
// 带超时的等待函数
int i2c_wait_tip_timeout(uint32_t timeout_us) {
    uint32_t count = 0;
    
    while (REG32(I2C_STATUS) & I2C_STATUS_TIP) {
        count++;
        if (count > timeout_us) {
            return -1;  // 超时
        }
        // 简单延时（实际应用中应使用定时器）
        for (volatile int i = 0; i < 10; i++);
    }
    
    return 0;  // 成功
}

// 使用示例
if (i2c_wait_tip_timeout(1000) != 0) {
    uart_puts("I2C timeout\n");
    return -1;
}
```

### 寄存器访问封装

将寄存器访问封装成函数，提高代码可读性和可维护性：

```c
// 封装 GPIO 操作
typedef struct {
    uint32_t base_addr;
} gpio_t;

void gpio_set_direction(gpio_t* gpio, uint32_t pin, int is_output) {
    uint32_t out_en = REG32(gpio->base_addr + 0x08);
    if (is_output) {
        out_en |= (1 << pin);
    } else {
        out_en &= ~(1 << pin);
    }
    REG32(gpio->base_addr + 0x08) = out_en;
}

void gpio_write(gpio_t* gpio, uint32_t pin, int value) {
    uint32_t data_out = REG32(gpio->base_addr + 0x04);
    if (value) {
        data_out |= (1 << pin);
    } else {
        data_out &= ~(1 << pin);
    }
    REG32(gpio->base_addr + 0x04) = data_out;
}

int gpio_read(gpio_t* gpio, uint32_t pin) {
    uint32_t data_in = REG32(gpio->base_addr + 0x00);
    return (data_in & (1 << pin)) ? 1 : 0;
}

// 使用示例
gpio_t gpio = { .base_addr = 0x40030000 };
gpio_set_direction(&gpio, 0, 1);  // 引脚 0 设为输出
gpio_write(&gpio, 0, 1);          // 引脚 0 输出高电平
```

### 多外设管理

当系统中有多个相同类型的外设时，使用结构体数组管理：

```c
// UART 设备结构
typedef struct {
    uint32_t base_addr;
    uint32_t baud_rate;
    int initialized;
} uart_dev_t;

// UART 设备数组
uart_dev_t uart_devices[] = {
    { .base_addr = 0x40000000, .baud_rate = 115200, .initialized = 0 },
    { .base_addr = 0x40010000, .baud_rate = 115200, .initialized = 0 }
};

// 通用 UART 初始化函数
void uart_init_dev(uart_dev_t* uart, uint32_t sys_clk_mhz) {
    uint64_t nco = ((uint64_t)uart->baud_rate << 20) / (sys_clk_mhz * 1000000);
    uint32_t ctrl = ((uint32_t)nco << 16) | 0x3;
    REG32(uart->base_addr + 0x10) = ctrl;
    uart->initialized = 1;
}

// 通用 UART 发送函数
void uart_putc_dev(uart_dev_t* uart, char c) {
    if (!uart->initialized) return;
    
    volatile uint32_t* status = (volatile uint32_t*)(uart->base_addr + 0x14);
    volatile uint32_t* wdata = (volatile uint32_t*)(uart->base_addr + 0x1C);
    
    while (*status & 1);
    *wdata = (uint32_t)c;
}

// 使用示例
void multi_uart_demo(void) {
    // 初始化所有 UART
    for (int i = 0; i < 2; i++) {
        uart_init_dev(&uart_devices[i], 50);
    }
    
    // 向 UART0 发送
    uart_putc_dev(&uart_devices[0], 'A');
    
    // 向 UART1 发送
    uart_putc_dev(&uart_devices[1], 'B');
}
```

### 调试技巧

#### 1. 寄存器转储

```c
// 打印寄存器值用于调试
void dump_gpio_registers(void) {
    uart_puts("GPIO Registers:\n");
    uart_puts("  DATA_IN:  0x");
    uart_puthex32(REG32(GPIO_DATA_IN));
    uart_puts("\n");
    uart_puts("  DATA_OUT: 0x");
    uart_puthex32(REG32(GPIO_DATA_OUT));
    uart_puts("\n");
    uart_puts("  OUT_EN:   0x");
    uart_puthex32(REG32(GPIO_OUT_EN));
    uart_puts("\n");
}
```

#### 2. 状态检查宏

```c
// 定义调试宏
#define CHECK_STATUS(cond, msg) do { \
    if (!(cond)) { \
        uart_puts("ERROR: " msg "\n"); \
        return -1; \
    } \
} while(0)

// 使用示例
int spi_safe_transfer(uint8_t data) {
    CHECK_STATUS(!(REG32(SPI_STATUS) & 0x4), "SPI TX FIFO full");
    REG32(SPI_TXDATA) = data;
    
    CHECK_STATUS(!(REG32(SPI_STATUS) & 0x2), "SPI RX FIFO empty");
    return REG32(SPI_RXDATA);
}
```

#### 3. 性能测量

```c
// 简单的性能计数器（假设有一个计数器寄存器）
#define CYCLE_COUNTER 0x50000000

uint32_t get_cycles(void) {
    return REG32(CYCLE_COUNTER);
}

void measure_performance(void) {
    uint32_t start = get_cycles();
    
    // 执行要测量的操作
    spi_transfer_byte(0x12);
    
    uint32_t end = get_cycles();
    uint32_t cycles = end - start;
    
    uart_puts("Cycles: ");
    uart_putdec(cycles);
    uart_puts("\n");
}
```

## 6.3.7 常见问题和解决方案

### GPIO 问题

**问题：GPIO 输出无效**
- 检查是否设置了 OUT_EN 寄存器
- 确认引脚没有被其他功能复用
- 检查硬件连接

```c
// 诊断代码
void gpio_diagnose(uint32_t pin) {
    uint32_t out_en = REG32(GPIO_OUT_EN);
    uint32_t data_out = REG32(GPIO_DATA_OUT);
    
    uart_puts("Pin ");
    uart_putdec(pin);
    uart_puts(" status:\n");
    uart_puts("  Direction: ");
    uart_puts((out_en & (1 << pin)) ? "OUTPUT\n" : "INPUT\n");
    uart_puts("  Value: ");
    uart_putdec((data_out & (1 << pin)) ? 1 : 0);
    uart_puts("\n");
}
```

### UART 问题

**问题：UART 无输出**
- 检查波特率配置是否正确
- 确认 TX_EN 位已设置
- 检查系统时钟频率是否正确

```c
// 测试 UART 是否工作
void uart_test(void) {
    // 发送测试字符
    for (char c = 'A'; c <= 'Z'; c++) {
        uart_putc(c);
    }
    uart_puts("\n");
}
```

**问题：UART 乱码**
- 波特率不匹配（最常见）
- 系统时钟频率设置错误
- 接收端配置不正确

### SPI 问题

**问题：SPI 通信失败**
- 检查 CPOL 和 CPHA 设置是否与从机匹配
- 确认时钟频率在从机支持范围内
- 检查片选信号是否正确

```c
// SPI 诊断
void spi_diagnose(void) {
    uint32_t status = REG32(SPI_STATUS);
    uint32_t control = REG32(SPI_CONTROL);
    
    uart_puts("SPI Status:\n");
    uart_puts("  BUSY: ");
    uart_putdec(status & 0x1);
    uart_puts("\n");
    uart_puts("  RX_EMPTY: ");
    uart_putdec((status >> 1) & 0x1);
    uart_puts("\n");
    uart_puts("  TX_FULL: ");
    uart_putdec((status >> 2) & 0x1);
    uart_puts("\n");
    
    uart_puts("SPI Control:\n");
    uart_puts("  ENABLE: ");
    uart_putdec(control & 0x1);
    uart_puts("\n");
    uart_puts("  CPOL: ");
    uart_putdec((control >> 1) & 0x1);
    uart_puts("\n");
    uart_puts("  CPHA: ");
    uart_putdec((control >> 2) & 0x1);
    uart_puts("\n");
    uart_puts("  DIV: ");
    uart_putdec((control >> 8) & 0xFF);
    uart_puts("\n");
}
```

### I2C 问题

**问题：I2C 设备无响应（NACK）**
- 检查设备地址是否正确
- 确认设备已上电
- 检查上拉电阻（I2C 需要上拉电阻）
- 确认 SCL 和 SDA 连接正确

```c
// I2C 总线扫描（查找所有设备）
void i2c_scan(void) {
    uart_puts("Scanning I2C bus...\n");
    
    for (uint8_t addr = 0x08; addr < 0x78; addr++) {
        // 尝试发送起始条件和地址
        REG32(I2C_DATA) = (addr << 1) | 0;
        REG32(I2C_CMD) = I2C_CMD_STA | I2C_CMD_WR;
        i2c_wait_tip();
        
        if (i2c_check_ack()) {
            uart_puts("  Found device at 0x");
            uart_puthex8(addr);
            uart_puts("\n");
        }
        
        // 发送停止条件
        REG32(I2C_CMD) = I2C_CMD_STO;
        i2c_wait_tip();
    }
    
    uart_puts("Scan complete\n");
}
```

**问题：I2C 时钟拉伸**
- 某些 I2C 从机会拉低 SCL 来延长时钟周期
- 确保主机支持时钟拉伸
- 增加超时时间

## 6.3.8 总结

本章介绍了 CoralNPU 的主要外设接口：

1. **GPIO**：最简单的数字 I/O，适合控制 LED、读取按钮等
2. **UART**：串行通信，常用于调试输出和简单的设备通信
3. **SPI**：高速同步串行接口，适合与 Flash、显示屏等设备通信
4. **I2C**：双线串行接口，适合连接多个低速传感器

**选择外设的建议：**
- 需要调试输出 → 使用 UART
- 需要高速数据传输 → 使用 SPI
- 需要连接多个传感器 → 使用 I2C
- 需要简单的数字控制 → 使用 GPIO

**编程要点：**
- 始终使用 `volatile` 关键字访问硬件寄存器
- 添加错误检查和超时保护
- 封装寄存器操作，提高代码可维护性
- 使用调试输出帮助排查问题

通过合理使用这些外设，可以让 CoralNPU 与各种外部设备进行通信，构建完整的嵌入式系统。

