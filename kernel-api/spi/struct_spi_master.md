# 名称
struct spi_master

SPI master Controller的接口

# 概述
```
struct spi_master {
  struct device dev;
  struct list_head list;
  s16 bus_num;
  u16 num_chipselect;
  u16 dma_alignment;
  u16 mode_bits;
  u32 bits_per_word_mask;
    #define SPI_BPW_MASK(bits) BIT((bits) - 1)
    #define SPI_BIT_MASK(bits) (((bits) == 32) ? ~0U : (BIT(bits) - 1))
    #define SPI_BPW_RANGE_MASK(min# max) (SPI_BIT_MASK(max) - SPI_BIT_MASK(min - 1))
  u32 min_speed_hz;
  u32 max_speed_hz;
  u16 flags;
    #define SPI_MASTER_HALF_DUPLEX	BIT(0)
    #define SPI_MASTER_NO_RX	BIT(1)
    #define SPI_MASTER_NO_TX	BIT(2)
    #define SPI_MASTER_MUST_RX      BIT(3)
    #define SPI_MASTER_MUST_TX      BIT(4)
  size_t (* max_transfer_size) (struct spi_device *spi);
  spinlock_t bus_lock_spinlock;
  struct mutex bus_lock_mutex;
  bool bus_lock_flag;
  int (* setup) (struct spi_device *spi);
  int (* transfer) (struct spi_device *spi,struct spi_message *mesg);
  void (* cleanup) (struct spi_device *spi);
  bool (* can_dma) (struct spi_master *master,struct spi_device *spi,struct spi_transfer *xfer);
  bool queued;
  struct kthread_worker kworker;
  struct task_struct * kworker_task;
  struct kthread_work pump_messages;
  spinlock_t queue_lock;
  struct list_head queue;
  struct spi_message * cur_msg;
  bool idling;
  bool busy;
  bool running;
  bool rt;
  bool auto_runtime_pm;
  bool cur_msg_prepared;
  bool cur_msg_mapped;
  struct completion xfer_completion;
  size_t max_dma_len;
  int (* prepare_transfer_hardware) (struct spi_master *master);
  int (* transfer_one_message) (struct spi_master *master,struct spi_message *mesg);
  int (* unprepare_transfer_hardware) (struct spi_master *master);
  int (* prepare_message) (struct spi_master *master,struct spi_message *message);
  int (* unprepare_message) (struct spi_master *master,struct spi_message *message);
  int (* spi_flash_read) (struct  spi_device *spi,struct spi_flash_read_message *msg);
  bool (* flash_read_supported) (struct spi_device *spi);
  void (* set_cs) (struct spi_device *spi, bool enable);
  int (* transfer_one) (struct spi_master *master, struct spi_device *spi,struct spi_transfer *transfer);
  void (* handle_err) (struct spi_master *master,struct spi_message *message);
  int * cs_gpios;
  struct spi_statistics statistics;
  struct dma_chan * dma_tx;
  struct dma_chan * dma_rx;
  void * dummy_rx;
  void * dummy_tx;
  int (* fw_translate_cs) (struct spi_master *master, unsigned cs);
};  
Me
};  
```
# 成员
- dev

    该驱动的设备接口

- list
    
    链接到全局的spi_master list

- bus_num

    板级标识符(通常特定于SOC)，用于指定的SPI控制器

- num_chipselect

    片选用于区分不同的SPI slave，并且片选编号从0到num_chipselects。每个slave有一个片选信号，但通常不是每个片选都连接到一个slave。
- dma_alignment

    SPI控制器对DMA buffer alignment的约束

- mode_bits

    controller驱动可理解标志

- bits_per_word_mask

    一个mask，指示驱动支持的bits_per_word的值，位n表示驱动支持每个word n+1bits。如果设置了，SPI core将拒绝任何不支持的bits_per_word传输。如果没设置，这个值就被忽略，由各自的驱动来进行验证

- min_speed_hz

    最低支持的传输速度

- max_speed_hz

    最高支持的传输速度

- flags

    驱动相关的其他约束条件

- max_transfer_size

    函数，用于返回一个spi_device的最大传输大小。可能返回0，那么默认的SIZE_MAX将会被使用

- bus_lock_spinlock

    用于SPI bus锁定的自旋锁

