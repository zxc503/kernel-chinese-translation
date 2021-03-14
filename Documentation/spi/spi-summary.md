# 什么是SPI？
略

# 谁使用SPI，在怎样的系统上使用？
略

# SPI的四种时钟模式是怎样的？
略

# 这些驱动的接口是如何工作的？
<linux/spi/spi.h>头文件包括kernel doc，与主要源代码，您应该阅读内核API文档。本章只是一个概述，所以你会得到了一个大概的了解在了解细节之前。

SPI请求总是进入I/O队列。对给定SPI设备的请求总是按FIFO顺序执行，并且通过回调函数异步地完成请求。还有一些简单的同步包装器，用于这些调用，包括一个常见的transaction类型，如发送一个命令，然后读取响应。

有两种SPI driver类型，叫做
- Controller drivers：Controller可以被内置在SOC上，并且通常支持主/从模式。这些驱动需要接触硬件寄存器，并且可能使用DMA。或它们可以成为PIO bitbangers，仅仅使用GPIO。
- Protocol drivers：这些驱动通过控制器传输消息，以便与SPI线上的主/从设备通信。

例如，一个Protocol driver可能与MTD层通信，将数据导出到存储在SPI flash（如DataFlash）上的文件系统；而其他的一些驱动，可以用来控制音频接口，将触屏传感器作为作为输入接口，或是在运行过程中监控温度和电压。而这写驱动可能共用一个Controller drivers。

存在一个spi_device结构体，在两种driver之间，封装了一个控制器端的接口。

SPI编程接口有一个最小的核心，主要作用是通过使用板级的初始化代码，并通过驱动模型连接contronller和protocol驱动。SPI 出现在sysfs中的多个位置：
- /sys/devices/.../CTLR  给定SPI控制器的物理节点
- /sys/devices/.../CTLR/spiB.C  在总线S"B",片选"C"上的spi_device，通过CTLR操作
- /sys/bus/spi/devices/spiB.C  物理设备.../CTLR/spiB.C的符号链接
- /sys/devices/.../CTLR/spiB.C/modalias 标识应与该device一起使用的driver(用于冷热插拔)
- /sys/bus/spi/drivers/D  用于一个或多个spi*.*的driver
- /sys/class/spi_master/spiB 一个逻辑节点的符号链接（或实际的设备节点），它可以为管理总线“B”的SPI主控制器hold与状态相关的类。所有的spiB.*设备共享一个物理SPI总线段，有SCLK、MOSI和MISO
- /sys/devices/.../CTLR/slave 一个虚拟文件，用于为SPI从控制器(取消注册)注册从设备。将SPI slave Handler的driver名称写入到这个文件以注册从设备。写null来取消注册从设备。
- /sys/class/spi_slave/spiB 一个逻辑节点的符号链接（或实际的设备节点），它可以为管理总线“B”的SPI从控制器hold与状态相关的类。当注册好之后，一个spiB.*设备将出现在这里。可能与其他SPI从设备共享物理SPI总线段

请注意，控制器的类状态的实际位置取决于你是否启用了CONFIG_SYSFS_DEPRECATED。此时，唯一与类有关的状态是总线号（就是"spiB "中的 "B"），所以这些sysclass条目只对快速识别总线有用。
# 板级的初始化代码如何声明SPI设备
Linux需要多个信息来正确配置SPI设备。这些信息通常由与板子相关的代码提供，即使是某些支持自动发现/枚举的芯片也是如此。

## 声明Controller
第一种信息是存在哪些SPI控制器的列表。对于基于SOC的板子，controller通常是platform devices，所以controller需要一些platform_data才能正常工作。结构体platform_device会包含一些资源，诸如第一个控制器的物理地址和IRQ号。

platform通常会抽象"register SPI controller"这一操作，可能将其与代码耦合以初始化引脚配置，以便多个板子的arch/.../mach-*/board-*.c文件可以共享相同的基本控制器设置代码。这是因为大多数soc都有多个SPI的控制器，通常只应设置和注册给定板子上实际可用的控制器。

