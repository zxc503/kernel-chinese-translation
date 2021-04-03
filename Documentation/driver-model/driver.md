# 设备驱动
参见kernel doc中的device_driver结构体

# 分配
设备驱动驱动是静态分配的结构。即使可能在系统中存在一个驱动支持多个设备的情况，结构体device_driver还是将驱动作为一个整体表示(而不是一个特定的设备实例)。

# 初始化
驱动必须至少初始化name和bus字段。驱动还需要初始化devclass字段(当到达时)，所以可能要获取适当的内部联系。驱动需要尽可能初始化更多的回调函数，即使每个回调函数都是可选的。

# 声明
如上所述，结构体device_driver对象是静态分配的。下面是一个声明eepro100驱动的例子。此声明仅为假设，这个假设依赖于驱动已经完全转化为新model。
```
static struct device_driver eepro100_driver = {
       .name          = "eepro100",
       .bus           = &pci_bus_type,

       .probe         = eepro100_probe,
       .remove                = eepro100_remove,
       .suspend               = eepro100_suspend,
       .resume                = eepro100_resume,
};
```
大多数驱动不能被完全转化成新model。因为它们从属的bus拥有一个特定于bus的结构，而这个结构具有无法通用化的特定于bus的字段。

最常见的例子是:设备ID结构。一个驱动通常会定义一系列设备ID(驱动支持的)。设备ID结构的格式和比较设备ID的语义完全是特定于bus的。将它们定义为特定于bus的条目会牺牲类型安全性，因此我们保留特定于bus的结构。

特定于bus的驱动需要在驱动的定义中包含一个通用的结构体device_driver。像这样：
```
struct pci_driver {
       const struct pci_device_id *id_table;
       struct device_driver     driver;
};
```
一个包含特定于bus的字段的定义，像这样(再次使用eepro100驱动作为例子)：
```
static struct pci_driver eepro100_driver = {
       .id_table       = eepro100_pci_tbl,
       .driver               = {
              .name           = "eepro100",
              .bus            = &pci_bus_type,
              .probe          = eepro100_probe,
              .remove         = eepro100_remove,
              .suspend        = eepro100_suspend,
              .resume         = eepro100_resume,
       },
};
```
你可能会发现嵌入结构体初始化的语法很傻甚至难看。但是这是到目前为止，我们找到的最好的方法。

# 注册
```
int driver_register(struct device_driver *drv);
```
驱动在启动时注册这个结构体。对于没有特定于bus的字段(即没有特定于bus驱动结构)的驱动，它们可以使用driver_register()并且传递一个指向device_driver对象的指针。

然而，大对数驱动拥有一个特定于bus的结构并且需要使用像pci_driver_register()之类的方法在总线上注册。

# 过渡到总线驱动
通过定义wrapper方法，可以更容易地过渡到新的model。驱动可以完全忽略通用结构，让bus wrapper填写字段。对于回调函数，总线可以定义通用的回调函数，将调用转发到特定于bus的回调函数上。

这个解决方法只是暂时的。为了获得驱动中的class信息，驱动无论如何进行修改。因为将驱动转化到新的model可以减少基础结构的复杂性和代码大小。建议在添加类信息时对其进行转换。

# 访问
一旦一个对象被注册，就可以访问这个对象的通用字段，比如锁和设备列表：
```
int driver_for_each_dev(struct device_driver *drv, void *data,
                        int (*callback)(struct device *dev, void *data));
```
设备字段是一个列表，其包含所有已经绑定到驱动的设备。LDM核心提供一个helper方法来操作所有驱动控制的设备。这个helper在每次节点访问驱动时锁定驱动，并在每个设备访问驱动时对其进行适当的引用计数。

# sysfs
当一个设备被注册，就会在其bus目录下创建一个sysfs目录。在这个目录中，驱动可以导出一个接口到用户空间，以在全局范围内控制驱动的操作。比如开关驱动内的debug输出。

这个目录的未来功能将是作为一个 "devices"目录。这个目录将包含symlinks，连接到其支持的设备的目录。

# 回调函数

