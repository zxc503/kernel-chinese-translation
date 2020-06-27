# GPIO Descriptor Driver Interface
这个文档可作为GPIO芯片驱动编写者的指南。请注意，这个文档描述的是新的基于描述符的接口(descriptor-based interface)。关于已经被启弃用的基于整数的接口(integer-based interface)请参考文档gpio-legacy.txt。

每个GPIO控制器驱动都需要include下面这个头文件，它定义了定义GPIO驱动需要用到的结构体。
```
    #include <linux/gpio/driver.h>
```
# Internal Representation of GPIOs
一个GPIO chip处理一条或多条GPIO lines。作为一个GPIO chip，line必须符合定义：通用输入/输出。如果一个line不是通用的，它就不是GPIO，就不该由GPIO chip处理。某些line在系统中可能被称为GPIO，但是却用作特殊目的，因此不符合通用输入/输出的标准。另一方面，一些LED驱动线可以用作GPIO，因此仍应由GPIO chip驱动处理。

在GPIO驱动程序中，每个GPIO line由它们的硬件编号来标识，有时也被称为偏移(offset)。偏移是0到n-1的唯一数，其中n是芯片管理的GPIO数量。

硬件GPIO编号从硬件上来看，应该是直观的，例如如果一个系统使用一组内存映射的I/O寄存器来处理32个GPIO line，这个32位的寄存器的每一个bit都处理一个GPIO line,那么使用硬件偏移量0~31，对应寄存器的0~31 bit，是很合理的。

这个数字只对内部可见，特定GPIO line的硬件编号在驱动之外是看不见的。

在这个内部编号的基础上，每个GPIO还需要在整数GPIO命名空间中拥有一个全局编号，以便兼容传统的GPIO接口。每个chip必须有一个“base”数(可以被自动分配)，那么每个GPIO line的全局编号就是base+硬件编号。尽管基于整数的GPIO接口已经被弃用，但是仍有大量用户，因此需要维护。

举个例子，一个平台的GPIO全局编号范围是32~159，它使用一个控制器以32作为base，来定义128个GPIO line。而另一个平台使用全局编号0~63作为一组GPIO控制器，64-79作为另一种类型的GPIO控制器，并且在另一个特定的板子上，使用80~95表示一个FPGA。剩余的数字不必是连续的，这两个平台中的任何一个也可以使用数字2000~2063来表示I2C GPIO扩展器组中的GPIO line。

# Controller Drivers: gpio_chip
在gpiolib框架中，每个GPIO 控制器被封装成一个“gpio chip”（查看<linux/gpio/driver.h>获取详细定义)，其成员为该类型的每个控返回梯制器所共有，这些成员应该由驱动代码分配：

+ 一个方法用来建立GPIO line的方向
+ 一个方法用来访问GPIO line的值
+ 一个方法用来为给定的GPIO line配置电气属性
+ 一个方法用来返回与给定GPIO line相关联的IRQ编号
+ 一个flag说明调用其方法是否会休眠
+ 可选的line名称数组用来定义line
+ 可选的debugfs dump方法（显示额外的状态信息）
+ 可选的base编号（如果忽略的话，会自动分配）
+ optional label for diagnostics and GPIO chip mapping using platform data

实现gpio_chip的代码需要支持多个控制器实例，最好使用驱动模型。该代码将配置每个gpio_chip并且会执行gpiochip_add(),gpiochip_add_data()或devm_gpiochip_add_data()。移出一个GPIO控制器是很少见的，必须要移除的时候使用gpiochip_remove()。

通常情况下，gpio_chip是一个特定实例结构的一部分，其状态不被GPIO接口所暴露，如寻址、电源管理等。 音频编解码器等芯片会有复杂的non-GPIO状态。

任何debugfs dump方法通常应该忽略没有被请求的line。debugfs dump方法可以使用函数gpiochip_is_requested()来判断，这个函数会返回NULL或者在GPIO line被请求时返回与该GPIO line关联的label。

实时注意事项:如果要在实时内核上从原子上下文中调用GPIO API（在硬IRQ处理程序和类似的上下文内），GPIO驱动就不能在gpio_chip实现中(如.get/.set和方向控制回调函数)使用spinlock_t或者其他任何可睡眠的API(如PM runtime 运行时电源管理)。通常情况下没有这个限制。

# GPIO electrical configuration
GPIO line可以被配置成多个电气操作模式，通过使用.set_config()回调函数。当前这个API支持设置：

+ 消除抖动
+ 单端模式（开漏/开源)
+ 上拉和下拉电阻使能

这些设置会在下面详细描述。

.set_config()会调函数使用与pinctrl驱动相同的枚举和配置定义。这不是巧合：gpio_chip很有可能会把set_config()分配给函数gpiochip_generic_config()，而gpiochip_generic_config()将导致pinctrl_gpio_set_config()被调用，并且最终在GPIO控制器背后的pinctrl中结束。

## GPIO lines with debounce support

消抖是一个设置到引脚上的配置，用以表明引脚连接到一个机械开关或按钮，或者类似的会产生抖动的东西。抖动意味着line因为机械原因在非常短的间隔内被快速拉高/低。这会导致值不稳定或是IRQ会被重复调用，除非line是防抖的。

在实际中消抖意味着当在line上发生事件的时候，会设置一个定时器，等到一会儿之后再重新在line上重新采样，以便查看line上是否具有相同的值。这也可以通过一个clever state machine来重复，等待一个line变稳定。在这两种情况下，它都会为消抖设置一定的毫秒数，如果消抖时间不可配置，则仅仅设置on/off.

## GPIO lines with open drain/source support
开漏(CMOS)和开集(TTL)意味着line没有主动输出高：相反地，你提供漏极/集电极作为输出，所以当集体管没有被打开时，他会对外部rail呈现高阻（三态）。
```
CMOS CONFIGURATION      TTL CONFIGURATION

         ||--- out              +--- out
  in ----||                   |/
         ||--+         in ----|
             |                |\
            GND                 GND
```
这个配置通常是作为实现下列两个事情中的一个的方法：

## GPIO lines with pull up/down resistor support

GPIO line可以通过使用.set_config()回调函数来支持上/下拉。这意味着输出GPIO line有一个可用的上/下拉电阻，并且它可以通过软件控制。

在一般的设计中，一个上拉或下拉电阻是简单的焊在电路板上的。这样我们就不需要在软件中处理什么。最让你先到的是这些line会被配置为开漏或开源(查看上面一节)。

回调函数.set_config()仅仅只能开关上/下拉，不能控制使用的电阻值。这说明它仅仅开关寄存器中的一个bit，来启用/禁用上下拉。

如果GPIO line支持调节上/下拉使用的不同电阻值，那么GPIO chip回调函数.set_config()就不够用了。为了实现这种复杂的用法，GPIO chip和pinctrl必须合在一起使用，因为pinctrl的pin config接口支持控制多种电气属性，并且可以处理多种上下拉电阻值。

## GPIO drivers providing IRQs

GPIO驱动(GPIO chips)同样提供中断支持，通常是级联自上一级中断控制器，并且在一些特殊情况下

GPIO block的IRQ部分是使用irq_chip实现的，使用头文件<linux/irq.h>。所以这个联合驱动同时利用两个子系统：gpio和irq。

对于任何IRQ消费者来说都可以从irq_chip请求一个IRQ，即使这是一个GPIO和IRQ的联合驱动。基础的前提是gpio_chip和irq_chip是独立的，它们相互独立的提供服务。

gpiod_to_irq()是以个便利的函数用来指出某个GPIO line的IRQ，并切不需要在IRQ使用之前被调用，也能够正常使用。

请准备好硬件，并使其在GPIO和irq_chip API中的回调函数中能够正常工作做好准备。不要依赖于首先调用的gpiod_to_irq()。

我们可以把GPIO irqchips分在两个板目录中：

+ 级联中断芯片：这意味着GPIO chip有一个通用的中断输出线，它通过chip上任何被启用的GPIO line来触发。这个中断输出线将被级联到上一级的中断控制器中，最简单的情况下是系统主中断控制器。这个模型是irqchip决定的，irqchip会检查GPIO控制器中的bits，来确定是哪一个line触发了中断。驱动的irqchip部分需要检查寄存器来确定是那个line触发中断，而且似乎还需要通过清除某些bit来确认它处理了这个中断(有时是隐式的，比你读状态寄存器)。而且通常还需要进行一些配置，例如边缘敏感度(上升沿/下降沿，或是高/低电平中断)

+ 分层中断芯片这意味着每一个GPIO line有一个专用的irq line到上一级的中断控制器。不需要查询GPIO硬件来确定哪个line被触发，但仍有必要进行清除中断（确认中断）并且配置像是边缘敏感度等配置。

实时(Realtime)注意事项：一个实时兼容的GPIO驱动在irqchip的实现中不应该使用spinlock_t或其他可睡眠的API（例如PM runtime）。

+ 使用raw_spinlock_t代替spinlock_t
+ 如果必须使用可睡眠API，可以通过使用.irq_bus_lock()和.irq_bus_unlock()回调函数来完成，因为这是irqchip上唯一的慢路径回调函数。如果需要就创建回调函数。

# Cascaded GPIO irqchips
级联GPIO irqchip通常分为三类：
+ 链式级联GPIO irqchip(CHAINED CASCADED GPIO IRQCHIPS)：这种类型通常是嵌入在SOC中的。这意味着GPIOs有一个快速IRQ流处理程序，从父IRQ处理程序（最典型的是系统中断控制器）以链式方式被调用。这意味着GPIO irqchip处理程序将立即从父irqchip调用，同时保持IRQs禁用。然后GPIO irqchip将最终在其中断处理程序中像下面这个顺序调用函数：

    ```
    static irqreturn_t foo_gpio_irq(int irq, void *data);
    chained_irq_enter(...); 
    generic_handle_irq(...); 
    chained_irq_exit(...);
    ```

    链式GPIO irqchip一般不能设置结构体中的.can_sleep标志，因为所有的事都直接在发生在回调中，而不是像I2C这样的慢总线通信。

    实时注意事项：请注意，链式IRQ处理程序不会在-RT上强制线程化。因此，spinlock_t或任何可睡眠的API（如PM runtime）不能在链式IRQ处理程序中使用。

    如果需要的话，链式IRQ处理程序可以被转化为通用IRQ处理程序(如果不能转化的话，请参考下文)，通过这种方式它将在-RT上变为线程化的IRQ处理程序，在非-RT上变为硬IRQ处理程序(详情请见[3])。

    函数generic_handle_irq()应该在IRQ禁用的时候调用，当这个函数在被强制到线程的IRQ处理程序中被调用时，IRQ core会发出抱怨，fake？raw lock可以被用来解决这个问题：
    ```
    raw_spinlock_t wa_lock;
    static irqreturn_t omap_gpio_irq_handler(int irq, void *gpiobank)
            unsigned long wa_lock_flags;
            raw_spin_lock_irqsave(&bank->wa_lock, wa_lock_flags);
            generic_handle_irq(irq_find_mapping(bank->chip.irq.domain, bit));
            raw_spin_unlock_irqrestore(&bank->wa_lock, wa_lock_flags);
    ```
+ 通用链式GPIO IRQCHIP(GENERIC CHAINED GPIO IRQCHIPS)：这与“链式GPIO irqchips”相同，但是不使用链式IRQ处理程序。相反，GPIO IRQ是由通用IRQ处理程序执行的，通用IRQ处理程序是由request_irq()配置的。然后GPIO irqchip将最终在其中断处理程序中像下面这个顺序调用函数：
    ```
    static irqreturn_t gpio_rcar_irq_handler(int irq, void *dev_id)
        for each detected GPIO IRQ
            generic_handle_irq(...);
    ```
    实时(Realtime)注意事项：这种类型的中断处理程序将会在-RT上被强制线程化，因此当在IRQ启用时调用generic_handle_irq()，IRQ core会发出抱怨。可以采用和“链式GPIO irqchips”相同的变通方法。

+ 嵌套的线程化GPIO irqchips(NESTED THREADED GPIO IRQCHIPS)：这是片外GPIO扩展器，或是其他位于休眠总线另一旁的GPIO irqchip，比如I2C和SPI。

    当然这些驱动需要慢速总线通信来读出IRQ状态和其他类似的东西，这些通信可能会导致其他的IRQ发生，因此当IRQ禁用时，不能在快速IRQ中断处理程序中进行处理。相反地，它们需要生成一个线程，然后屏蔽父IRQ line，直到驱动程序处理中断为止。这个驱动程序的特点是在它的中断处理程序中调用类似的东西：
    ```
    static irqreturn_t foo_gpio_irq(int irq, void *data)
    ...
    handle_nested_irq(irq);
    ```
    线程化GPIO irqchips的特点是它们将struct GPIO芯片上的.can_sleep标志设置为true，这表明该芯片在访问GPIO时可能会休眠。

    这类irqchips天生就具有实时性，因为它们已经就是被生成用来处理睡眠上下文的。

# Infrastructure helpers for GPIO irqchips
用于帮助设置和管理GPIO irqchips、相关irqdomain和资源分配回调函数。通过选择Kconfig symbol GPIOLIB_IRQCHIP来激活IRQCHIP。如果同时选择了IRQ_DOMAIN_HIERARCHY ，则会提供层次结构帮助程序。大部分的代码开销由gpiolib提供，前提是中断和GPIO line之间的映射是1对1的：
```
GPIO line offset	Hardware IRQ
0	0
1	1
2	2
…	…
ngpio-1	ngpio-1
```
如果有些GPIO line没有相应额IRQ，那么gpio_irq_chip中的位掩码valid_mask和标志need_valid_mask来屏蔽某些line，使其与irq关联无效。

设置帮助程序的首先方式是，在添加gpio_chip之前，填写gpio_chip中的结构体gpio_irq_chip。如果你这样做，gpiolib将在设置剩余GPIO功能的同时设置额外的irq_chip。以下是一个使用gpio_irq_chip的级联中断处理程序的典型例子：
```
/* Typical state container with dynamic irqchip */
struct my_gpio {
    struct gpio_chip gc;
    struct irq_chip irq;
};

int irq; /* from platform etc */
struct my_gpio *g;
struct gpio_irq_chip *girq;

/* Set up the irqchip dynamically */
g->irq.name = "my_gpio_irq";
g->irq.irq_ack = my_gpio_ack_irq;
g->irq.irq_mask = my_gpio_mask_irq;
g->irq.irq_unmask = my_gpio_unmask_irq;
g->irq.irq_set_type = my_gpio_set_irq_type;

/* Get a pointer to the gpio_irq_chip */
girq = &g->gc.irq;
girq->chip = &g->irq;
girq->parent_handler = ftgpio_gpio_irq_handler;
girq->num_parents = 1;
girq->parents = devm_kcalloc(dev, 1, sizeof(*girq->parents),
                             GFP_KERNEL);
if (!girq->parents)
    return -ENOMEM;
girq->default_type = IRQ_TYPE_NONE;
girq->handler = handle_bad_irq;
girq->parents[0] = irq;

return devm_gpiochip_add_data(dev, &g->gc, g);
```
帮助程序也支持使用分层中断控制器。在这种情况下，典型设置如下：
```
/* Typical state container with dynamic irqchip */
struct my_gpio {
    struct gpio_chip gc;
    struct irq_chip irq;
    struct fwnode_handle *fwnode;
};

int irq; /* from platform etc */
struct my_gpio *g;
struct gpio_irq_chip *girq;

/* Set up the irqchip dynamically */
g->irq.name = "my_gpio_irq";
g->irq.irq_ack = my_gpio_ack_irq;
g->irq.irq_mask = my_gpio_mask_irq;
g->irq.irq_unmask = my_gpio_unmask_irq;
g->irq.irq_set_type = my_gpio_set_irq_type;

/* Get a pointer to the gpio_irq_chip */
girq = &g->gc.irq;
girq->chip = &g->irq;
girq->default_type = IRQ_TYPE_NONE;
girq->handler = handle_bad_irq;
girq->fwnode = g->fwnode;
girq->parent_domain = parent;
girq->child_to_parent_hwirq = my_gpio_child_to_parent_hwirq;

return devm_gpiochip_add_data(dev, &g->gc, g);
```
如你所见这两种方式非常的相似，但是你不需要为IRQ提供一个parent handler，取而代之的是一个parent irqdomian，一个硬件的fwnode和一个函数。child_to_parent_hwirq()用于从一个子（也就是这个gpio_chip）硬件irq查找父硬件irq。通常通过查看内核树种的实例以获得有关如何找到所需片段的建议。

旧的在gpio_chip注册之后添加irqchip的方式仍然是可用的，但是我们试图摆脱这一点：
+ 淘汰的：gpiochip_irqchip_add()把链式级联irqchip添加到一个gpiochip中。它会把芯片的struct gpio_chip*传递给所有IRQ回调函数，所以回调函数需要把gpio_chip嵌入到它自身的state container中，并且使用container_of()获得一个指向container的指针。
+ gpiochip_irqchip_add_nested():添加一个嵌套的级联irqchip到一个gpiochip中，如上面关于不同类型的级联irqchip所讨论的。级联irq必须由线程化的中断处理函数处理。除此之外，它的工作原理和链式irqchip完全一样。
+ gpiochip_set_nested_irqchip()：为一个来自父级IRQ的gpio_chip设置一个嵌套级联的irq处理函数。因为父级IRQ通常由驱动程序显式请求，这除了将所有子IRQ标记为具有一个父IRQ之外，没有其他的作用。