举个例子，arch/.../mach-*/board-*.c文件可能含有像这样的代码：
```
#include <mach/spi.h>	/* for mysoc_spi_data */

	static struct mysoc_spi_data pdata __initdata = { ... };

	static __init board_init(void)
	{
		...
		/* 这个板子仅仅使用SPI Controller #2 */
		mysoc_register_spi(2, &pdata);
		...
	}
```
并且特定于SOC的实用程序代码可能像：
```
#include <mach/spi.h>

	static struct platform_device spi2 = { ... };

	void mysoc_register_spi(unsigned n, struct mysoc_spi_data *pdata)
	{
		struct mysoc_spi_data *pdata2;

		pdata2 = kmalloc(sizeof *pdata2, GFP_KERNEL);
		*pdata2 = pdata;
		...
		if (n == 2) {
			spi2->dev.platform_data = pdata2;
			register_platform_device(&spi2);

            /* 或是：初始化引脚模式，以便spi2信号能够在相关的引脚上可见，
             * 生产的板子上的bootloaders可能已经做了这些工作。
             * 但是开发板上通常需要linux来做这些工作。
			 */
		}
		...
	}
```

请注意，即使使用相同的SOC控制器，板子的platform_data也可能不同。 例如，在一块板子上，SPI可能会使用外部时钟，而另一块板子上SPI可能来自于当前设置的主时钟。

## 声明从设备
第二种信息是目标板上存在哪些SPI从设备的列表。通常需要一些板级的数据才能使驱动程序正常工作。

通常arch/.../mach-*/board-*.c文件可能提供一个列表，列出了每个板子上的SPI设备。像是：
```
static struct ads7846_platform_data ads_info = {
		.vref_delay_usecs	= 100,
		.x_plate_ohms		= 580,
		.y_plate_ohms		= 410,
	};

	static struct spi_board_info spi_board_info[] __initdata = {
	{
		.modalias	= "ads7846",
		.platform_data	= &ads_info,
		.mode		= SPI_MODE_0,
		.irq		= GPIO_IRQ(31),
		.max_speed_hz	= 120000 /* max sample rate at 3V */ * 16,
		.bus_num	= 1,
		.chip_select	= 0,
	},
	};
```
同样，请注意如何提供板级的信息。每个芯片可能需要几种类型。这个例子展示了一个通用的约束条件，例如最大允许的SPI时钟速率，和IRQ引脚如何连接，加上特定于芯片的限制，例如一个由引脚上的电容决定的延迟值。

还有"controller_data"，一个对controller driver有用的信息。一个例子是特定于外设的DMA调优数据或是片选回调函数。这些信息在之后储存在spi_device中。

board_info需要提供足够的信息，使系统在不加载芯片驱动程序的情况下工作。其中最麻烦的可能是spi_device.mode字段中的SPI_CS_HIGH位，因为与一个“向后”解释片选的设备共享总线是不可能的，除非基础设施知道如何deselect。

然后你的板级初始化代码将会注册这个列表到SPI基础实施中。这样在以后注册SPI master controller时就这些信息可用了。
```
spi_register_board_info(spi_board_info, ARRAY_SIZE(spi_board_info));
```
与其他静态板级初始化一样，你不用取消注册这些信息。

## 非静态配置
开发板(Developer boards)与产品板(Product boards)往往遵循不同的规则，一个例子是存在一个潜在的需求，那就是需要热插拔SPI设备和控制器。

在这个案例中，你可能需要使用spi_busnum_to_master()来寻找spi_master，并且似乎需要spi_new_device()来提供已经插入的热插拔设备的board_info。当然在板子移除之后，你需要调用spi_unregister_device()。

# 如何写一个SPI Protocol Driver？
大多数SPI驱动目前都是内核驱动。但也支持用户空间的驱动，这里我们仅仅讨论内核驱动。

