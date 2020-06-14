PINCTRL ��ϵͳ<br>
����ĵ�������Linux��Pin control��ϵͳ<br>
�����ϵͳ�����������飺
+ ö�ٺ�����Pins
+ Pins��pads��fingers�ȵĸ��ã������������
+ Pins��pads��finger�ȵĵ������ã���������������©��

# Top-level interface
PIN CONTROLLER�Ķ��壺<br>
+ Pin controller��һ��Ӳ����ͨ����һ����������pins�ļĴ��������ǿ�������Ϊһ��pins��һ��pins���ø��á�ƫ�á��������ȡ�
PIN�Ķ���<br>
+ PINS����pads��fingers��һ������Ҫ���Ƶ�input/output line��PINS��һ����0-maxpins��������ʾ�����number space(0-maxpins)ȡ����PIN CONTROLLER��������һ��ϵͳ������ж��number space�����pin space��һ���������ģ������м������һЩpin�ǲ����ڵġ�

��һ��PIN CONTROLLER��ʵ����֮������ע��һ��descriptor������������pin control ����У����PIN CONTROLLER����������һ��pin����������Щpin�����������pin controller����

������һ��оƬ�����ӣ�
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
��Ҫע��һ��pin controller�����������оƬ�ϵ�����pin�����ǿ���������������д��
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

��Ҫ����pinctrl��ϵͳ��PINMUX��PINCONF��subgroups�Լ�ѡ��������������Ҫ�����оƬ��Kconfig��Ŀ��ѡ�����ǣ���Ϊ����ͨ���������оƬ������ϵ�ġ����Կ���arch/arm/mach-u300/Kconfig��Ϊ���ӡ�<br>

pins nameͨ��������������������ͬ���������оƬ��datasheet���ҵ����ǡ�ע�⣬pinctrl core�е�pinctrl.h�ṩһ������PINCTRL_PIN()�ĺ����ڴ���pinctrl_pin_desc�ṹ�塣����Կ�������0~63ö���˴�A8��H1���������š���ʵ��ʹ���У�����뿼����Щ�����������pin��ʹ���ǿ����������õ�оƬ�ļĴ�����������ϣ����������úܸ��ӡ��㻹��Ҫ����offset�Ƿ��pin controller�����Ƶ�GPIO ranges���ϡ�

# Pin groups
һЩ��������Ҫ����һ��pins������pinctrl�ṩһ������ö�ٺͼ���һ��pin�Ļ��ơ�<br>

��һ������SPI��pins{0,8,16,24},��һ������I2C��pin{24,25}��Ϊ���ӡ�<br>

������pin������pinctrl��ͨ��ʵ����һ��pinctrl_ops����ʾ��

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

Pin controller ��ϵͳ����ͨ������. get_groups_count()��������ȡgroup��������Ȼ��pin controller�Ϳ���ͨ������foo_get_group_name()��foo_get_group_pins()����ȡ��Ӧgroup�����ֺ��������pins��ʵ��group�����ݽṹȡ����������������Ǹ��򵥵����ӡ�

# Pin configuration
Pin����ͨ�����ַ�ʽ�������ã��󲿷�����Ϊinputs��outputsʱ�ĵ������ԡ��ٸ����ӣ�����Ҫ����ĳ��pinΪ���迹����������Ҫ�ý�һ��pin����Ϊ��/������<br>

����ͨ�������Ŀ��ӳ�������������pin configuration�������������Board/machine configuration�Ĳ��֡�<br>

���ò����ĸ�ʽ�ͺ��壬����pin controller��������ġ�����PLATFORM_X_PULL_UP<br>

Pin configuration������pin controller��ʵ���������������õĻص�����,������������
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

��Ϊ��ЩоƬ������һ����pin�������߼��������ǿ���ͨ������Ļص�����������һ����pin�������䲻�봦����飬pin_config_group_set()�������Է��ش���-EAGAIN��������ֻ����Щ�鼶�Ĵ���Ȼ������������е�pin�����������ÿ��pin����ͨ����������pin_config_set()�������á�

# Interaction with the GPIO subsystem
GPIO����������Ҫ��ͬһ��pin��ִ�ж��ֲ������������pin�Ѿ���pin controller��ע�ᡣ<br>

��������Ҫ�ģ���������ϵͳ������ȫ����ʹ�ã�������������pin control requests from drivers��drivers needing both pin control and GPIOs���֡�������һЩ�������Ҫ��pins��GPIO֮ǰ��������ϵͳ��ӳ�䡣<br>

