PINCTRL 子系统<br>
这个文档概述了Linux的Pin control子系统<br>
这个子系统处理以下事情：
+ 枚举和命名Pins
+ Pins、pads、fingers等的复用，详情请见下文
+ Pins、pads、finger等的电气配置，例如上下拉、开漏等

# Top-level interface
PIN CONTROLLER的定义：<br>
+ Pin controller是一个硬件，通常是一组用来控制pins的寄存器。它们可以用来为一个pins或一组pins配置复用、偏置、下上拉等。
PIN的定义<br>
+ PINS就是pads、fingers等一切你想要控制的input/output line。PINS用一个从0-maxpins的整数表示。这个number space(0-maxpins)取决于PIN CONTROLLER，所以在一个系统里可能有多个number space。这个pin space不一定是连续的，它们中间可能有一些pin是不存在的。

当一个PIN CONTROLLER被实例化之后，它将注册一个descriptor（描述符）到pin control 框架中，这个PIN CONTROLLER描述符包含一组pin描述符，这些pin描述符由这个pin controller处理。

下面是一个芯片的例子：
```
        A   B   C   D   E   F   G   H

   8    o   o   o   o   o   o   o   o

   7    o   o   o   o   o   o   o   o

   6    o   o   o   o   o   o   o   o

   5    o   o   o   o   o   o   o   o

   4    o   o   o   o   o   o   o   o

   3    o   o   o   o   o   o   o   o

   2    o   o   o   o   o   o   o   o

   1    o   o   o   o   o   o   o   o
```
想要注册一个pin controller并且命名这个芯片上的所有pin，我们可以在驱动中这样写：
```
#include <linux/pinctrl/pinctrl.h>

const struct pinctrl_pin_desc foo_pins[] = {
      PINCTRL_PIN(0, "A8"),
      PINCTRL_PIN(1, "B8"),
      PINCTRL_PIN(2, "C8"),
      ...
      PINCTRL_PIN(61, "F1"),
      PINCTRL_PIN(62, "G1"),
      PINCTRL_PIN(63, "H1"),
};

static struct pinctrl_desc foo_desc = {
	.name = "foo",
	.pins = foo_pins,
	.npins = ARRAY_SIZE(foo_pins),
	.owner = THIS_MODULE,
};

int __init foo_probe(void)
{
	int error;

	struct pinctrl_dev *pctl;

	error = pinctrl_register_and_init(&foo_desc, <PARENT>, NULL, &pctl);
	if (error)
		return error;

	return pinctrl_enable(pctl);
}
```

想要启用pinctrl子系统、PINMUX和PINCONF的subgroups以及选定的驱动，你需要在你的芯片的Kconfig条目中选择他们，因为他们通常是与你的芯片紧密联系的。可以看看arch/arm/mach-u300/Kconfig作为例子。<br>

pins name通常和上面描述的有所不同，你可以在芯片的datasheet中找到它们。注意，pinctrl core中的pinctrl.h提供一个叫做PINCTRL_PIN()的宏用于创建pinctrl_pin_desc结构体。你可以看到我用0~63枚举了从A8到H1的所有引脚。在实际使用中，你必须考虑这些数字所代表的pin，使它们可以与你所用的芯片的寄存器布局相符合，否则代码会变得很复杂。你还需要考虑offset是否和pin controller所控制的GPIO ranges符合。

# Pin groups
一些控制器需要处理一组pins，所以pinctrl提供一个用于枚举和检索一组pin的机制。<br>

以一组用于SPI的pins{0,8,16,24},和一组用于I2C的pin{24,25}作为例子。<br>

这两组pin可以在pinctrl中通过实例化一个pinctrl_ops来表示：

```
#include <linux/pinctrl/pinctrl.h>

struct foo_group {
	const char *name;
	const unsigned int *pins;
	const unsigned num_pins;
};

static const unsigned int spi0_pins[] = { 0, 8, 16, 24 };
static const unsigned int i2c0_pins[] = { 24, 25 };

static const struct foo_group foo_groups[] = {
	{
		.name = "spi0_grp",
		.pins = spi0_pins,
		.num_pins = ARRAY_SIZE(spi0_pins),
	},
	{
		.name = "i2c0_grp",
		.pins = i2c0_pins,
		.num_pins = ARRAY_SIZE(i2c0_pins),
	},
};


static int foo_get_groups_count(struct pinctrl_dev *pctldev)
{
	return ARRAY_SIZE(foo_groups);
}

static const char *foo_get_group_name(struct pinctrl_dev *pctldev,
				       unsigned selector)
{
	return foo_groups[selector].name;
}

static int foo_get_group_pins(struct pinctrl_dev *pctldev, unsigned selector,
			       const unsigned **pins,
			       unsigned *num_pins)
{
	*pins = (unsigned *) foo_groups[selector].pins;
	*num_pins = foo_groups[selector].num_pins;
	return 0;
}

static struct pinctrl_ops foo_pctrl_ops = {
	.get_groups_count = foo_get_groups_count,
	.get_group_name = foo_get_group_name,
	.get_group_pins = foo_get_group_pins,
};


static struct pinctrl_desc foo_desc = {
       ...
       .pctlops = &foo_pctrl_ops,
};
```

Pin controller 子系统可以通过调用. get_groups_count()函数来获取group的数量，然后pin controller就可以通过调用foo_get_group_name()和foo_get_group_pins()来获取相应group的名字和其包含的pins。实际group的数据结构取决于驱动，这仅仅是个简单的例子。

# Pin configuration
Pin可以通过多种方式进行配置，大部分是作为inputs和outputs时的电气特性。举个例子，你想要配置某个pin为高阻抗，或是你想要用将一个pin配置为上/下拉。<br>

可以通过添加条目到映射表来进行配置pin configuration，详情请见以下Board/machine configuration的部分。<br>

配置参数的格式和含义，是由pin controller驱动定义的。例如PLATFORM_X_PULL_UP<br>

Pin configuration驱动在pin controller中实现了设置引脚配置的回调函数,像下面这样：
```
#include <linux/pinctrl/pinctrl.h>
#include <linux/pinctrl/pinconf.h>
#include "platform_x_pindefs.h"

static int foo_pin_config_get(struct pinctrl_dev *pctldev,
		    unsigned offset,
		    unsigned long *config)
{
	struct my_conftype conf;

	... Find setting for pin @ offset ...

	*config = (unsigned long) conf;
}

static int foo_pin_config_set(struct pinctrl_dev *pctldev,
		    unsigned offset,
		    unsigned long config)
{
	struct my_conftype *conf = (struct my_conftype *) config;

	switch (conf) {
		case PLATFORM_X_PULL_UP:
		...
		}
	}
}

static int foo_pin_config_group_get (struct pinctrl_dev *pctldev,
		    unsigned selector,
		    unsigned long *config)
{
	...
}

static int foo_pin_config_group_set (struct pinctrl_dev *pctldev,
		    unsigned selector,
		    unsigned long config)
{
	...
}

static struct pinconf_ops foo_pconf_ops = {
	.pin_config_get = foo_pin_config_get,
	.pin_config_set = foo_pin_config_set,
	.pin_config_group_get = foo_pin_config_group_get,
	.pin_config_group_set = foo_pin_config_group_set,
};

/* Pin config operations are handled by some pin controller */
static struct pinctrl_desc foo_desc = {
	...
	.confops = &foo_pconf_ops,
};
```

因为有些芯片有配置一整组pin的特殊逻辑，则它们可以通过特殊的回调函数来配置一整组pin。对于其不想处理的组，pin_config_group_set()函数可以返回错误-EAGAIN。或者其只想做些组级的处理，然后迭代处理所有的pin，这种情况下每个pin可以通过单独调用pin_config_set()进行配置。

# Interaction with the GPIO subsystem
GPIO驱动可能需要在同一个pin上执行多种操作，并且这个pin已经在pin controller中注册。<br>

首先最重要的，这两个子系统可以完全独立使用，详情请见下面的pin control requests from drivers和drivers needing both pin control and GPIOs部分。但是在一些情况下需要在pins和GPIO之前建立跨子系统的映射。<br>

因为pin controller子系统有着其本身的pinspace，这个pinspace是pin controller内部的，所以我们一个映射，这样pin control子系统就能知道哪个pin controller控制哪个GPIO。因为一个pin controller可能包含多个GPIO range（典型的SOC有一组pins，但是其内部有多个GPIO blocks，每个GPIO blocks被抽象成一个gpio_chip），因此任意数量的GPIO range可以被添加到pin controller实例，像下面这样：<br>
```
struct gpio_chip chip_a;
struct gpio_chip chip_b;

static struct pinctrl_gpio_range gpio_range_a = {
	.name = "chip a",
	.id = 0,
	.base = 32,
	.pin_base = 32,
	.npins = 16,
	.gc = &chip_a;
};

static struct pinctrl_gpio_range gpio_range_b = {
	.name = "chip b",
	.id = 0,
	.base = 48,
	.pin_base = 64,
	.npins = 8,
	.gc = &chip_b;
};

{
	struct pinctrl_dev *pctl;
	...
	pinctrl_add_gpio_range(pctl, &gpio_range_a);
	pinctrl_add_gpio_range(pctl, &gpio_range_b);
}
```
如上，这个系统由一个pin controller控制两个不同的gpio chip。其中chip_a包含16个pins，chip_b包含8个pins。chip_a和chip_b有着不同的.pin_base。.pin_base表示一个gpio range起始的pin number。<br>

chip_a的GPIO range从base 32开始，并且实际的pin range也是从32开始的。而chip_b的GPIO range是从48开始的，pin range是从64开始的。<br>

我们可以通过pin_base将gpio number转化成实际的pin number。
他们在全局GPIO pin空间的映射是：
```
chip a:
 - GPIO range : [32 .. 47]
 - pin range  : [32 .. 47]
chip b:
 - GPIO range : [48 .. 55]
 - pin range  : [64 .. 71]
 ```
如果上面的映射假设gpio和pins的关系式线性的。如果映射不是线性的，一个任意的pins列表可以像下面这样组合起来进行映射:<br>
```
static const unsigned range_pins[] = { 14, 1, 22, 17, 10, 8, 6, 2 };

static struct pinctrl_gpio_range gpio_range = {
        .name = "chip",
        .id = 0,
        .base = 32,
        .pins = &range_pins,
        .npins = ARRAY_SIZE(range_pins),
        .gc = &chip;
};
```
在这种情况下，pin_base属性会被忽略。如果pin group的名字是已知的，上面结构体的pin和npins元素，可以使用函数pinctrl_get_group_pins()来进行对pin group“foo”来进行初始化。<br>
```
pinctrl_get_group_pins(pctl, "foo", &gpio_range.pins,
                       &gpio_range.npins);
```
当pin control子系统中与gpio相关的函数被调用时，这个组合将用来查找合适的pin controller,通过检查pin并将其与所有pin controller的pin范围相匹配的方式来进行查找。当一个pin controller处理的pin范围与这个组合相匹配。GPIO相关的函数将会在指定的pin controller上被调用。<br>

对于所有涉及引脚偏置和复用等的操作，pin controller将会在传入的gpio编号中查找相符的pin编号，并且在组合的内部查找pin编号。在那之后pin controller子系统将把它传递给pin control驱动，这样驱动就会得到pin编号。而且，它还会传递范围的ID编号，这样pin controller就能知道他要处理的是哪一个pin范围。<br>

从pinctrl 驱动中调用函pinctrl_add_gpio_range是淘汰的。请查看2.1小节 documentation/devicetree/bindings/gpio/gpio.txt查看如何把pinctrl和gpio驱动绑定。<br>


# PINMUX interfaces

这些调用使用pinmux_作为前缀，其他的调用不会使用这个前缀。

