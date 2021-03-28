# 平台设备
平台设备是一种设备，其在系统中通常作为自主的实体出现。它包括传统的基于端口(IO)的设备和连接外围总线的主机桥，以及集成在SOC的大多数控制器。它们通常的共同点是直接从CPU总线寻址。少数情况下，一个平台设备、会通过其他某种总线的一段连接；但它的寄存器仍然可以直接寻址。

平台设备被赋予一个名称，用于驱动绑定，以及一个资源列表，如地址和IRQ:
```
struct platform_device {
      const char      *name;
      u32             id;
      struct device   dev;
      u32             num_resources;
      struct resource *resource;
};
```
# 平台驱动
平台驱动程序遵循标准驱动程序模型约定，发现/枚举操作在驱动程序外部进行，驱动程序提供probe（）和remove（）方法。它们支持使用标准约定的电源管理和关机通知：
```
struct platform_driver {
      int (*probe)(struct platform_device *);
      int (*remove)(struct platform_device *);
      void (*shutdown)(struct platform_device *);
      int (*suspend)(struct platform_device *, pm_message_t state);
      int (*suspend_late)(struct platform_device *, pm_message_t state);
      int (*resume_early)(struct platform_device *);
      int (*resume)(struct platform_device *);
      struct device_driver driver;
};
```
请注意，probe()通常应验证指定的设备硬件是否确实存在；有时平台设置代码无法确定。probe可以使用设备资源，包括时钟和设备平台数据。
```
int platform_driver_register(struct platform_driver *drv);
```
或者，在设备已知不可热插拔的常见情况下，probe()例程可以放在init部分中，以减少驱动程序的运行时内存占用。
```
int platform_driver_probe(struct platform_driver *drv,int (*probe)(struct platform_device *))
```
kernel模块可以由许多平台驱动组成。platform core提供helper来注册和注销一系列的驱动。
```
int __platform_register_drivers(struct platform_driver * const *drivers,
                              unsigned int count, struct module *owner);
void platform_unregister_drivers(struct platform_driver * const *drivers,
                                 unsigned int count);
```
如果其中的一个驱动注册失败，那么到目前为止注册的所有驱动将以相反的顺序被注销。请注意有一个方便的宏，其传递THIS_MODULE作为所有者参数。
```
#define platform_register_drivers(drivers, count)
```
# 设备枚举

作为一个规则，特定于平台(通常是特定于板子)的setup代码会注册平台设备。
```
int platform_device_register(struct platform_device *pdev);

int platform_add_devices(struct platform_device **pdevs, int ndev);
```
通常的规则是仅仅注册那些确实存在的设备，但是在一些情况下额外的设备也会被注册。举个例子，kernel可能被配置用来与外置网络适配器一起工作，这个适配器可能不在板子上。或者kernel可能被配置用来与一个集成控制器(外部的一个板子)一起工作，而板子本身没有连接任何外设。

在一些情况下，引导固件会导出一个描述给定板子上有哪些设备的表格。如果没有这个表格，那么通常系统setup代码设置正确设备的唯一方式是为特定的板子build一个kernel。这样的特定与板子的内核在嵌入式和定制系统开发中很常见。

在许多情况下，与平台设备相关的内存和IRQ资源不足以让设备的驱动程序工作。板子的配置代码通常会使用设备platform_data字段提供附加的信息。

嵌入式系统通常需要为平台设备提供一个或多个时钟。这些时钟通常是保持关闭的，直到主动需要它们（以节省电源）。系统设置也将这些时钟与设备关联起来，这样调用clk_get(&pdev->dev, clock_name)就可以根据需要返回这些时钟。

# 传统驱动：设备探测
有些驱动程序并没有完全转换为驱动程序模型，因为它们承担了一个非驱动程序的角色（驱动和设备是耦合的）：平台设备由驱动注册，而不是由其他系统基础设施注册。这样的驱动程序不能被热插拔或冷插拔，因为这些插拔机制要求设备的创建与设备的驱动是在不同的系统组件中的。

这么做的唯一好处是用来处理那些旧的系统设计，这些旧系统像最初的IBM PC一样，依靠容易出错的 "探测硬件 "模式进行硬件配置。较新的系统基本上已经放弃了这种模式，而采用以总线为支持的动态配置（比如PCI、USB），或是由引导固件提供的设备表（例如x86上的PNPACPI）。关于什么东西可能在哪里，有太多相互矛盾的选择，即使是操作系统有根据的猜测也会经常出错，从而造成麻烦。