��Ϊpin controller��ϵͳ�����䱾���pinspace�����pinspace��pin controller�ڲ��ģ���������һ��ӳ�䣬����pin control��ϵͳ����֪���ĸ�pin controller�����ĸ�GPIO����Ϊһ��pin controller���ܰ������GPIO range�����͵�SOC��һ��pins���������ڲ��ж��GPIO blocks��ÿ��GPIO blocks�������һ��gpio_chip�����������������GPIO range���Ա���ӵ�pin controllerʵ����������������<br>
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
���ϣ����ϵͳ��һ��pin controller����������ͬ��gpio chip������chip_a����16��pins��chip_b����8��pins��chip_a��chip_b���Ų�ͬ��.pin_base��.pin_base��ʾһ��gpio range��ʼ��pin number��<br>

chip_a��GPIO range��base 32��ʼ������ʵ�ʵ�pin rangeҲ�Ǵ�32��ʼ�ġ���chip_b��GPIO range�Ǵ�48��ʼ�ģ�pin range�Ǵ�64��ʼ�ġ�<br>

���ǿ���ͨ��pin_base��gpio numberת����ʵ�ʵ�pin number��
������ȫ��GPIO pin�ռ��ӳ���ǣ�
```
chip a:
 - GPIO range : [32 .. 47]
 - pin range  : [32 .. 47]
chip b:
 - GPIO range : [48 .. 55]
 - pin range  : [64 .. 71]
 ```
��������ӳ�����gpio��pins�Ĺ�ϵʽ���Եġ����ӳ�䲻�����Եģ�һ�������pins�б�������������������������ӳ��:<br>
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
����������£�pin_base���Իᱻ���ԡ����pin group����������֪�ģ�����ṹ���pin��npinsԪ�أ�����ʹ�ú���pinctrl_get_group_pins()�����ж�pin group��foo�������г�ʼ����<br>
```
pinctrl_get_group_pins(pctl, "foo", &gpio_range.pins,
                       &gpio_range.npins);
```
��pin control��ϵͳ����gpio��صĺ���������ʱ�������Ͻ��������Һ��ʵ�pin controller,ͨ�����pin������������pin controller��pin��Χ��ƥ��ķ�ʽ�����в��ҡ���һ��pin controller�����pin��Χ����������ƥ�䡣GPIO��صĺ���������ָ����pin controller�ϱ����á�<br>

���������漰����ƫ�ú͸��õȵĲ�����pin controller�����ڴ����gpio����в��������pin��ţ���������ϵ��ڲ�����pin��š�����֮��pin controller��ϵͳ���������ݸ�pin control���������������ͻ�õ�pin��š����ң������ᴫ�ݷ�Χ��ID��ţ�����pin controller����֪����Ҫ���������һ��pin��Χ��<br>

��pinctrl �����е��ú�pinctrl_add_gpio_range����̭�ġ���鿴2.1С�� documentation/devicetree/bindings/gpio/gpio.txt�鿴��ΰ�pinctrl��gpio�����󶨡�<br>


# PINMUX interfaces

��Щ����ʹ��pinmux_��Ϊǰ׺�������ĵ��ò���ʹ�����ǰ׺��

## ʲô�����Ÿ��ã�
PinmuxҲ����Ϊpadmux��ballmux,���ù�����оƬ�����������оƬʱ��ĳ��������ʵ�ֶ�����⹦�ܵ�һ�ַ�ʽ����ȡ����Ӧ�á��ڴ��������У�Ӧ��ͨ����ָ�ڵ���ϵͳ�еķ�װ�Ͳ��ߣ�������ʹ��������ʱҲ�ܹ��ı��װ���ܡ�<br>

������һ��GPA��װ��оƬ�����ӣ�<br>
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
�ⲻ�Ƕ���˹���飬�����ǹ������塣�������е�PGA/BGA��װ��������ʽ�ģ������ǽ������Ϊ��򵥵�һ�����ӡ����㿴����Щ����֮�У���һЩ�ᱻ����VCC��GND�ȸ�оƬ����Ķ���ռ�ã����в��ٻᱻ�����ⲿ�洢�������Ĵ�ӿ�ռ�á�ʣ�µ����������ᱻ��Ϊ���Ÿ��á�<br>

����8x8��PGA��װʾ����pin���0~63�ֱ�ָ�����������š�ʹ��pinctrl_register_pins()���ͺ��ʵ����ݼ������������ţ�A1,A2,A3��H6,H7,H8������ǰ������<br
>
�����8x8��BGA��װ�У����ţ�A8,A7,A6,A5�����Ա�����SPI�˿�(CLK,RXD,TXD,FRM)����������У�B5���ſ��Ա�������ͨIO�ڡ�Ȼ������һ�������У�����{A5��B5}������ΪI2C�˿�ʹ�ã�SCL,SDA��������˵�����ǲ���ͬʱʹ��SPI��I2C��Ȼ���ڷ�װ�ڲ���ִ��ִ��SPI�߼��ĵ�·����ѡ���Ե��л���{G4,G3,G2,G1}�����<br>

