# 名称
struct spi_device

SPI slave设备的master侧代理

# 概述
```
struct spi_device {
  struct device dev;
  struct spi_master * master;
  u32 max_speed_hz;
  u8 chip_select;
  u8 mode;
    #define SPI_CPHA	0x01
    #define SPI_CPOL	0x02
    #define SPI_MODE_0	(0|0)
    #define SPI_MODE_1	(0|SPI_CPHA)
    #define SPI_MODE_2	(SPI_CPOL|0)
    #define SPI_MODE_3	(SPI_CPOL|SPI_CPHA)
    #define SPI_CS_HIGH	0x04
    #define SPI_LSB_FIRST	0x08
    #define SPI_3WIRE	0x10
    #define SPI_LOOP	0x20
  u8 bits_per_word;
  int irq;
  void * controller_state;
  void * controller_data;
  const char * modalias;
}; 
```
# 成员
- dev

    设备的驱动模型表示。

- master

    设备使用的SPI controller

- max_speed_hz

    此chip使用的最大的时钟速率。可由设备的驱动更改，spi_transfer.speed_hz每次传输时可以覆盖着值

- chip_select

    片选，用于区分master处理的chip

- mode

    spi模式定义了数据输入和输出的方式。可被设备的驱动更改。片选默认的“active low”可以被覆盖(通过指定SPI_CS_HIGH)，正如传输中每个字的 "MSB优先 "默认值一样(通过指定SPI_LSB_FIRST)
- bits_per_word

    数据传输包含一个或者多个words。word的大小通常是8或12bits。内存中的字大小是两个字节的幂数（例如20位采样使用32位）。字长可以通过设备的驱动程序改变，或保留默认值(0)，表示字长是8bit。spi_transfer.bits_per_word可以在每次传输时覆盖这个值。

- irq

    负数，或是被传递给request_irq的数字，用于接收设备中断。

- controller_state

    controller的运行时状态

- controller_data

    controller的板级定义，例如FIFO初始化参数，来自board_info.controller_data。

- modalias

    这个设备使用的驱动名称，或是别名。它出现在驱动冷插拔的sysfs的“modalias”属性，以及用于热插拔的uevents中。

# 描述
spi_device用于在SPI slave和CPU存储器之间交换数据。在dev中platform_data用于保存关于这个设备的信息，这些信息对设备的protocol driver有用，但是对设备的controller没用。一个例子是一个标识符用于表示chip之间细微的不同，另一个例子是关于这个特定的板子如何连接芯片的引脚的。