这种风格的驱动程序是不鼓励的。如果你要更新这样的驱动程序，请尝试在驱动之外，将设备枚举移到一个更合适的位置。这类驱动通常会被清理，因为这类驱动往往已经有了 "典型"的模式，比如驱动使用由PNP或者平台设备配置创建的设备节点。

尽管如此，还是有一些API来支持这样的遗留驱动。避免使用这些调用，除非使用这种存在热插拔缺陷的驱动。

```
struct platform_device *platform_device_alloc(const char *name, int id);
```
你可以使用platform_device_alloc()来动态的分配一个设备，然后用resources和platform_device_register()方法来初始化它。一个更好的解决方案通常是使用：
```
struct platform_device *platform_device_register_simple(
                const char *name, int id,
                struct resource *res, unsigned int nres);
```
你可以使用platform_device_register_simple()一步来分配和注册设备。

# 设备命名和驱动绑定
platform_device.dev.bus_id是设备的规范名称。它是由两个部分构建的：
- platform_device.name ,它也用于与驱动匹配。
- platform_device.id 设备实例号，或者是"-1 "表示只有一个。

这两个部分是连接在一起使用的，name=serial/id=0，表示bus_id"serial.0"，name=serial/id=3 表示bus_id “serial.3”。它们都使用名为serial的platform_driver。而name=my_rtc/id=-1的bus_id就是"my_rtc"(没有实例号)，并且使用叫做"my_rtc"的platform_driver.

驱动程序绑定是由驱动核心自动执行的，当找到驱动和设备之间的匹配之后，驱动的probe()就会被执行。当probe()执行成功之后，驱动和设备就会像通常一样被绑定在一起。有三种不同的方法来寻找这种匹配。

- 每当一个设备被注册后，该总线的驱动就会检查是否匹配。 平台设备应该在系统启动非常早期就被注册。
- 当使用platform_driver_register()注册一个驱动时，所有该总线上未绑定的设备都应被检查是否匹配。驱动程序通常在启动过程中注册，或者通过模块加载。
- 使用platform_driver_probe()注册驱动程序就像使用platform_driver_register()一样，不同的是，使用platform_driver_probe()，如果有其他设备注册，驱动以后就不会被probe到。(这是好的，因为此接口仅用于非热插拔设备。)

# 早期的平台设备和驱动程序
早期的platform接口，在系统启动的早期为平台设备驱动提供platform data。这段代码建立在early_param()命令行解析之上，可以很早的执行。

例子: 注册“earlyprintk” 类早期串口控制台的六个分步骤

1. 注册早期平台设备data

      架构代码使用函数early_platform_add_devices()注册平台设备的data。在早期串口控制台的例子中，这个data应该是串口的硬件配置。在这个时间点注册的设备，之后将被与平台驱动匹配。

2. 解析内核命令行

      架构代码调用parse_early_param()来解析内核的命令行，这将执行所有匹配的early_param()回调。这将执行所有匹配的early_param()回调。用户指定的早期平台设备将在此时被注册。在早期串口控制台的例子中，用户可以在内核命令行上指定端口为“earlyprintk=serial.0”，其中“earlyprintk”是string类型。，“serial”是平台驱动的名字，0是平台设备的id。如果id是-1，那么点和id都可以被忽略。

3. 安装属于某一类的早期平台驱动

      架构代码可以通过使用函数early_platform_driver_register_all()，选择性地强制注册属于某个类的所有早期平台驱动程序。用户在步骤2指定的设备的优先级高于这一步的设备。这个步骤被串口驱动的例子省略了，因为早期的串口驱动代码应该是被禁用的，除非用户在内核命令行中指定了端口。

4. 早期平台驱动注册

      使用early_platform_init()编译的平台驱动会在步骤2或3中自动注册。串口驱动的例子中，应该使用early_platform_init("earlyprintk", &platform_driver)。

5. probe属于某个类的早期平台驱动

      架构代码调用early_platform_driver_probe()来匹配已注册的早期平台设备(与某个类相关联)与已注册的早期平台驱动。匹配到的设备会被probed()。这一步可以在早期启动期间的任何时候执行。对于串口的例子，尽快执行可能会更好。

6. 在早期的平台驱动probe()类

      在早期启动期间，驱动程序代码需要特别小心，尤其是在内存分配和中断注册方面。probe()函数中的代码可以使用is_early_platform_device()来检查是它是在早期平台设备时被调用还是在常规的平台设备时调用。早期串口驱动此时会执行 register_console()。


## 更多信息请参见<linuxplatform_device.h>。