�����һ��{ A1, B1, C1, D1, E1, F1, G1, H1 } ����һ������Ķ�������һ���ⲿMMC���ߣ������Ա�����Ϊ2/4/8λ���ֱ�ռ��2/4/8�����ţ�����Ҫôʹ�� { A1, B1 ��Ҫôʹ�� { A1, B1, C1, D1 } ��Ҫôʹ�����һ�����е����š��������ʹ��8λģʽ����ô����Ȼ����ʹ�� { G4, G3, G2, G1 } ��<br>

����оƬ�ڲ��ĵ�·�����ڲ�ͬ�����ŷ�Χ(pin range)���á�muxed��.ͨ���ִ���SOC�����������I2C,SPI,SDIO/MMC�ȵȽӿڣ�ͨ�����Ÿ��õ����ÿ��Խ���Щ�ӿ����õ���ͬ�������ϡ�<br>
>
��ΪGPIO�ӿ�ͨ���Ƕ�ȱ�ģ����������ǰû������I/O�˿�ռ��,��ôͨ���κ����Ŷ�������ΪGPIO��<br>


## Pinmux conventions
Pinctrl��ϵͳ��Pinmux���ܵ�Ŀ����Ϊ�˳���machine������ѡ��ʵ�������豸����Ϊ����豸�ṩ���Ÿ������á����ܵ�CLK��GPIO��regulator��ϵͳ�������������豸�����������ǵ����Ÿ����趨����ͬʱ�������󵥸����ţ�������ΪGPIO��<br>

���壺<br>
+ ���ܣ�FUNCTIONS������ͨ��drivers/pinctrl/*Ŀ¼�µ�pinctrl��ϵͳ�е������������л���Pinctrl����֪����ѡ�Ĺ���(FUNCTIONS)��������������У�����Զ����������Ÿ��ù���(Pinmux functions)��SPI��I2C��MMC��
+ ���蹦��(FUNCTIONS)��һ��һά�������д�0��ʼö�١���������£����������Ա���������{ spi0, i2c0, mmc0 }��
+ ����(FUNCTIONS)���ж�����ͨ�ò����ϵ�������(PIN GROUP)������ĳ������(FUNCTIONS)ͨ�����ض���������(PIN GROUP)������������ǵ���������Ҳ�����Ƕ�������顣������������У�I2C���ܺ�����  { A5, B5 }�������{ A5, B5 }��Pin space�ﱻö��Ϊ { 24, 25 } ��<br>
SPI����(FUNCTIONS)��������(PIN GROUP)  { A8, A7, A6, A5 } �� { G4, G3, G2, G1 }����������Ƿֱ�ö��Ϊ��� { 0, 8, 16, 24 } �� { 38, 46, 54, 62 }��<br>
ÿ��Pinctrl�ϵ�������(PIN GROUP)�����ֱ�����Ψһ�ģ�ͬһ��Pinctrl�ϵ��������������ֲ�����ͬ��
+ ����(FUNCTION)������(PIN GROUP)��ͬ������ĳ�����ŵ�ĳ�ֹ��ܡ�����(FUNCTION)������(PIN GROUP)�ͻ���(MACHINE)�ض���ϸ���Ǳ���װ�������еģ�������ֻ֪��ö����������������������
    + �ض����(Selector)(>=0)��Ӧ�Ĺ���(FUNCTIONS)��
    + ĳ������(FUNCTIONS)��Ӧ��PIN GROUP�б�
    + �б���ĳ��GROUPҪ�����ĳ������(FUNCTION)<br>
�����Ѿ����ܹ���������(PIN GROUP)���Լ�����ģ�����pinctrl���Ļ�������в���������(PIN GROUP)ʵ�ʵ����ŷ�Χ(pin range)<br>

+ ĳ��pinctrl�ϵĹ���(FUCNTIONS)��������(PIN GROUP)��ͨ��board file���豸���������������ƵĻ������û���ӳ�䵽ĳ���豸�ϵġ�����ֻ�趨��һ��Pinctrl��functions��group���Ϳ���Ψһȷ��ĳ���豸Ҫʹ�õ����š������ĳ������ֻ��Ӧһ�������飬�������ṩ������core��ѡ��Ψһһ�����õ��飩��<br>
������������У����Ƕ�������ض��Ļ����ڵ�һ��pinctrl(pinctrl0)��,�豸spi0��Ӧ���ù���(FUNCTION) fspi0��������(GROUP)gspi0���豸i2c0��Ӧ����fi2c0 ��������gi2c0�����ǿ�������ӳ�䣺
    ```
    {
            {"map-spi0", spi0, pinctrl0, fspi0, gspi0},
            {"map-i2c0", i2c0, pinctrl0, fi2c0, gi2c0}
    }
    ```
    ÿ��ӳ��������һ��״̬(STATE)��,һ��pinctrl��һ���豸��һ������(FUNCTION)�������鲻�Ǳ���ģ����ʡ���������飬���ѡ�������ṩ�ĵ�һ�������ڸù��ܵ������飬�ڼ򵥵�ʹ������ǳ����á�<br>
    ���Խ�ͬһdevice��pinctrl��function�����ӳ�䵽���group���������ͬһ��pinctrl�ϵ�ͬһ��function�����ڲ�ͬ��������Ҫʹ�ò�ͬ���š�

+ ��ĳ��pinctrl��ʹ��ĳ��������(PIN GROUP)��ĳ������(FUNCTION)�������ȵ��ȵõķ�ʽ�ṩ�ġ�������������豸�ĸ������û���GPIO�����Ѿ�ռ����ĳ���������ţ����㽲�޷�ʹ��������Ҫ����һ���µ��趨���ͱ�����ͣ�þɵ��趨��

��ʱ�ĵ���Ӳ���Ĵ�����ʹ��pad����finger��������pins�����б�д��ֻѡ�������Щ�������õ����Ž���ö�١�<br>

���裺<br>
���Ǽ��蹦��(FUNCTION)ӳ�䵽������(PIN GROUP)����������Ӳ�����Ƶġ�Ҳ����˵���費�����κι��ܶ���ӳ�䵽�κ��������ϵͳ������ĳ������(FUNCTION)���õ�������(PIN GROUP)�Ǳ������ڼ���ѡ��֮�ڵģ��������˸����ң������Ǽ��ٸ����κ�������ѡ�����������ͨ��������е�Ӳ�����ֵģ���Ҳ��һ����Ҫ�ļ��裬��Ϊ����ϣ��pinmux������������ϵͳ�ṩ���п��ܵĹ���(FUCNTIONS)��������ӳ��(PIN GROUP)��

## Pinmux drivers
Pinmux���ĸ����ֹ�����ϵĳ�ͻ�����ҵ���pinctrl������ִ�в�ͬ�����á�<br>
Pinmux����������ʩ�ӽ�һ��������(�����ɸ�����ɵĵ����ϵ�����)���������Ƿ�����������Ĺ���(FUNCTION)�����������ĸ������ñ������Ͷ�Ӳ���������á�<br>
Pinmux������Ҫ�ṩһЩ�ص���������Щ�ص������ǿ�ѡ�ġ�ͨ����Ҫʵ��set_mux()��������ֵд��ĳ���Ĵ������Լ���ĳ�����ŵ����Ը��ù��ܡ�<br>
һ���򵥵�������ʵ����������ӣ�ͨ������bit 0,1,2,3��4��һЩ��Ϊmux�ļĴ�������ʹĳ��������(PIN GROUP)�ϵ��ض�����(FUNCTION)�ܹ���ʹ�ܣ�<br>
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
����������У�����muxing 0��1��ͬʱ����bit 0��1��ʹ����һ����ͬ�����ţ��������ǻᷢ����ͻ��<br>

Pinmux��ϵͳ�ĺô����ڣ���һֱ�������������Լ�˭��ʹ�����ǣ���������ܾ�����һ�������ܵ�����������������Ҫ���ģ������õ�һ����������selector��ʱ��pinmux��ϵͳ��ȷ��û�������豸����GPIO�����Ѿ���ʹ����ѡ�����š�<br>

��˿��ƼĴ�����0��1��Զ����ͬʱ��λ��<br>

���ϵ����й��ܶ���pinmux��������ʵ�ֵġ�<br>

## Pin control interaction with the GPIO subsystem
ע�⣬�����������ζ��ʹ��<linux/gpio.h>�е�API��gpio_request()�����Ƶĺ�����ʹ��Linux�ں˵�ĳ�����š�<br>

��һЩ����£�����ܻ�ʹ��һЩ��datasheet�б���Ϊ��GPIO mode���Ķ�������ʵ������ֻ��ĳ���豸�ĵ������á���ο��µġ�GPIO mode pitfalls��һ�ڡ�<br>

����Pinmux API������������pinctrl_gpio_request()��pinctrl_gpio_free()������������ֻ�ܱ�����gpiolib���������á�pinctrl_gpio_request()��pinctrl_gpio_free()��Ϊ����gpio_request() ��gpio_free()��һ���֡�ͬ����pinctrl_gpio_direction_input()��pinctrl_gpio_direction_output()Ҳֻ����gpiolib�е�gpio_direction_input()��gpio_direction_output()ʵ�ֵ�ʱ�򱻵��á�<br>

ע�⣬ƽ̨�����͵���������Ӧ��ֱ������GPIO���ű����ƣ����縴�����롣�෴�أ�Ӧ��ʵ��һ��gpiolib��������ͨ��gpiolib������Ϊ���������ʵ��ĸ��ù��ܺ������Ŀ��ơ�<br>

�����б���ܻ��úܳ����ر�������㽫ÿ������������ת��Ϊ�������������ŵ�GPIO���ţ����ҳ��Խ�ÿ�����Ŷ���Ϊһ������(FUNCTION)��<br>

����������£�function���齫���64����Ŀ������ÿ��GPIO���ú����豸function��<br>

��������ԭ��pinctrl��������ʵ���������������ڵ���������ʹ��GPIO��.gpio_request_enable() ��gpio_disable_free()��<br>

������������������pinctrl core�����GPIO range���������֪����ЩGPIO pin��request����Ӱ�졣<br>

������������Ҫ�ӿ����ָʾ�ǽ�GPIO pin�������뻹������������ʵ��gpio_set_direction()�����������������ú�������Ҫ��gpiolib�������ã�������Ӱ���GPIO ��Χ(range)������ƫ��(offset)������ķ���(����/���)�������ݸ��ú�����<br>

����ʹ����Щ����ĺ�������ȫ����ʹ�ö�ÿ��GPIO����ʹ�ö���Ĺ���(function)������pinctrl_gpio_request() �����Ի�ȡ���ܡ���gpioN�����С�N����ȫ��GPIO ���źţ����û�������GPIO-handler�Ļ���<br>

# GPIO mode pitfalls
����Ӳ�����̵�����ϰ�ߣ���GPIO���ĺ������ں��еĶ��岻ͬ�������߿��ܻ��datasheet�еĽ�һ���������óɡ�GPIO mode����˵���е����󡣿���Ӳ������ʦ��˵�ġ�GPIO mode���ĺ��岻һ�����ں˽ӿ�<linux/gpio.h>���ἰ�ģ�����ں˴����л�ȡһ��pin��Ȼ��������롢���ǽ�����Ϊ��/�͵�ƽ��<br>

��Ӳ��������Ϊ��GPIO mode����ζ�������ͨ��������Ƶ�һЩ���ŵ������ԡ���������Ŵ�������״̬�����縴�����룬���޷�������Щ�������ԡ�<br>

���ŵ�GPIO���֣�������ص�ĳ��pinctrl���ú͸����߼�֮��Ĺ�ϵ����ͨ�����ַ�ʽ��������������������:
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
��Щ���������������Ƿ�����GPIO������ʹ�õġ�����㽫һ�����Ÿ���ΪGPIO�������ͨ��GPIO�Ĵ�����������Ϊ��/�͵�ƽ�����ߣ����������ĳ�����������ƣ����������Ȼ����Ӧ������������������ԡ�GPIO������ʹ�ø����ŵ��豸֮���ǻ���Ӱ��ġ�<br>

������ģʽ�У�GPIO���ֵļĴ�������GPIOӲ��ģ��ļĴ������ܴ�����һ���������ڴ淶Χ�ڣ��ⲿ���ڴ�ֻ����GPIO���������������ú����Ÿ�����ʹ�õļĴ���������һ���ڴ��ַ�У�������datasheet������Ҳ�Ƕ������ڵġ�<br>

�ṹ��pinmux_ops�е�һ����־��strict�������ڼ��;ܾ�GPIO�����Ÿ����ڴ���Ӳ���϶�ͬһ�����ŵ�ͬʱ���ʡ�Pinctrl����Ӧ����Ӧ���øñ�־��<br>
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
������ģʽ�У�GPIO���ܿ���ʼ�ձ����ã���SPI/I2C/MMC�����ƽ�ź�ʱ��GPIO������������������źš�����ģʽ�£���GPIOģ����ִ�д���Ĳ������ܻ��ƻ������ϵ�ͨ�ţ���Ϊ����δ�����Ͽ���GPIO���������á����Ÿ��üĴ������ܻᱻ�����ڴ��е�ͬһ����Χ����datasheet�п��ܱ�����ͬһ�����С�<br>

��һЩ���ſ������У���Ȼ�������ŵ���ƺ�B��ͬ������GPIO������Ȼ���������蹦��ͬʱ���á���������������¡�strict����־ҲҪ�����ã��Ծܾ�GPIO�����Ÿ���ͬʱ���á�<br>

���ں˵ĽǶ���������Щ��Ӳ���ϵĲ�ͬ���棬Ӧ�ñ��Ž���ͬ����ϵͳ�У�
1.	�������ŵ������ԣ�����ƫ�ú�����ǿ�ȣ��ļĴ�����Ӧ��ͨ��pinctrl��ϵͳ��¶��������Ϊ��pin configuration�����á�
2.	������������Ӳ��ģ��(����I2C��MMC��GPIO)���ź��������ϸ��õļĴ�������Щ�Ĵ���Ӧ��ͨ��pinctrl��ϵͳ��¶��������Ϊ���ù���(mux function)��
3.	��Щ����GPIO���ܵļĴ���������GPIO����Ĵ�����GPIO����Ĵ������Ϳ���GPIO����ļĴ�����Ӧ��ͨ��GPIO��ϵͳ��¶������������ǻ�֧���жϣ���Ӧ��ͨ��irqchip����<br>

���ݾ���Ӳ���Ĵ�����ƣ���Щͨ��GPIO��ϵͳ��¶�����ĺ���������Ҫ����pinctrl��ϵͳ���Ա�Э��Ӳ��ģ��֮��ļĴ������á��ر��Ƕ�����Щ���ж���GPIOģ���pin����ģ���Ӳ����˵��������Ҫ�ģ�����GPIO��������pin�������еļĴ������õĶ�����GPIOģ�顣<br>

����������£����ŵĵ�������(����ƫ�ú�����ǿ��)�������Դ������Щ����ר�õļĴ����ϣ������Ǵ����GPIO�Ĵ�����(�ر���������B��)���Ⲣ����ζ����Щ����һ����Linux�ں���˵�� "GPIO "�йء�<br>

�ٸ����ӣ�һ������ͨ������ΪUART��TX���ŵģ�������ϵͳ���ߵ�ʱ��������Ҫ����������ΪGPIOģʽ�����������͡�<br>

���Ϊ���������GPIO��ϵͳ������һ��һ��ӳ�䣬��ʱ����ܾ�����Ҫ����·ǳ����ӣ���Ϊ���������Ҫ��Ϊͬʱ��ΪUART TX��GPIO������ȡһ��pinctrl handle����������Ϊĳ��״̬������UART TX��Ϊ�������룬֮����˯���ڼ佫��ת��Ϊgpioģʽ����ʹ��gpio_direction_output()����������������ţ�Ȼ���ڻ��ѵ�ʱ�������¸��ó�UART TX���������������ڼ�ʹ��gpio_request/gpio_free����һ�ж����úܸ��ӡ�<br>

����ķ����ǲ�Ҫ��Ϊ�����ֲ�����˵�� "GPIOģʽ "������<linux/gpio.h>�ӿ����������ǰ���������һ���ض��������������á�������<linux/pinctrl/pinconf-generic.h>�в��ң�������ĵ����ҵ������<br>
```
PIN_CONFIG_OUTPUT
�⽫���������ó������ʹ�ò���1��ָ������ߵ�ƽ��ʹ�ò���0��ָ������͵�ƽ
```

�����ȫ������ƽʱʹ��pinctrl mapһ����������������ΪGPIOģʽ������͵�ƽ�������������Ϊpinctrl map�е�һ���Ϊ���ӣ�UART��������������д��<br>
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

### ���һ��������ÿ���������������ӣ�
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
������Ҫ���Ƶ������ڡ�u0_group���У�������һ��������u0���Ĺ���(FUNCTION)�������������������ã�Ȼ��һ�о�������ʹ��UARTһ�������⻹��һ����Ϊ��gpio-mode���Ĺ��ܣ�FUNCTION��������Ӧ������ͬ�������ϣ�ʹ������ű�ΪGPIOģʽ��<br>

�����Ϳ����ڲ���GPIO��ϵͳ������������õ�Ԥ�ڵ�Ч������ֻ���豸�ڽ�������״̬ʱʹ�õĵ������ԣ��������������datasheet�п��ܱ���ΪGPIO mode���ص��ǣ����Կ��Ա�UART�豸��������UART������ʹ�õ����ţ�ʹ��Щ���Ž���UART��Ҫ��ģʽ��Linux�ں��е�GPIOָ����1-bit��line��������˵��GPIO��һ����<br>

��ν����üĴ�����ʵ��push����pull,����͵�ƽ,�Լ��������Щ�����ϸ��ù��ܡ�u0�����ǡ�gpio-mode�������������顣<br>

��һЩdatasheet�гơ�GPIO mode��Ϊ�͹���ģʽ�������Ǻ�GPIO�йص��κζ������ӵ�������������ָ����ͬһ�����飬���������Ϊ�͹���ģʽ���������ʦ�ܿ�ͻᷢ������һ���ض��ĸ��ã�������һ�����ã������Ǻ�GPIO API��صĶ�����<br>

# Board/machine configuration
Board��Machine configuration������һ���������е�ϵͳ����������һ��ģ�����GPIOs���豸�������ϵģ����Ÿ�����������õġ�<br>

һ��machine��pinctrl configuration�����������Ǽ򵥵�regulator configuration��������ǿ������������������ӵĵڶ���fucntionӳ��������I2C��SPI��
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
�����dev_name��Ψһ���豸������ƥ�䣬������������device�ṹ�壨����clockdev��regulatorsһ����������(FUCNTION)�����ֱ���ʹ�pin range��pinmux�������ṩ�Ĺ���(FUNCTION)��ƥ�䡣<br>

��������������ϵͳ�Ͽ����ж��pinctrl�����������Ҫָ����һ��pinctrl������������Ҫӳ��Ĺ���(FUNCTION)��<br>

�����ͨ�����·�ʽ���� pinmux mappingע�ᵽpinmux��ϵͳ��
```
ret = pinctrl_register_mappings(mapping, ARRAY_SIZE(mapping));
```
��Ϊ����Ĺ���ǳ������������һ���궨�������򻯴��롣�ٸ����ӣ��������ʹ����Ϊpinctrl-foo��pinctrl��λ��0��ӳ�䣬��������д��
```
static struct pinctrl_map mapping[] __initdata = {
        PIN_MAP_MUX_GROUP("foo-i2c.o", PINCTRL_STATE_DEFAULT,
                          "pinctrl-foo", NULL, "i2c0"),
};
```
ӳ���ͬ��������pin configuration��Ŀ�� ͨ��ÿ��pin/groupͨ�������˶��configuration��Ŀ���������ñ���������һ���������ò�����ֵ�����顣������ʹ�ú��������ӳ�������ӣ�
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
���һЩ�豸����ӳ������ĳ���ض���������״̬�����ڲ���Ҫ�κ�pinctrl���õ�Ӳ��������ʱ��ӳ�����Ȼ���������Щ״̬������ȷ��ʾ�ṩ����Щ״̬�ˣ�����Ҫ�����ÿա�ӳ�����ʹ�õĺ궨�� PIN_MAP_DUMMY_STATE���ڶ���һ��״̬�����ǲ��ᵼ��pinctrl��program��
```
static struct pinctrl_map mapping[] __initdata = {
        PIN_MAP_DUMMY_STATE("foo-i2c.0", PINCTRL_STATE_DEFAULT),
};
```

# Pin control requests from drivers
��һ���豸������Ҫprobe��ʱ��device core���Զ���������Щ�豸��ִ��pinctrl_get_select_default()��ͨ�����ַ�ʽ��д�������˲���Ҫ��������ᵽ��������롣Ȼ������Ҫ���豸����״̬(state)ѡ����ǲ�ʹ�á�default��״̬ʱ����������豸�����϶�pinctrl handle��state��Щ����<br>

��ˣ������ֻ�뽫ĳ���豸����������default state���������һЩ������ô�����ṩ�ʺϵ�ӳ����⣬��ʲôҲ����Ҫ����device core���Ḻ��ʣ��Ĳ��֡�<br>

ͨ���������õ������������ò�����pinctrl����ˣ����ܵĻ���platform code�д���pinctrl���������������Է��ʵ�������Ӱ���struct device *ָ��ĵط�����pinctrl����ĳЩ����£�������Ҫ������ʱ�л���ͬ��Muxӳ�䣬���ǲ����ܵġ�<br>

һ�����͵������ǣ�������ʱ��PINCTRL_STATE_DEFAULT�л���PINCTRL_STATE_SLEEP �� һ��������Ҫ�����ŵ�ƫ�ô�����ģʽ�л���˯��״̬��ͨ�����µ���ƫ���������¸�����������ʡ������<br>

һ���������ܻ����󼤻�ĳ��state��ͨ������������default state��
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
����㲻����ÿ�������϶�ִ��get/lookup/select/put��һϵ�еĲ����Ļ���������֪���������ϵ�arrangement�Ļ�����ô��������(bus drivers)Ҳ���Դ�����Щ������<br>

��ЩPinctrl API�����ã�
+ pinctrl_get()���ڽ��̵��������б����ã����ڻ�ȡһ�������豸������pinctrl��Ϣ�ľ����handle�����������ں��ڴ��з���һ���ṹ�壬������pinmux state�����е�ӳ���Ľ������������Ƶ�slow operations�������API�н��С�
+ devm_pinctrl_get()����pinctrl_get()��һ�����֣�����ص��豸���Ƴ���ʱ�򣬶�Ӧ��pinctrl_put()�����Ѽ�����ָ�����Զ���ִ�С�����ʹ�����������������ͨ��pinctrl_get()
+ pinctrl_lookup_state()���ڳ�����������б����ã��Ի�ȡָ���ض�״̬�ľ����handle���������������Ҳ������
+ pinctrl_select_state()������ӳ��������״̬��state�������pinctrlӲ���������á�����������һ������·����������Ϊ������漰��һЩ�Ĵ���ֵ���õ�Ӳ���С�ע�⣬ĳЩpinctrl�ļĴ���Ψһslow/IRQ-based�������ϣ�����豸������Ϊ���ǿ����ڷ��������������е���pinctrl_select_state();
+ pinctrl_put�������ͷ������й�pinctrl handle����Ϣ��
+ devm_pinctrl_put��������һ��pinctrl_put�����ı��֣���������������һ����devm_pinctrl_get()���ص�pinctrl����Ȼ������ʹ�������������Ϊ��ʹ���������������pinctrl����Ҳ���Զ����١�<br>

    pinctrl_get()�����pinctrl_put()ƥ��ʹ�á�pinctrl_get()��devm_pinctrl_put()�Ͳ�ƥ�䡣devm_pinctrl_get()������ѡ��ĺ�devm_pinctrl_put()ƥ��ʹ�á�devm_pinctrl_get()����ͨ��pinctrl_put()��ƥ�䡣<br>

ͨ��pinctrl core�ᴦ��get/put֮���ƥ�䣬�������豸����bookkeeping (����)��������������õĹ���(FUNCTIONS)����ص����š���select_state�����ݸ�pinctrl��������ͨ�����üĴ���������/������صĸ�������(mux setting)��<br>

����ִ��devm_pinctrl_get()������ʱ������豸�����ž��Ѿ�������ˣ�֮����Ϳ������������ŵ�debugfs�б��п�������<br>

ע�⣺����Ҳ��������pinctrl handle��pinctrlϵͳ�᷵��-EPROBE_DEFER�������Ǹ�pinctrl��������ûע�ᡣ���ȷ������豸�еĴ���·���Ѿ���������׼����֮������������( startup process)�У����³���probe��<br>

# Drivers needing both pin control and GPIOs
��˵һ�Σ�����������������lookup��select pinctrl state��������Щ����²��ɱ��⡣<br>

���������������������ȡ��Դ��
```
#include <linux/pinctrl/consumer.h>
#include <linux/gpio.h>

struct pinctrl *pinctrl;
int gpio;

pinctrl = devm_pinctrl_get_select_default(&dev);
gpio = devm_gpio_request(&dev, 14, "foo");
```
���⣬�������������ض���pin state��֮������ʹ��GPIO 14������������������ֿ�ʹ����ϵͳ(subsystem)����������Ӧ������������GPIO֮ǰ�õ����pinctrl handle��ѡ�������pinctrl state������һ��Լ����Ϊ�˱�������ϲ��뷢�����������GPIO��ϵͳ��ʼ��������֮ǰ����϶���Ҫ��ĳ�ַ�ʽ���ú�ƫ��������š�<br>

��������ݿ��Ժ��ԣ�ʹ��device core��pinctrl core��probe֮ǰ���������ú�pins�����ã��������ú�pin�ĸ��ã�������GPIO��ϵͳ������<br>

��Ҳ��һЩ����£�GPIO��ϵͳ����ֱ�Ӻ�pinctrl��ϵͳͨ�ţ�ʹ�ú�����Ϊback-end����ʱGPIO�������Ե������桰Pin control interaction with the GPIO subsystem��һ�����ᵽ�ĺ�����������漰ÿ�����ŵĸ��ã�������ȫ������GPIO_*()����֮������������£�������ȫ����Ҫ��pinctrl��ϵͳ������<br>

���pinctrl������GPIO����������ͬ�����ţ������漰���ã��������������pinctrl��ΪGPIO������back-end���������Ӳ�����ʹ��GPIO���������Ը���pinctrl�ĸ���״̬�������ú�pinctrlϵͳ������<br>

# Runtime pinmuxing
����������ʱ���ı����ŵĸ��ù��ܣ����署SPI�ڴ�һ�������Ƶ���һ�����š����罫���������е�spi0��Ϊ���ӣ�������ͬһ������(FUNCIONS)�ϱ�¶�����鲻ͬ�����ţ�������ӳ���е�������ͬ����˶���һ��SPI�豸������������state����pos-A���͡�pos-B����<br>
��δ�������Ϊ���鶼��ʼ����һ��state����(��foo_probe()������)��Ȼ������A�������и��øù��ܣ��������B�������и��øù��ܡ�<br>
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
����Ӧ���ڳ��������������ɡ���state������ʱ�����ŵ�reservation����ɣ�����һ���ض������ſ���������ϵͳ�ĵĲ�ͬʱ�䱻��ͬ�Ĺ�����Ӧ��
