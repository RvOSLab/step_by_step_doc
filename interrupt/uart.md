# 驱动 UART

## 介绍

通过读取设备树信息，我们可以知道 QEMU 使用的 UART 设备与 ns16550a 兼容，阅读全志哪吒 D1 开发板的文档可知它的 UART 设备也与 ns16550a 兼容（有部分区别）。因此我们参考 ns16550a 的文档编写两者的驱动程序。其中对于 D1 开发板，我们暂时只关注 UART0，也就是主板上的调试接口。

```
// QEMU Device Tree
uart@10000000 {
    interrupts = <0x0a>;
    interrupt-parent = <0x03>;
    clock-frequency = "\08@";
    reg = <0x00 0x10000000 0x00 0x100>;
    compatible = "ns16550a";
};
```

关于 ns16550a，这里主要参考 *[Serial UART information](https://www.lammertbies.nl/comm/info/serial-uart)*、*D1_User_Manual_V0.1(Draft Version)*与*[龙芯 1C300 处理器用户手册 1.4 版](https://mirrors.aliyun.com/loongson/loongson1c_bsp/%e9%be%99%e8%8a%af1C%e5%a4%84%e7%90%86%e5%99%a8%e7%94%a8%e6%88%b7%e6%89%8b%e5%86%8cV14.pdf)*，其中龙芯 1C300 中所用 UART 芯片也是 16550a 兼容芯片，可供参考。

UART 全称通用异步接收发射器，它提供与外部设备、调制解调器（数据载波设备、DCE）的异步串行通信。它将外设接收的数据进行串-并行转换，并将转换后的数据传输到内部总线。它也将传输到外设的数据执行并-串行转换。

D1 开发板的 UART 芯片有以下几个特性：

- 兼容 16550 UART
- 有两个独立的 FIFO：接收 FIFO 和发送 FIFO，UART0 中每个都是 64 bytes
- 从 APB 总线时钟获得工作参考时钟（忽略）
- RS-232 字符的 5 到 8 位数据位，或 RS-485 格式的 9 位
- 支持 1、1.5 或 2 位停止位
- 可编程的校验（奇校验、偶校验或不校验）
- 支持 DMA
- 支持软件/硬件流控制

### UART 操作模式

#### 数据帧格式

UART_LCR 寄存器可以设置数据帧的基本参数：数据宽度(5-8 bits)、停止位（1、1.5 或 2 位停止位）、校验形式。

UART 的帧传输包括启动信号、数据信号、奇偶校验位和停止信号。最低有效位（LSB）首先传输。

- 开始信号（开始位）：数据帧的开始标志。根据 UART 协议，开始位 TXD 信号为低电平。
- 数据信号（数据位）：数据位宽度可以配置为 5 - 8 位。
- 奇偶校验位：1 位的错误校正信号。奇偶校验位可通过 UART_LCR 设置奇校验、偶校验或无校验。
- 停止信号（停止位）：数据帧的停止位。UART_LCR 寄存器可以设置停止位为 1 位、1.5 位或 2 位。结束位 TXD 信号为高电平。

#### 波特率

波特率计算如下：波特率 = SCLK /（16 * 除数）。D1 开发板中，SCLK 通常是 APB1，可在 CCU 中设置。

除数是 UART 的分频器。分频器有 16 位，低 8 位在 UART_DLL 寄存器中，高 8 位在 UART_DLH 寄存器中。

如果要设置波特率为 115200，时钟源的频率为 24 MHz，根据公式可得除数(divisor)约为 13.

#### DLAB 定义

UART_LCR 的第 7 位为 DLAB(Divisor Latch Access Bit)，控制除数锁存器的访问。

- DLAB = 0，第一、二个寄存器是 UART_RBR/UART_THR (RX/TX FIFO) 寄存器、UART_IER 寄存器。
- DLAB = 1，第一、二个寄存器是 UART_DLL（除数低位）寄存器、UART_DLH（除数高位）寄存器

初始化设置除数时须设 DLAB = 1，结束配置时须设 DLAB = 0 以便访问数据。

#### CHCFG_AT_BUSY 定义（全志平台特有）

CHCFG_AT_BUSY（UART_HALT[1]）和 CHANGE_UPDATE（UART_HALT[2]）的功能如下。

CHCFG_AT_BUSY：启用该位，则软件可以在 UART 繁忙时依然设置 UART 控制器，例如 UART_LCR、UART_DLH、UART_DLL 寄存器。

CHANGE_UPDATE：如果启用了 CHCFG_AT_BUSY，并将 CHANGE_UPDATE 写入 1，则 UART 控制器的设置会被更新。完成更新后，位会自动清除到 0。

设置除数应执行以下步骤：

1. 设 CHCFG_AT_BUSY = 1
2. 设 DLAB = 1 并设置 UART_DLH 与 UART_DLL 寄存器
3. 设 CHANGE_UPDATE = 1 以更新配置，更新完成后将会自动被设为 0

#### UART 忙标志位（全志平台特有）

UART_USR[0] 是 UART 控制器的忙标志位。当有收发数据时，或 FIFO 不为空时，忙标志位就会被自动设为 1，表明 UART 控制器正忙。

## 寄存器数据结构设计

寄存器参考 D1 文档第 9.2.5 节寄存器列表与 9.2.6 节寄存器描述。

- QEMU：只有从 `UART_RBR` 到 `UART_SCH` 的寄存器，我们使用全部这些寄存器。
  - 寄存器宽度为 8bit
  - 其余寄存器均为全志平台特有
- 全志平台：对于全志平台我们使用从 `UART_RBR` 到 `UART_HALT` 的寄存器。
  - 寄存器宽度为 32bit，但多数都只有低 8 位有效（兼容 16550a）

## 程序设计

### 初始化

#### 系统初始化(全志平台特有，忽略)

1. 在时钟控制管理单元(CCU)下配置 APB1_CFG_REG 寄存器以设置 APB1 总线时钟频率（默认为 24MHz）
2. 设置 UART_BGR_REG[UARTx_GATING] = 1 以打开对应 UART 模块的时钟，设置 UART_BGR_REG[UARTx_RST] = 1 可使模块无效（释放）

#### UART 控制器初始化

I/O 配置(全志平台特有，忽略)：将 GPIO 多路复用配置为 UART 功能，并将 UART 引脚设置为内部上拉模式。

波特率配置:
1. 设置 UART 波特率
2. 设置 UART_FCR[FIFOE] = 1 以启用 TX/RX FIFO
3. 设置 UART_HALT[HALT_TX] = 1 以禁用 TX 传输
4. 设置 UART_LCR[DLAB] = 1，其他位保留默认配置，使第一第二个端口进入波特率设置模式以访问分频锁存器
5. 向 UART_DLH 寄存器写入分频的高八位，向 UART_DLL 寄存器写入分频的低八位
6. 设置 UART_LCR[DLAB] = 0, 保留其他位不变，恢复正常模式
7. 设置 UART_HALT[HALT_TX] = 0 以启用 TX 传输

控制参数配置：
1. 写入 UART_LCR 以设置数据宽度、停止位、奇偶校验方式
2. 重置，启用 FIFO，写入 UART_FCR 寄存器设置 FIFO 触发条件
3. 写入 UART_MCR 设置控制流参数

中断配置：

在中断模式中，配置 UART_IER 寄存器以启用对应的中断请求，如接收/发送的中断等

### 中断模式中收发数据

#### 数据发送

1. 设置 UART_IER[ETBEI] = 1 以启用 UART 发送中断
2. 写入将要被传送至 UART_THR 寄存器的数据
3. 当 TX_FIFO 的数据达到触发条件时（如达到FIFO/2或FIFO/4)，UART 发送中断信号将会产生
4. 检查 UART_USR[TFE] 以检查 TX_FIFO 是否为空。若 UART_USR[TFE] = 1，则表明 TX_FIFO 已完全发送了
5. 设 UART_IER[ETBEI] = 0 可以禁用发送中断

#### 数据接收

1. 设置 UART_IER[ERBFI] = 1 以启用 UART 接收中断
2. 当 RX_FIFO 的数据达到触发条件时（如达到FIFO/2或FIFO/4)，UART 接受中断信号将会产生
3. 读取 UART_USR[RFNE] 检查 RX_FIFO 状态，若该位为 1，继续从 UART_RBR 读取数据，直至该位变为 0，接收结束