- int (*probe) (struct device *dev)
       probe()是在任务上下文中被调用的，伴随着bus的rwsem锁定以及驱动程序绑定到设备上。驱动通常使用container_of()来将“dev"转化到特定于bus的类型，在probe()和其他方法中都是如此。这些特定于bus的"dev"通常提供设备的resource data，例如pci_dev.resource[]或platform_device.resources，除了dev->platform_data之外，这些resource data也被用来初始化驱动。

       这个驱动回调函数包含特定于驱动的逻辑，用于将驱动程序绑定到给定设备。这些逻辑包含验证设备是否存在，是否是驱动可以处理版本，驱动的数据结构能否被分配和初始化，以及硬件能否被初始化。驱动通常储存一个指向它们状态的指针，通过使用dev_set_drvdata()。当驱动成功绑定其自身到设备时，probe()会返回0并且驱动程序模型代码将完成其绑定驱动程序到该设备上的部分.

       驱动程序的probe()可能会返回一个负的errno值，表示驱动程序没有绑定到这个设备，在这种情况下，它应该释放它分配的所有资源。


- void (*sync_state)(struct device *dev)
       sync_state()对于一个设备只能被调用一次。当驱动的所有消费者设备已经被probe的时候这个方法被调用。设备的消费者列表是通过查找连接该设备与其消费设备的链接，获得的。

       第一次尝试调用sync_state()是在late_initcall_sync()期间发生的，以使固件和驱动程序有时间相互链接设备。在第一次尝试调用sync_state()期间，如果所有设备的消费者在此时都已经probe成功了，sync_state()马上被调用。如果在第一次尝试期间，不存在设备消费者，这也被认为是“所有的设备消费者已经被probe”，并且sync_state()也会被立马调用。

       如果在首次尝试为设备调用sync_state()时，还要消费者没有probe成功，sync_state()调用会被延迟，只有在一个或多个消费者probe成功时才会重新尝试。当驱动核心发现一个或多个消费者还没有被probe时，sync_state()调用会再次被延迟。

       一个 sync_state()的典型用例是，让内核干净利落地从bootloader中接管设备的管理。举个例子，如果一个设备被bootloader配置为一个特定的硬件配置，那么设备的驱动可能需要保持这个配置，直到所有的消费者设备都被已经probe了。一旦所有的消费者设备都被probe了，设备的驱动可以同步设备的硬件状态，以匹配所有设备请求的软件状态。所以叫sync_state()。

       举个明显的例子，包括regulator之类的resource可以从sync_state()中受益。sync_state()同样对复杂的resource很有用，比如IOMMUs。举个例子，IOMMUs包含多个消费者(设备,其地址由IOMMU重新映射)可能需要保持它们的映射固定在启动配置上，直到所有消费者都已经probe了。

       虽然sync_state()的典型用例是，让内核干净利落地从bootloader中接管设备的管理，但是sync_state()的用法不限于此。在任何需要在所有消费者probe之后采取有意义的动作时，都可以使用这个方法。

- int (*remove) (struct device *dev); 
       remove被调用来解除驱动与设备的绑定。在设备被从系统上物理移除、驱动模块被卸载、在重启期间，或是其他情况下，这个方法都可能被调用。

       由驱动决定设备是否存在。这个方法需要释放为这个设备特别分配的任何资源。比如，driver_data字段中的所有东西。
       如果设备还存在，应该静默设备，并将其置于低功耗状态。

- int (*suspend) (struct device *dev, pm_message_t state)
       suspend被调用来使设备置于低功耗状态

- int (*resume) (struct device *dev)
       resume用于使设备从低功耗状态恢复。

# 属性
```
struct driver_attribute {
        struct attribute        attr;
        ssize_t (*show)(struct device_driver *driver, char *buf);
        ssize_t (*store)(struct device_driver *, const char *buf, size_t count);
};
```
设备驱动可以通过sysfs目录导出属性。<br/>
驱动可以使用DRIVER_ATTR_RW和DRIVER_ATTR_RO宏来声明属性，其工作原理与DEVICE_ATTR_RW和DEVICE_ATTR_RO宏相同。

例子：
```
DRIVER_ATTR_RW(debug);
```
这相当于声明：
```
struct driver_attribute driver_attr_debug;
```
然后可以使用这个方法从驱动程序的目录中添加和删除属性:
```
int driver_create_file(struct device_driver *, const struct driver_attribute *);
void driver_remove_file(struct device_driver *, const struct driver_attribute *);
```