## 什么是引脚复用？
Pinmux也被称为padmux，ballmux,复用功能是芯片制造商在设计芯片时在某个引脚上实现多个互斥功能的一种方式，这取决于应用。在此上下文中，应用通常是指在电子系统中的封装和布线，这个框架使得在运行时也能够改变封装功能。<br>

下面是一个GPA封装的芯片的例子：<br>
```
      A   B   C   D   E   F   G   H
   +---+
8  | o | o   o   o   o   o   o   o
   |   |
7  | o | o   o   o   o   o   o   o
   |   |
6  | o | o   o   o   o   o   o   o
   +---+---+
5  | o | o | o   o   o   o   o   o
   +---+---+               +---+
4    o   o   o   o   o   o | o | o
                           |   |
3    o   o   o   o   o   o | o | o
                           |   |
2    o   o   o   o   o   o | o | o
   +-------+-------+-------+---+---+
1  | o   o | o   o | o   o | o | o |
   +-------+-------+-------+---+---+
```
这不是俄罗斯方块，更像是国际象棋。不是所有的PGA/BGA封装都是棋盘式的，但我们将这个作为最简单的一个例子。在你看到这些引脚之中，有一些会被像是VCC和GND等给芯片供电的东西占用，还有不少会被像是外部存储器这样的大接口占用。剩下的引脚往往会被作为引脚复用。<br>

上面8x8的PGA封装示例讲pin编号0~63分别指向其物理引脚。使用pinctrl_register_pins()，和合适的数据集来来命名引脚｛A1,A2,A3…H6,H7,H8｝，如前所述。<br
>
在这个8x8的BGA封装中，引脚｛A8,A7,A6,A5｝可以被用作SPI端口(CLK,RXD,TXD,FRM)。这个例子中，B5引脚可以被用作普通IO口。然而在另一中配置中，引脚{A5，B5}可以作为I2C端口使用（SCL,SDA）。不用说，我们不能同时使用SPI和I2C。然而在封装内部，执行执行SPI逻辑的电路可以选择性的切换到{G4,G3,G2,G1}输出。<br>