如果需要从这些帮助程序处理的IRQ域中排除某些GPIO line，我们可以在devm_gpiochip_add_data()和gpiochip_add_data() 被调用之前设置gpiochip的.irq.need_valid_mask。它将分配一个.irq.valid_mask，其设置的位数与芯片中GPIO line的位数相同，每个位代表line 0...n-1。驱动程序可以通过清除这个掩码中的位来排除GPIO line。这个掩码必须在gpiochip_irqchip_add() 或gpiochip_irqchip_add_nested()调用之前设置。

使用帮助程序的时候请注意以下几点：
+ 确保指定了与struct gpio_chip相关的成员，以便irq_chip可以初始化。比如：.dev和.can_sleep应该被正确的设置。
+  Nominally set all handlers to handle_bad_irq() in the setup call and pass handle_bad_irq() as flow handler parameter in gpiochip_irqchip_add() if it is expected for GPIO driver that irqchip .set_type() callback will be called before using/enabling each GPIO IRQ. Then set the handler to handle_level_irq() and/or handle_edge_irq() in the irqchip .set_type() callback depending on what your controller supports and what is requested by the consumer.

# Locking IRQ usage
因为GPIO和irq_chip是独立的，我可以发现发现不同用例之间的冲突。例如一个用于IRQ的GPIO line必须要是输入的，因为在一个输出 line上触发中断时没意义的。

如果子系统内部存在竞争，哪一方正在使用资源（例如某个GPIO line和寄存器），就需要在gpiolib子系统内部拒绝某些操作并跟踪使用情况。

输入GPIO可以作为IRQ信号。发生这种情况时，驱动程序必须标记GPIO为用作IRQ:
```
int gpiochip_lock_as_irq(struct gpio_chip *chip, unsigned int offset)
```
这会防止使用non-irq相关的GPIO API，直到GPIO IRQ锁被释放：
```
void gpiochip_unlock_as_irq(struct gpio_chip *chip, unsigned int offset)
```
当我们在GPIO驱动内声明一个irqchip时，这两个函数一般需要在irqchip中的.startup和shutdown回调函数内被调用。

当使用gpiolib irqchip 帮助程序的时候，回调会自动被指派。

# Real-Time compliance for GPIO IRQ chips
任何 irqchips 的提供者都需要精心定制以支持实时抢占。GPIO子系统中的所有irqchips都应该牢记这一点，并进行适当的测试，以确保它们是实时的。

所以请注意上面文档中的实时注意事项。

以下是在准备一个实时驱动程序时需要遵循的检查表：
+ 确保spinlock_t没有irq_chip声明中使用
+ 确保没有在irq_chip中使用可睡眠的API，如果必须使用可睡眠的API，可以通过使用.irq_bus_lock()和.irq_bus_unlock()回调函数。
+ 链式GPIO irqchips：确保spinlock_t和可睡眠的API没有在链式IRQ处理程序中使用。
+ 通用链式GPIO irqchips：注意 generic_handle_irq()的调用，并采用相应的解决方案。
+ 链式GPIO irqchips：如果可能尽量使用通用IRQ处理程序，而不是链式IRQ处理程序
+ regmap_mmio：可以通过设置.disable_locking并且处理GPIO驱动中的锁老禁用regmap中内部锁。
+ 使用适当的内核实时测试用例测试您的驱动程序，包括电平和边缘irq

# Requesting self-owned GPIO pins
有时候，允许GPIO芯片驱动通过gpiolib API来请求自己的GPIO描述符是很有用的。GPIO驱动可以使用以下函数来请求和释放描述符。
```
struct gpio_desc *gpiochip_request_own_desc(struct gpio_desc *desc,
                                            u16 hwnum,
                                            const char *label,
                                            enum gpiod_flags flags)

void gpiochip_free_own_desc(struct gpio_desc *desc)
```
有gpiochip_request_own_desc()请求的描述符必须通过gpiochip_free_own_desc()来释放。

这些函数必须小心使用，因为它们不会影响模块使用计数。不要使用函数请求不属于调用驱动程序的gpio描述符。

+ [1] http://www.spinics.net/lists/linux-omap/msg120425.html
+ [2] https://lkml.org/lkml/2015/9/25/494
+ [3] https://lkml.org/lkml/2015/9/25/495