SPI Protocol Driver有点类似于平台设备驱动：
```
static struct spi_driver CHIP_driver = {
		.driver = {
			.name		= "CHIP",
			.owner		= THIS_MODULE,
			.pm		= &CHIP_pm_ops,
		},

		.probe		= CHIP_probe,
		.remove		= CHIP_remove,
	};
```
驱动核心将会自动绑定这个驱动到board_info中modalias为”CHIP“的任何SPI设备中。你的probe()代码可能看起来像这样，除非你创建的设备是用于管理总线的（出现在/sys/class/spi_master）。
```
static int CHIP_probe(struct spi_device *spi)
	{
		struct CHIP			*chip;
		struct CHIP_platform_data	*pdata;

		/* 假设驱动需要板级数据: */
		pdata = &spi->dev.platform_data;
		if (!pdata)
			return -ENODEV;

		/* get memory for driver's per-chip state */
		chip = kzalloc(sizeof *chip, GFP_KERNEL);
		if (!chip)
			return -ENOMEM;
		spi_set_drvdata(spi, chip);

		... etc
		return 0;
	}
```
一旦进入probe(),驱动就会使用结构体"spi_message"发出I/O请求到SPI设备。当remove()返回或是probe()失败的时候，驱动会保证不再发送消息。
- 一个spi_message是一系列协议操作，被作为一个原子序列执行。spi driver控制如下操作：
    + 当双向读和写开始，通过spi_driver决定spi_transfer请求队列如何排序。
    + spi_driver决定使用哪一个I/O buffer，每个spi_transfer包装了一个buffer用于不同传输方向，支持全双工和半双工传输。
    + spi_driver可选地定义了一个短延迟在传输之后，使用spi_transfer.delay_usecs来设置延迟。
    + spi_driver决定在传输和延迟之后，片选是否inacive。通过spi_transfer.cs_change标志进行设置。
    + spi_driver提示下一条消息是否可能也发送到相同的设备。使用原子队列的最后一次传输中的spi_transfer.cs_change标志进行设置。可以潜在地节约片选deslect和select操作之间带来的消耗。
- 在你的message中，应遵循标准的kernel规则，并且提供"DMA安全"的buffer。这样使用DMA的controller drivers就不会强制进行额外的拷贝，除非硬件要求其这么做。

如果这些buffer不适合使用标准的dma_map_single()句柄，你可以使用spi_message.is_dma_mapped来告诉controller driver你已经提供了一个相关的DMA地址。

- 基本I/O的原函数是spi_async()。异步(asnyc)请求可能在任何上下文中被提出(irq处理函数，任务，等等)并且异步请求的完成情况会通过使用由message提供的回调函数进行报告。在检测到错误之后，片选会被deselct并且spi_message的处理过程会被取消。
- 同样也存在同步的包装器，比如spi_sync(),和包装器spi_read()，spi_write(),spi_write_then_read().这些函数只能在可睡眠的上下文中被调用，并且它们都是通过spi_async()实现的。
- spi_write_then_read()和其他由其实现的包装器，应该仅用在少量的数据上，这样额外的复制就可以被忽略。它被用来支持RPC样式的请求，例如spi_w8r16就是其中一个包装器，它发送一个8bit的命令，并且读取16bit的响应。

一些设备可能需要修改spi_deivce特性，例如传输模式，字大小，或是时钟速率。这些可以通过spi_setup()完成，通常在第一次 I/O 到设备完成之前在 probe() 中调用。然而，spi_setup()可以在没有message要传输的任何时间被调用。

spi_device在驱动的底层，它的上层可能包括sysfs(特别是传感器读数)，输入层，ALSA, 网络, MTD，字符设备框架，或是其他的linux子系统。

注意，你的驱动必须管理两种内存类型，作为与SPI设备交互的一部分。
 - I/O buffer使用通常的Linux规则，并且必须是”DMA安全“的。你通常需要从堆或是空闲的页面池中分配buffer。不要使用栈，或是其他被声明为静态的内存。
 - spi_message和spi_transfer元数据(metadata)用于将这些I/O buffer关联到一组protocol transaction。这些可以在任何地方被分配，包括作为其他只分配一次的驱动的数据结构的一部分，需要初始化为零。

 如果你愿意，spi_message_alloc()和spi_message_free()这两个函数可以方便地分配一个具有多个传输的spi_message并将其初始化为零。

 # 如何写一个SPI Master controller Driver？
一个SPI controller 通常注册在platform_bus上；写一个驱动来绑定设备，无论涉及那条bus。

这类驱动的主要任务是提供一个spi_master，使用spi_alloc_master()来分配一个master，使用spi_master_get_devdata()来获取驱动为设备分配的私有数据。
```
struct spi_master	*master;
	struct CONTROLLER	*c;

	master = spi_alloc_master(dev, sizeof *c);
	if (!master)
		return -ENODEV;

	c = spi_master_get_devdata(master);
```
驱动程序将初始化该spi_master的字段，包括总线号(可能与平台设备ID相同)和三种用于与SPI核和SPI协议驱动交互的函数。驱动程序还会初始化自身相关的内部状态。