在最后一排{ A1, B1, C1, D1, E1, F1, G1, H1 } ，有一个特殊的东西——一个外部MMC总线，他可以被设置为2/4/8位宽，分别占用2/4/8个引脚，所以要么使用 { A1, B1 ，要么使用 { A1, B1, C1, D1 } ，要么使用最后一排所有的引脚。如果我们使用8位模式，那么就自然不能使用 { G4, G3, G2, G1 } 。<br>

这样芯片内部的电路可以在不同的引脚范围(pin range)复用“muxed”.通常现代的SOC往往包含多个I2C,SPI,SDIO/MMC等等接口，通过引脚复用的配置可以将这些接口配置到不同的引脚上。<br>
>
因为GPIO接口通常是短缺的，所以如果当前没有其他I/O端口占用,那么通常任何引脚都可以作为GPIO。<br>


## Pinmux conventions
Pinctrl子系统中Pinmux功能的目的是为了抽象machine配置中选择实例化的设备，并为这个设备提供引脚复用设置。它受到CLK、GPIO和regulator子系统的启发，所以设备将会请求它们的引脚复用设定，但同时可以请求单个引脚，例如作为GPIO。<br>

定义：<br>
+ 功能（FUNCTIONS）可以通过drivers/pinctrl/*目录下的pinctrl子系统中的驱动来进行切换。Pinctrl驱动知道可选的功能(FUNCTIONS)。在上面的例子中，你可以定义三个引脚复用功能(Pinmux functions)，SPI、I2C、MMC。
+ 假设功能(FUNCTIONS)在一个一维的数组中从0开始枚举。这种情况下，这个数组可以被定成这样{ spi0, i2c0, mmc0 }。
+ 功能(FUNCTIONS)具有定义在通用层面上的引脚组(PIN GROUP)，所以某个功能(FUNCTIONS)通常与特定的引脚组(PIN GROUP)相关联，可以是单个引脚组也可以是多个引脚组。在上面的例子中，I2C功能和引脚  { A5, B5 }相关联，{ A5, B5 }在Pin space里被枚举为 { 24, 25 } 。<br>
SPI功能(FUNCTIONS)和引脚组(PIN GROUP)  { A8, A7, A6, A5 } 和 { G4, G3, G2, G1 }相关联，它们分别被枚举为编号 { 0, 8, 16, 24 } 和 { 38, 46, 54, 62 }。<br>
每个Pinctrl上的引脚组(PIN GROUP)的名字必须是唯一的，同一个Pinctrl上的两个引脚组名字不能相同。
+ 功能(FUNCTION)和引脚(PIN GROUP)共同决定了某组引脚的某种功能。功能(FUNCTION)、引脚(PIN GROUP)和机器(MACHINE)特定的细节是被封装在驱动中的，从外面只知道枚举器，并且驱动可以请求：
    + 特定编号(Selector)(>=0)对应的功能(FUNCTIONS)名
    + 某个功能(FUNCTIONS)对应的PIN GROUP列表
    + 列表中某个GROUP要激活的某个功能(FUNCTION)<br>
上文已经介绍过，引脚组(PIN GROUP)是自己定义的，所以pinctrl核心会从驱动中查找引脚组(PIN GROUP)实际的引脚范围(pin range)<br>

+ 某个pinctrl上的功能(FUCNTIONS)和引脚组(PIN GROUP)是通过board file，设备树，或是其他相似的机器配置机制映射到某个设备上的。这样只需定义一个Pinctrl、functions和group，就可以唯一确定某个设备要使用的引脚。（如果某个功能只对应一个引脚组，则无需提供组名，core将选择唯一一个可用的组）。<br>
在下面的例子中，我们定义这个特定的机器在第一个pinctrl(pinctrl0)上,设备spi0对应复用功能(FUNCTION) fspi0和引脚组(GROUP)gspi0，设备i2c0对应功能fi2c0 和引脚组gi2c0。我们可以这样映射：
    ```
    {
            {"map-spi0", spi0, pinctrl0, fspi0, gspi0},
            {"map-i2c0", i2c0, pinctrl0, fi2c0, gi2c0}
    }
    ```
    每个映射必须分配一个状态(STATE)名,一个pinctrl，一个设备，一个功能(FUNCTION)。引脚组不是必须的，如果省略了引脚组，则会选择驱动提供的第一个适用于该功能的引脚组，在简单的使用中这非常有用。<br>
    可以将同一device、pinctrl和function的组合映射到多个group。这是针对同一个pinctrl上的同一个function可能在不同的配置中要使用不同引脚。

+ 在某个pinctrl上使用某个引脚组(PIN GROUP)的某个功能(FUNCTION)是依据先到先得的方式提供的。所以如果其他设备的复用配置或是GPIO请求已经占用了某个物理引脚，则你讲无法使用它。想要激活一个新的设定，就必须先停用旧的设定。

有时文档和硬件寄存器会使用pad或是finger，而不是pins来进行编写。只选择对你那些对你有用的引脚进行枚举。<br>

假设：<br>
我们假设功能(FUNCTION)映射到引脚组(PIN GROUP)的数量是受硬件限制的。也就是说假设不存在任何功能都能映射到任何引脚组的系统。所以某个功能(FUNCTION)可用的引脚组(PIN GROUP)是被限制在几个选择之内的（比如最多八个左右）而不是几百个或任何数量的选择。这个特性是通过检查现有的硬件发现的，这也是一个必要的假设，因为我们希望pinmux驱动可以向子系统提供所有可能的功能(FUCNTIONS)和引脚组映射(PIN GROUP)。

## Pinmux drivers
Pinmux核心负责防止引脚上的冲突，并且调用pinctrl驱动来执行不同的配置。<br>
Pinmux驱动有责任施加进一步的限制(比如由负载造成的电气上的限制)，来决定是否允许所请求的功能(FUNCTION)，如果所请求的复用设置被允许，就对硬件进行设置。<br>
Pinmux驱动需要提供一些回调函数，有些回调函数是可选的。通常需要实现set_mux()函数，将值写进某个寄存器，以激活某个引脚的特性复用功能。<br>
一个简单的例子来实现上面的例子，通过设置bit 0,1,2,3或4到一些名为mux的寄存器，来使某个引脚组(PIN GROUP)上的特定功能(FUNCTION)能够被使能：<br>
```
#include <linux/pinctrl/pinctrl.h>
#include <linux/pinctrl/pinmux.h>

struct foo_group {
        const char *name;
        const unsigned int *pins;
        const unsigned num_pins;
};

static const unsigned spi0_0_pins[] = { 0, 8, 16, 24 };
static const unsigned spi0_1_pins[] = { 38, 46, 54, 62 };
static const unsigned i2c0_pins[] = { 24, 25 };
static const unsigned mmc0_1_pins[] = { 56, 57 };
static const unsigned mmc0_2_pins[] = { 58, 59 };
static const unsigned mmc0_3_pins[] = { 60, 61, 62, 63 };

static const struct foo_group foo_groups[] = {
        {
                .name = "spi0_0_grp",
                .pins = spi0_0_pins,
                .num_pins = ARRAY_SIZE(spi0_0_pins),
        },
        {
                .name = "spi0_1_grp",
                .pins = spi0_1_pins,
                .num_pins = ARRAY_SIZE(spi0_1_pins),
        },
        {
                .name = "i2c0_grp",
                .pins = i2c0_pins,
                .num_pins = ARRAY_SIZE(i2c0_pins),
        },
        {
                .name = "mmc0_1_grp",
                .pins = mmc0_1_pins,
                .num_pins = ARRAY_SIZE(mmc0_1_pins),
        },
        {
                .name = "mmc0_2_grp",
                .pins = mmc0_2_pins,
                .num_pins = ARRAY_SIZE(mmc0_2_pins),
        },
        {
                .name = "mmc0_3_grp",
                .pins = mmc0_3_pins,
                .num_pins = ARRAY_SIZE(mmc0_3_pins),
        },
};


static int foo_get_groups_count(struct pinctrl_dev *pctldev)
{
        return ARRAY_SIZE(foo_groups);
}

static const char *foo_get_group_name(struct pinctrl_dev *pctldev,
                                unsigned selector)
{
        return foo_groups[selector].name;
}

static int foo_get_group_pins(struct pinctrl_dev *pctldev, unsigned selector,
                        unsigned ** const pins,
                        unsigned * const num_pins)
{
        *pins = (unsigned *) foo_groups[selector].pins;
        *num_pins = foo_groups[selector].num_pins;
        return 0;
}

static struct pinctrl_ops foo_pctrl_ops = {
        .get_groups_count = foo_get_groups_count,
        .get_group_name = foo_get_group_name,
        .get_group_pins = foo_get_group_pins,
};

struct foo_pmx_func {
        const char *name;
        const char * const *groups;
        const unsigned num_groups;
};

static const char * const spi0_groups[] = { "spi0_0_grp", "spi0_1_grp" };
static const char * const i2c0_groups[] = { "i2c0_grp" };
static const char * const mmc0_groups[] = { "mmc0_1_grp", "mmc0_2_grp",
                                        "mmc0_3_grp" };

static const struct foo_pmx_func foo_functions[] = {
        {
                .name = "spi0",
                .groups = spi0_groups,
                .num_groups = ARRAY_SIZE(spi0_groups),
        },
        {
                .name = "i2c0",
                .groups = i2c0_groups,
                .num_groups = ARRAY_SIZE(i2c0_groups),
        },
        {
                .name = "mmc0",
                .groups = mmc0_groups,
                .num_groups = ARRAY_SIZE(mmc0_groups),
        },
};

static int foo_get_functions_count(struct pinctrl_dev *pctldev)
{
        return ARRAY_SIZE(foo_functions);
}

static const char *foo_get_fname(struct pinctrl_dev *pctldev, unsigned selector)
{
        return foo_functions[selector].name;
}

static int foo_get_groups(struct pinctrl_dev *pctldev, unsigned selector,
                        const char * const **groups,
                        unsigned * const num_groups)
{
        *groups = foo_functions[selector].groups;
        *num_groups = foo_functions[selector].num_groups;
        return 0;
}

static int foo_set_mux(struct pinctrl_dev *pctldev, unsigned selector,
                unsigned group)
{
        u8 regbit = (1 << selector + group);

        writeb((readb(MUX)|regbit), MUX)
        return 0;
}

static struct pinmux_ops foo_pmxops = {
        .get_functions_count = foo_get_functions_count,
        .get_function_name = foo_get_fname,
        .get_function_groups = foo_get_groups,
        .set_mux = foo_set_mux,
        .strict = true,
};

/* Pinmux operations are handled by some pin controller */
static struct pinctrl_desc foo_desc = {
        ...
        .pctlops = &foo_pctrl_ops,
        .pmxops = &foo_pmxops,
};
```
在这个例子中，激活muxing 0和1的同时设置bit 0和1，使用了一个共同的引脚，所以它们会发生冲突。<br>

Pinmux子系统的好处在于，它一直跟踪所有引脚以及谁在使用他们，所以他会拒绝这样一个不可能的请求，所以驱动不需要担心，当它得到一个传进来的selector的时候，pinmux子系统会确保没有其他设备或是GPIO分配已经在使用所选的引脚。<br>

因此控制寄存器的0和1永远不会同时置位。<br>

以上的所有功能都是pinmux驱动必须实现的。<br>

## Pin control interaction with the GPIO subsystem
注意，下面的内容意味着使用<linux/gpio.h>中的API与gpio_request()和类似的函数来使用Linux内核的某个引脚。<br>

在一些情况下，你可能会使用一些在datasheet中被称为”GPIO mode“的东西，当实际上它只是某个设备的电气配置。请参考下的“GPIO mode pitfalls”一节。<br>

公共Pinmux API包含两个功能pinctrl_gpio_request()和pinctrl_gpio_free()。这两个函数只能被基于gpiolib的驱动调用。pinctrl_gpio_request()和pinctrl_gpio_free()作为函数gpio_request() 和gpio_free()的一部分。同样的pinctrl_gpio_direction_input()和pinctrl_gpio_direction_output()也只能在gpiolib中的gpio_direction_input()和gpio_direction_output()实现的时候被调用。<br>

注意，平台驱动和单独驱动不应该直接请求GPIO引脚被控制，例如复用输入。相反地，应该实现一个gpiolib驱动，并通过gpiolib驱动中为引脚请求适当的复用功能和其他的控制。<br>

功能列表可能会变得很长，特别是如果你将每个单独的引脚转化为独立于其他引脚的GPIO引脚，并且尝试将每个引脚定义为一个功能(FUNCTION)。<br>

在这种情况下，function数组将变成64个条目，包含每个GPIO设置和其设备function。<br>

出于这种原因，pinctrl驱动可以实现两个函数，以在单个引脚上使能GPIO：.gpio_request_enable() 和gpio_disable_free()。<br>

这两个函数将传入由pinctrl core定义的GPIO range，这样你就知道哪些GPIO pin受request操作影响。<br>

如果你的驱动需要从框架中指示是将GPIO pin用作输入还是输出，则可以实现gpio_set_direction()函数。如上所述，该函数将需要从gpiolib驱动调用，并且受影响的GPIO 范围(range)、引脚偏移(offset)和所需的方向(输入/输出)将被传递给该函数。<br>

除了使用这些特殊的函数，完全可以使用对每个GPIO引脚使用定义的功能(function)，函数pinctrl_gpio_request() 将尝试获取功能””gpioN，其中“N”是全局GPIO 引脚号，如果没有特殊的GPIO-handler的话。<br>

# GPIO mode pitfalls
由于硬件工程的命名习惯，“GPIO”的含义与内核中的定义不同，开发者可能会对datasheet中的将一个引脚设置成”GPIO mode”的说法感到困惑。看来硬件工程师所说的“GPIO mode”的含义不一定是内核接口<linux/gpio.h>中提及的：你从内核代码中获取一个pin，然后监听输入、或是将其置为高/低电平。<br>

而硬件工程认为”GPIO mode”意味着你可以通过软件控制的一些引脚电气特性。而如果引脚处于其他状态，比如复用输入，则将无法控制这些电气特性。<br>

引脚的GPIO部分，及其相关的某个pinctrl配置和复用逻辑之间的关系可以通过多种方式构建。下面是两个例子:
```
 (A)
                  pin config
                  logic regs
                  |               +- SPI
Physical pins --- pad --- pinmux -+- I2C
                          |       +- mmc
                          |       +- GPIO
                          pin
                          multiplex
                          logic regs
```
有些配置是无论引脚是否用于GPIO都可以使用的。如果你将一个引脚复用为GPIO，你可以通过GPIO寄存器将其配置为高/低电平。或者，这个引脚由某个外设所控制，这种情况仍然可以应用所需的引脚配置属性。GPIO与其他使用该引脚的设备之间是互不影响的。<br>

在这种模式中，GPIO部分的寄存器或是GPIO硬件模块的寄存器可能存在于一个独立的内存范围内，这部分内存只用于GPIO驱动。而引脚配置和引脚复用所使用的寄存器存在另一个内存地址中，并且在datasheet中它们也是独立存在的。<br>

结构体pinmux_ops中的一个标志“strict”可用于检查和拒绝GPIO和引脚复用在此类硬件上对同一个引脚的同时访问。Pinctrl驱动应该相应设置该标志。<br>
```
(B)

                  pin config
                  logic regs
                  |               +- SPI
Physical pins --- pad --- pinmux -+- I2C
                  |       |       +- mmc
                  |       |
                  GPIO    pin
                          multiplex
                          logic regs
```
在这种模式中，GPIO功能可以始终被启用，当SPI/I2C/MMC输出电平信号时，GPIO输入可以用作监听该信号。这种模式下，在GPIO模块上执行错误的操作可能会破坏引脚上的通信，因为它从未真正断开。GPIO、引脚配置、引脚复用寄存器可能会被放在内存中的同一个范围，在datasheet中可能被放在同一部分中。<br>

在一些引脚控制器中，虽然物理引脚的设计和B相同，但是GPIO功能仍然不能与外设功能同时启用。所以在这种情况下“strict”标志也要被设置，以拒绝GPIO和引脚复用同时启用。<br>

从内核的角度来看，这些是硬件上的不同层面，应该被放进不同的子系统中：
1.	控制引脚电气属性（例如偏置和驱动强度）的寄存器，应该通过pinctrl子系统暴露出来，作为“pin configuration”设置。
2.	控制来自其他硬件模块(例如I2C、MMC、GPIO)的信号在引脚上复用的寄存器，这些寄存器应该通过pinctrl子系统暴露出来，作为复用功能(mux function)。
3.	那些控制GPIO功能的寄存器，例如GPIO输入寄存器、GPIO输出寄存器，和控制GPIO方向的寄存器，应该通过GPIO子系统暴露出来，如果它们还支持中断，则应该通过irqchip抽象。<br>

根据具体硬件寄存器设计，有些通过GPIO子系统暴露出来的函数可能需要调用pinctrl子系统，以便协调硬件模块之间的寄存器设置。特别是对于那些具有独立GPIO模块和pin控制模块的硬件来说可能是需要的，例如GPIO方向是由pin控制器中的寄存器配置的而不是GPIO模块。<br>

在所有情况下，引脚的电气属性(例如偏置和驱动强度)，都可以存放在那些引脚专用的寄存器上，或者是存放在GPIO寄存器上(特别是在例子B中)。这并不意味着这些属性一定与Linux内核所说的 "GPIO "有关。<br>

举个例子：一个引脚通常是作为UART的TX引脚的，但是在系统休眠的时候，我们需要将引脚配置为GPIO模式，并将其拉低。<br>

如果为这个引脚在GPIO子系统中做了一对一的映射，这时你可能觉得你要想的事非常复杂，因为这个引脚需要作为同时作为UART TX和GPIO，你会获取一个pinctrl handle并将它设置为某个状态以启用UART TX作为复用输入，之后在睡眠期间将其转化为gpio模式并且使用gpio_direction_output()函数来拉低这个引脚，然后在唤醒的时候将其重新复用成UART TX，甚至可能在这期间使用gpio_request/gpio_free。这一切都会变得很复杂。<br>

解决的方法是不要认为数据手册上所说的 "GPIO模式 "必须由<linux/gpio.h>接口来处理，而是把它看作是一个特定的引脚配置设置。例如在<linux/pinctrl/pinconf-generic.h>中查找，你会在文档中找到这个。<br>
```
PIN_CONFIG_OUTPUT
这将会引脚设置成输出，使用参数1来指明输出高电平，使用参数0来指明输出低电平
```

因此完全可以像平时使用pinctrl map一样，将”引脚设置为GPIO模式并输出低电平”，这个配置作为pinctrl map中的一项。作为例子，UART驱动可以像这样写：<br>
```
#include <linux/pinctrl/consumer.h>

struct pinctrl          *pinctrl;
struct pinctrl_state    *pins_default;
struct pinctrl_state    *pins_sleep;

pins_default = pinctrl_lookup_state(uap->pinctrl, PINCTRL_STATE_DEFAULT);
pins_sleep = pinctrl_lookup_state(uap->pinctrl, PINCTRL_STATE_SLEEP);

/* Normal mode */
retval = pinctrl_select_state(pinctrl, pins_default);
/* Sleep mode */
retval = pinctrl_select_state(pinctrl, pins_sleep);
```

### 并且机器的配置可能像下面这个样子：
```
static unsigned long uart_default_mode[] = {
        PIN_CONF_PACKED(PIN_CONFIG_DRIVE_PUSH_PULL, 0),
};

static unsigned long uart_sleep_mode[] = {
        PIN_CONF_PACKED(PIN_CONFIG_OUTPUT, 0),
};

static struct pinctrl_map pinmap[] __initdata = {
        PIN_MAP_MUX_GROUP("uart", PINCTRL_STATE_DEFAULT, "pinctrl-foo",
                "u0_group", "u0"),
        PIN_MAP_CONFIGS_PIN("uart", PINCTRL_STATE_DEFAULT, "pinctrl-foo",
                        "UART_TX_PIN", uart_default_mode),
        PIN_MAP_MUX_GROUP("uart", PINCTRL_STATE_SLEEP, "pinctrl-foo",
                "u0_group", "gpio-mode"),
        PIN_MAP_CONFIGS_PIN("uart", PINCTRL_STATE_SLEEP, "pinctrl-foo",
                        "UART_TX_PIN", uart_sleep_mode),
};

foo_init(void) {
        pinctrl_register_mappings(pinmap, ARRAY_SIZE(pinmap));
}
```
我们想要控制的引脚在“u0_group”中，并且有一个叫做“u0”的功能(FUNCTION)可以在这组引脚上启用，然后一切就像往常使用UART一样。另外还有一个名为“gpio-mode”的功能（FUNCTION）可以在应用在相同的引脚上，使这个引脚变为GPIO模式。<br>

这样就可以在不与GPIO子系统交互的情况，得到预期的效果。这只是设备在进入休眠状态时使用的电气属性，这个电气属性在datasheet中可能被称为GPIO mode。重点是，它仍可以被UART设备用来控制UART驱动中使用的引脚，使这些引脚进入UART需要的模式。Linux内核中的GPIO指的是1-bit的line，和这里说的GPIO不一样。<br>

如何将配置寄存器以实现push或是pull,输出低电平,以及如何在这些引脚上复用功能“u0”或是“gpio-mode”是驱动的事情。<br>

在一些datasheet中称“GPIO mode”为低功耗模式，而不是和GPIO有关的任何东西。从电气上来讲，这指的是同一件事情，但是如果称为低功耗模式，软件工程师很快就会发现这是一种特定的复用，或是是一种配置，而不是和GPIO API相关的东西。<br>

# Board/machine configuration
Board和Machine configuration定义了一个完整运行的系统是如何组合在一起的，包括GPIOs和设备是如何组合的，引脚复用是如何设置的。<br>

一个machine的pinctrl configuration看起来就像是简单的regulator configuration，因此我们可以像这样在上面例子的第二个fucntion映射中启用I2C和SPI：
```
#include <linux/pinctrl/machine.h>

static const struct pinctrl_map mapping[] __initconst = {
        {
                .dev_name = "foo-spi.0",
                .name = PINCTRL_STATE_DEFAULT,
                .type = PIN_MAP_TYPE_MUX_GROUP,
                .ctrl_dev_name = "pinctrl-foo",
                .data.mux.function = "spi0",
        },
        {
                .dev_name = "foo-i2c.0",
                .name = PINCTRL_STATE_DEFAULT,
                .type = PIN_MAP_TYPE_MUX_GROUP,
                .ctrl_dev_name = "pinctrl-foo",
                .data.mux.function = "i2c0",
        },
        {
                .dev_name = "foo-mmc.0",
                .name = PINCTRL_STATE_DEFAULT,
                .type = PIN_MAP_TYPE_MUX_GROUP,
                .ctrl_dev_name = "pinctrl-foo",
                .data.mux.function = "mmc0",
        },
};
```
这里的dev_name与唯一的设备名称相匹配，可以用来查找device结构体（就像clockdev和regulators一样）。功能(FUCNTION)的名字必须和此pin range的pinmux驱动中提供的功能(FUNCTION)相匹配。<br>

正如你所看到的系统上可能有多个pinctrl，因此我们需要指定哪一个pinctrl包含了我们需要映射的功能(FUNCTION)。<br>

你可以通过以下方式来将 pinmux mapping注册到pinmux子系统中
```
ret = pinctrl_register_mappings(mapping, ARRAY_SIZE(mapping));
```
因为上面的构造非常常见，因此有一个宏定义用来简化代码。举个例子，如果你想使用名为pinctrl-foo的pinctrl和位置0来映射，可以这样写：
```
static struct pinctrl_map mapping[] __initdata = {
        PIN_MAP_MUX_GROUP("foo-i2c.o", PINCTRL_STATE_DEFAULT,
                          "pinctrl-foo", NULL, "i2c0"),
};
```
映射表同样还包含pin configuration条目。 通常每个pin/group通常包含了多个configuration条目。所以配置表项引用了一个包含配置参数和值的数组。下面是使用宏来定义的映射表的例子：
```
static unsigned long i2c_grp_configs[] = {
        FOO_PIN_DRIVEN,
        FOO_PIN_PULLUP,
};

static unsigned long i2c_pin_configs[] = {
        FOO_OPEN_COLLECTOR,
        FOO_SLEW_RATE_SLOW,
};

static struct pinctrl_map mapping[] __initdata = {
        PIN_MAP_MUX_GROUP("foo-i2c.0", PINCTRL_STATE_DEFAULT,
                          "pinctrl-foo", "i2c0", "i2c0"),
        PIN_MAP_CONFIGS_GROUP("foo-i2c.0", PINCTRL_STATE_DEFAULT,
                              "pinctrl-foo", "i2c0", i2c_grp_configs),
        PIN_MAP_CONFIGS_PIN("foo-i2c.0", PINCTRL_STATE_DEFAULT,
                            "pinctrl-foo", "i2c0scl", i2c_pin_configs),
        PIN_MAP_CONFIGS_PIN("foo-i2c.0", PINCTRL_STATE_DEFAULT,
                            "pinctrl-foo", "i2c0sda", i2c_pin_configs),
};
```
最后，一些设备期望映射表包含某个特定的已命名状态。当在不需要任何pinctrl配置的硬件上运行时，映射表仍然必须包含这些状态，来明确表示提供了这些状态了，但是要将其置空。映射表中使用的宏定义 PIN_MAP_DUMMY_STATE用于定义一个状态，但是不会导致pinctrl被program。
```
static struct pinctrl_map mapping[] __initdata = {
        PIN_MAP_DUMMY_STATE("foo-i2c.0", PINCTRL_STATE_DEFAULT),
};
```

# Pin control requests from drivers
当一个设备驱动将要probe的时候，device core会自动尝试在这些设备上执行pinctrl_get_select_default()。通过这种方式编写驱动的人不需要添加下面提到的样板代码。然而在需要对设备进行状态(state)选择或是不使用“default”状态时，你必须在设备驱动上对pinctrl handle和state做些处理。<br>

因此，如果你只想将某个设备的引脚置于default state并对其进行一些处理，那么除了提供适合的映射表外，你什么也不需要做。device core将会负责剩余的部分。<br>

通常不鼓励让当个驱动程序获得并启用pinctrl。因此，可能的话在platform code中处理pinctrl，或是在其他可以访问到所有受影响的struct device *指针的地方处理pinctrl。在某些情况下，驱动需要在运行时切换不同的Mux映射，这是不可能的。<br>

一个典型的例子是，在运行时从PINCTRL_STATE_DEFAULT切换到PINCTRL_STATE_SLEEP ， 一个驱动需要将引脚的偏置从正常模式切换到睡眠状态，通过重新调整偏置甚至重新复用引脚来节省电量。<br>

一个驱动可能会请求激活某个state，通常是像这样的default state：
```
#include <linux/pinctrl/consumer.h>

struct foo_state {
struct pinctrl *p;
struct pinctrl_state *s;
...
};

foo_probe()
{
        /* Allocate a state holder named "foo" etc */
        struct foo_state *foo = ...;

        foo->p = devm_pinctrl_get(&device);
        if (IS_ERR(foo->p)) {
                /* FIXME: clean up "foo" here */
                return PTR_ERR(foo->p);
        }

        foo->s = pinctrl_lookup_state(foo->p, PINCTRL_STATE_DEFAULT);
        if (IS_ERR(foo->s)) {
                /* FIXME: clean up "foo" here */
                return PTR_ERR(s);
        }

        ret = pinctrl_select_state(foo->s);
        if (ret < 0) {
                /* FIXME: clean up "foo" here */
                return ret;
        }
}
```
如果你不想在每个驱动上都执行get/lookup/select/put这一系列的操作的话，并且你知道你总线上的arrangement的话，那么总线驱动(bus drivers)也可以处理这些操作。<br>

这些Pinctrl API的作用：
+ pinctrl_get()：在进程的上下文中被调用，用于获取一个给定设备的所有pinctrl信息的句柄（handle）。它将从内核内存中分配一个结构体，来保存pinmux state。所有的映射表的解析，或是类似的slow operations都在这个API中进行。
+ devm_pinctrl_get()：是pinctrl_get()的一个变种，当相关的设备被移除的时候，对应的pinctrl_put()会在已检索的指针上自动被执行。建议使用这个函数而不是普通的pinctrl_get()
+ pinctrl_lookup_state()：在程序的上下文中被调用，以获取指向特定状态的句柄（handle）。这个操作可能也很慢。
+ pinctrl_select_state()：根据映射表给出的状态（state）定义对pinctrl硬件进行设置。理论上这是一个快速路径操作，因为这仅仅涉及将一些寄存器值设置到硬件中。注意，某些pinctrl的寄存器唯一slow/IRQ-based的总线上，因此设备不该认为它们可以在非阻塞的上下文中调用pinctrl_select_state();
+ pinctrl_put（）：释放所有有关pinctrl handle的信息。
+ devm_pinctrl_put（）：是一个pinctrl_put（）的变种，它可以用来销毁一个由devm_pinctrl_get()返回的pinctrl对象。然而很少使用这个函数，因为即使不调用这个函数，pinctrl对象也会自动销毁。<br>

    pinctrl_get()必须和pinctrl_put()匹配使用。pinctrl_get()和devm_pinctrl_put()和不匹配。devm_pinctrl_get()可以有选择的和devm_pinctrl_put()匹配使用。devm_pinctrl_get()和普通的pinctrl_put()不匹配。<br>

通常pinctrl core会处理get/put之间的匹配，并调用设备驱动bookkeeping (记账)操作，比如检查可用的功能(FUNCTIONS)和相关的引脚。而select_state被传递给pinctrl驱动，他通过配置寄存器来激活/禁用相关的复用设置(mux setting)。<br>

当你执行devm_pinctrl_get()函数的时候，你的设备的引脚就已经分配好了，之后你就可以在所有引脚的debugfs列表中看到它。<br>

注意：如果找不到请求的pinctrl handle，pinctrl系统会返回-EPROBE_DEFER，比如那个pinctrl驱动还被没注册。因此确保你的设备中的错误路径已经清理，并在准备好之后在启动过程( startup process)中，重新尝试probe。<br>

# Drivers needing both pin control and GPIOs
再说一次，不鼓励让驱动自身lookup和select pinctrl state，但是有些情况下不可避免。<br>

假设你的驱动正在这样获取资源：
```
#include <linux/pinctrl/consumer.h>
#include <linux/gpio.h>

struct pinctrl *pinctrl;
int gpio;

pinctrl = devm_pinctrl_get_select_default(&dev);
gpio = devm_gpio_request(&dev, 14, "foo");
```
在这，我们首先请求特定的pin state，之后请求使用GPIO 14。如果你正在像这样分开使用子系统(subsystem)，理论上你应该总是在请求GPIO之前得到你的pinctrl handle并选择所需的pinctrl state。这是一个约定，为了避免电气上不想发生的情况，在GPIO子系统开始处理引脚之前，你肯定想要以某种方式复用和偏置这个引脚。<br>

上面的内容可以忽略：使用device core，pinctrl core在probe之前可以先设置好pins的配置，并且配置好pin的复用，尽管与GPIO子系统正交。<br>

但也有一些情况下，GPIO子系统可以直接和pinctrl子系统通信，使用后者作为back-end。这时GPIO驱动可以调用上面“Pin control interaction with the GPIO subsystem”一节中提到的函数。这仅仅涉及每个引脚的复用，并且完全隐藏在GPIO_*()函数之后。在这种情况下，驱动完全不需要和pinctrl子系统交互。<br>

如果pinctrl驱动和GPIO驱动处理相同的引脚，并且涉及复用，你必须像这样将pinctrl作为GPIO驱动的back-end，除非你的硬件设计使得GPIO控制器可以覆盖pinctrl的复用状态，而不用和pinctrl系统交互。<br>

# Runtime pinmuxing
可以在运行时，改变引脚的复用功能，比如讲SPI口从一组引脚移到另一组引脚。比如将上面例子中的spi0作为例子，我们在同一个功能(FUNCIONS)上暴露了两组不同的引脚，但是在映射中的命名不同。因此对于一个SPI设备，我们有两个state名“pos-A”和“pos-B”。<br>
这段代码首先为两组都初始化了一个state对象(在foo_probe()函数中)，然后在组A的引脚中复用该功能，最后在组B的引脚中复用该功能。<br>
```
struct pinctrl *p;
struct pinctrl_state *s1, *s2;

foo_probe()
{
        /* Setup */
        p = devm_pinctrl_get(&device);
        if (IS_ERR(p))
                ...

        s1 = pinctrl_lookup_state(foo->p, "pos-A");
        if (IS_ERR(s1))
                ...

        s2 = pinctrl_lookup_state(foo->p, "pos-B");
        if (IS_ERR(s2))
                ...
}

foo_switch()
{
        /* Enable on position A */
        ret = pinctrl_select_state(s1);
        if (ret < 0)
        ...

        ...

        /* Enable on position B */
        ret = pinctrl_select_state(s2);
        if (ret < 0)
        ...

        ...
}
```
以上应该在程序的上下文中完成。当state被激活时，引脚的reservation将完成，所以一个特定的引脚可以在运行系统的的不同时间被不同的功能适应。
