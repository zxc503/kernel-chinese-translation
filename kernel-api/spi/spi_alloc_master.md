# 名称
spi_alloc_master

分配SPI master controller

# 简介
struct spi_master * spi_alloc_master (struct device * dev,unsigned size);

# 参数
- dev

    the controller，可能使用platform_bus。

- size

    需要allocate多大的驱动私有数据(已被初始化为0)。allocate内存的指针存储在返回设备的driver_data字段中，可以用spi_master_get_devdata访问这个指针。

# 上下文

可休眠

# 描述
这个函数仅由SPI master controller驱动使用，controller驱动是唯一直接接触芯片寄存器的。这个函数是在调用spi_register_master之前用于分配spi_master的方式。

必须从可以睡眠的上下文中调用。

这个函数的调用者负责指定总线号，并且在调用spi_register_master之前初始化master的方法；并且应该（在添加设备发生错误之后）调用spi_master_put来防止内存泄漏。

# 返回
成功时返回SPI master结构体，失败返回NULL