- bus_lock_mutex

    用于SPI bus锁定的互斥锁

- bus_lock_flag

    用于指示SPI bus已经被锁，用于独占的使用

- setup

    函数，用于更新设备的SPI controller使用的设备模式和时种模式。protocol代码可能会调用此命令，如果请求了无法识别或不支持的模式，则此操作必须失败。除非正在修改设置的设备上的传输挂起，否则调用此命令总是安全的。

- transfer

    函数，添加一条message到controller的传输队列

- cleanup

    函数，在release()时被调用，用于释放spi_controller提供的memory

- can_dma

    函数，返回master是否支持DMA

- queued

    标志位，这个master是否提供内置的message队列

- kworker

    用于message pump的线程的结构体

- kworker_task

    指针，指向用于message pump的kworker线程的task结构体

- pump_messages

    work结构体，用于安排work到message pump

- queue_lock

    自旋锁，用于同步message队列的访问

- queue

    message 队列

- cur_msg

    当前正在传送的消息

- idling

    标志位，设备是否进入空闲状态

- busy

    标志位，message pump是否繁忙

- running

    标志位，message pump是否运行中

- rt

    标志位，队列是否被设置成以实时任务运行

- auto_runtime_pm

    core应该确保在硬件准备好时，runtime PM引用已经被持有，通过使用父设备作为spidev。

- cur_msg_prepared

    标志位，true时，spi_prepare_message将被调用来处理当前传输中的消息。

- cur_msg_mapped

    标志位，信息是否已被映射到DMA

- xfer_completion

    由transfer_one_message使用

- max_dma_len

    最大DMA长度

- prepare_transfer_hardware

    函数，调用这个函数告知子系统一个message即将从队列中传来，这样子系统就请求驱动准备传输硬件。

- transfer_one_message

    函数，子系统调用驱动来传输单条message，同时对在此期间到达的传输进行排队。当驱动传输完message，它会调用spi_finalize_current_message，这样子系统就能发送一下条消息。

- unprepare_transfer_hardware

    函数，调用它来告诉子系统当前没有队列中没有更多的消息了，这样子系统就能通知驱动可以让硬件休息了。

- prepare_message

    函数，设置controller以传输单个消息，例如做DMA映射。该函数从线程上下文调用。

- unprepare_message
    
    撤销prepare_message的操作

- spi_flash_read

    函数，调用以让提供加速接口的spi controller来读取flash设备。

- flash_read_supported

    函数，用于判断controller是否支持flash读

- set_cs

    函数，用于设置片选的逻辑层，可能会从中断上下文调用。

- transfer_one
    函数，传输单个spi_transfer.如果过传输成功返回0，如果还在传输返回1。当驱动传输完这个transfer，它会调用spi_finalize_current_transfer，这样子系统就能发送一下条消息。注意：transfer_one和transfer_one_message是相互排斥的；当两者都被设置时，通用子系统不会调用你的transfer_one回调。

- handle_err

    函数，子系统调用驱动程序来处理transfer_one_message的通用实现中出现的错误。

- cs_gpios

    数组，用作片选线的GPIO的数组。每个CS号作为数组中的一个。对于片选线不是GPIO(由SPI controller自身驱动)，其值可以是-ENOENT。

- statistics

    作为spi_transfer的统计

- dma_tx

    DMA发送通道，被core dmaengine helpers使用

- dma_rx

    DMA接收通道，被core dmaengine helpers使用

- dummy_rx

    数组，用于全双工设备的dummy接收buffer

- dummy_tx

    数组，于全双工设备的dummy发送buffer


- fw_translate_cs

    如果引导固件使用的编号方案与linux期望的编号方案不同，这个可选的钩子可以被使用来在两者之间转化。

# 描述
每个SPI master controller可以与一个或多个spi_device通信。这将作为一个小的总线，它们共享MOSI MISO SCK，却使用不同片选。每个设备可以被配置来使用不同的时钟速率，因为直到片选被选中，这些共享信号线都会被忽略。

SPI控制器的驱动程序通过spi_message事务队列管理对这些设备的访问，在CPU内存和SPI从设备之间复制数据。对于它排队等待的每一条这样的消息，它都会在事务完成时调用消息的完成函数。