在你初始化spi_master之后，然后使用spi_register_master()来发布spi_master到系统的其他部分。这时，controller的设备节点和其他预先声明的SPI设备都将可用，这时驱动模型核心将负责将设备绑定到驱动上。

如果你需要移除spi控制器驱动，spi_unregister_master()的作用与spi_register_master()。

# 给总线编号
给总线编号很重要，因为Linux就是这样识别一个给定的SPI总线（共享SCK、MOSI、MISO）。有效总线号从0开始。在SOC上，总线号需要与芯片制造商定义的编号相同。例如，控制器SPI2的总线号就是2，并且连接到控制器的设备的spi_board_info将会使用这个编号。

如果你没有这样的硬件分配的总线号，并且由于某种原因你不能随便分配，那么提供一个负的总线号。 这样就会被一个动态分配的号码所取代。然后，你需要将其视为非静态配置（见上文）

# SPI Master 方法
- master->setup(struct spi_device *spi)
	
	用于设置设备时钟速率、SPI模式和字大小。驱动程序可以改变 board_info 提供的默认值，然后调用spi_setup(spi)来调用这个例程。 这能导致睡眠。
	
	除非每个SPI从机都有自己的配置寄存器，否则不要立即更改它们，否则，驱动程序可能会损坏其他SPI设备正在进行的I/O。

- master->cleanup(struct spi_device *spi)

	您的controller 驱动可能会使用 spi_device.controller_state 来保存它与该设备动态关联的状态。如果您这样做，请确保提供cleanup()方法来释放该状态。
- master->prepare_transfer_hardware(struct spi_master *master)
	
	方法将被队列机制调用，来向驱动发出有message即将到来的信号，因此子系统通过发出此调用请求驱动程序准备传输硬件。这可能导致睡眠。

- master->unprepare_transfer_hardware(struct spi_master *master)

	方法将被队列机制调用，来向驱动发出队列中不在有待处理的消息的信号，并且它有可能放松硬件（例如，通过电源管理调用）。这可能会导致睡眠。
	
- master->transfer_one_message(struct spi_master *master,struct spi_message *mesg)

	子系统调用驱动程序来传输一条message，同时对在此期间到达的传输进行排队。当驱动程序完成这个消息后，它必须调用spi_finalize_current_message()，这样子系统才能发出下一个消息。这可能会导致睡眠。

- master->transfer_one(struct spi_master *master, struct spi_device *spi,struct spi_transfer *transfer)

	子系统调用驱动程序来传输一次transfer，同时对在此期间到达的传输进行排队。当驱动程序完成这个传输后，它必须调用spi_finalize_current_transfer()，这样子系统才能发出下一个transfer。这样子系统才能发出下一次传输。这可能导致睡眠。注意：transfer_one和transfer_one_message是相互排斥的；当两者都被设置时，通用子系统不会调用transfer_one回调。
	
	返回值:
	- 负值：错误
	- 0：传输完成
	- 1：传输进行中

- master->set_cs_timing(struct spi_device *spi, u8 setup_clk_cycles,u8 hold_clk_cycles, u8 inactive_clk_cycles)

	该方法允许SPI client驱动请求SPI master controller配置设备特定的CS设置、保持和非活动时序要求。

# 已弃用方法
- master->transfer(struct spi_device *spi, struct spi_message *message)
	这一定不会导致睡眠。它的职责是安排传输的发生，并且安排complete()回调被调用。这两者通常会在其他传输完成后发生，如果控制器是空闲的，它将需要被启动。此方法不用于排队控制器，如果实现了transfer_one_message()和(un)prepare_transfer_hardware()，则此方法必须为NULL。

# SPI 消息队列
如果您对SPI子系统提供的标准排队机制满意，只需实现上面指定的排队方法。使用消息队列的好处是集中了大量的代码，并提供了纯进程上下文执行的方法。对于高优先级SPI通信，消息队列也可以提升为实时优先级。

除非选择SPI子系统中的队列机制，否则大部分的驱动程序将管理由现在已经废弃的函数transfer()提供的IO队列。

这个队列可能是纯概念性的。例如，一个只用于访问低频传感器的驱动可能使用同步PIO就可以了。

