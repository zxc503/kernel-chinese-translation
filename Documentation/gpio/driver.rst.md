# GPIO Descriptor Driver Interface
����ĵ�����ΪGPIOоƬ������д�ߵ�ָ�ϡ���ע�⣬����ĵ����������µĻ����������Ľӿ�(descriptor-based interface)�������Ѿ��������õĻ��������Ľӿ�(integer-based interface)��ο��ĵ�gpio-legacy.txt��

ÿ��GPIO��������������Ҫinclude�������ͷ�ļ����������˶���GPIO������Ҫ�õ��Ľṹ�塣
```
    #include <linux/gpio/driver.h>
```
# Internal Representation of GPIOs
һ��GPIO chip����һ�������GPIO lines����Ϊһ��GPIO chip��line������϶��壺ͨ������/��������һ��line����ͨ�õģ����Ͳ���GPIO���Ͳ�����GPIO chip����ĳЩline��ϵͳ�п��ܱ���ΪGPIO������ȴ��������Ŀ�ģ���˲�����ͨ������/����ı�׼����һ���棬һЩLED�����߿�������GPIO�������Ӧ��GPIO chip��������

��GPIO���������У�ÿ��GPIO line�����ǵ�Ӳ���������ʶ����ʱҲ����Ϊƫ��(offset)��ƫ����0��n-1��Ψһ��������n��оƬ�����GPIO������

Ӳ��GPIO��Ŵ�Ӳ����������Ӧ����ֱ�۵ģ��������һ��ϵͳʹ��һ���ڴ�ӳ���I/O�Ĵ���������32��GPIO line�����32λ�ļĴ�����ÿһ��bit������һ��GPIO line,��ôʹ��Ӳ��ƫ����0~31����Ӧ�Ĵ�����0~31 bit���Ǻܺ���ġ�

�������ֻ���ڲ��ɼ����ض�GPIO line��Ӳ�����������֮���ǿ������ġ�

������ڲ���ŵĻ����ϣ�ÿ��GPIO����Ҫ������GPIO�����ռ���ӵ��һ��ȫ�ֱ�ţ��Ա���ݴ�ͳ��GPIO�ӿڡ�ÿ��chip������һ����base����(���Ա��Զ�����)����ôÿ��GPIO line��ȫ�ֱ�ž���base+Ӳ����š����ܻ���������GPIO�ӿ��Ѿ������ã��������д����û��������Ҫά����

�ٸ����ӣ�һ��ƽ̨��GPIOȫ�ֱ�ŷ�Χ��32~159����ʹ��һ����������32��Ϊbase��������128��GPIO line������һ��ƽ̨ʹ��ȫ�ֱ��0~63��Ϊһ��GPIO��������64-79��Ϊ��һ�����͵�GPIO����������������һ���ض��İ����ϣ�ʹ��80~95��ʾһ��FPGA��ʣ������ֲ����������ģ�������ƽ̨�е��κ�һ��Ҳ����ʹ������2000~2063����ʾI2C GPIO��չ�����е�GPIO line��

# Controller Drivers: gpio_chip
��gpiolib����У�ÿ��GPIO ����������װ��һ����gpio chip�����鿴<linux/gpio/driver.h>��ȡ��ϸ����)�����ԱΪ�����͵�ÿ���ط��������������У���Щ��ԱӦ��������������䣺

+ һ��������������GPIO line�ķ���
+ һ��������������GPIO line��ֵ
+ һ����������Ϊ������GPIO line���õ�������
+ һ�������������������GPIO line�������IRQ���
+ һ��flag˵�������䷽���Ƿ������
+ ��ѡ��line����������������line
+ ��ѡ��debugfs dump��������ʾ�����״̬��Ϣ��
+ ��ѡ��base��ţ�������ԵĻ������Զ����䣩
+ optional label for diagnostics and GPIO chip mapping using platform data

ʵ��gpio_chip�Ĵ�����Ҫ֧�ֶ��������ʵ�������ʹ������ģ�͡��ô��뽫����ÿ��gpio_chip���һ�ִ��gpiochip_add(),gpiochip_add_data()��devm_gpiochip_add_data()���Ƴ�һ��GPIO�������Ǻ��ټ��ģ�����Ҫ�Ƴ���ʱ��ʹ��gpiochip_remove()��

ͨ������£�gpio_chip��һ���ض�ʵ���ṹ��һ���֣���״̬����GPIO�ӿ�����¶����Ѱַ����Դ����ȡ� ��Ƶ���������оƬ���и��ӵ�non-GPIO״̬��

�κ�debugfs dump����ͨ��Ӧ�ú���û�б������line��debugfs dump��������ʹ�ú���gpiochip_is_requested()���жϣ���������᷵��NULL������GPIO line������ʱ�������GPIO line������label��

ʵʱע������:���Ҫ��ʵʱ�ں��ϴ�ԭ���������е���GPIO API����ӲIRQ�����������Ƶ��������ڣ���GPIO�����Ͳ�����gpio_chipʵ����(��.get/.set�ͷ�����ƻص�����)ʹ��spinlock_t���������κο�˯�ߵ�API(��PM runtime ����ʱ��Դ����)��ͨ�������û��������ơ�

# GPIO electrical configuration
GPIO line���Ա����óɶ����������ģʽ��ͨ��ʹ��.set_config()�ص���������ǰ���API֧�����ã�

+ ��������
+ ����ģʽ����©/��Դ)
+ ��������������ʹ��

��Щ���û���������ϸ������

.set_config()�������ʹ����pinctrl������ͬ��ö�ٺ����ö��塣�ⲻ���ɺϣ�gpio_chip���п��ܻ��set_config()���������gpiochip_generic_config()����gpiochip_generic_config()������pinctrl_gpio_set_config()�����ã�����������GPIO�����������pinctrl�н�����

## GPIO lines with debounce support

������һ�����õ������ϵ����ã����Ա����������ӵ�һ����е���ػ�ť���������ƵĻ���������Ķ�����������ζ��line��Ϊ��еԭ���ڷǳ��̵ļ���ڱ���������/�͡���ᵼ��ֵ���ȶ�����IRQ�ᱻ�ظ����ã�����line�Ƿ����ġ�

��ʵ����������ζ�ŵ���line�Ϸ����¼���ʱ�򣬻�����һ����ʱ�����ȵ�һ���֮����������line�����²������Ա�鿴line���Ƿ������ͬ��ֵ����Ҳ����ͨ��һ��clever state machine���ظ����ȴ�һ��line���ȶ���������������£�������Ϊ��������һ���ĺ��������������ʱ�䲻�����ã����������on/off.

## GPIO lines with open drain/source support
��©(CMOS)�Ϳ���(TTL)��ζ��lineû����������ߣ��෴�أ����ṩ©��/���缫��Ϊ��������Ե������û�б���ʱ��������ⲿrail���ָ��裨��̬����
```
CMOS CONFIGURATION      TTL CONFIGURATION

         ||--- out              +--- out
  in ----||                   |/
         ||--+         in ----|
             |                |\
            GND                 GND
```
�������ͨ������Ϊʵ���������������е�һ���ķ�����

## GPIO lines with pull up/down resistor support

GPIO line����ͨ��ʹ��.set_config()�ص�������֧����/����������ζ�����GPIO line��һ�����õ���/�������裬����������ͨ��������ơ�

��һ�������У�һ�����������������Ǽ򵥵ĺ��ڵ�·���ϵġ��������ǾͲ���Ҫ������д���ʲô���������ȵ�������Щline�ᱻ����Ϊ��©��Դ(�鿴����һ��)��

�ص�����.set_config()����ֻ�ܿ�����/���������ܿ���ʹ�õĵ���ֵ����˵�����������ؼĴ����е�һ��bit��������/������������

���GPIO line֧�ֵ�����/����ʹ�õĲ�ͬ����ֵ����ôGPIO chip�ص�����.set_config()�Ͳ������ˡ�Ϊ��ʵ�����ָ��ӵ��÷���GPIO chip��pinctrl�������һ��ʹ�ã���Ϊpinctrl��pin config�ӿ�֧�ֿ��ƶ��ֵ������ԣ����ҿ��Դ����������������ֵ��

## GPIO drivers providing IRQs

GPIO����(GPIO chips)ͬ���ṩ�ж�֧�֣�ͨ���Ǽ�������һ���жϿ�������������һЩ���������

GPIO block��IRQ������ʹ��irq_chipʵ�ֵģ�ʹ��ͷ�ļ�<linux/irq.h>�����������������ͬʱ����������ϵͳ��gpio��irq��

�����κ�IRQ��������˵�����Դ�irq_chip����һ��IRQ����ʹ����һ��GPIO��IRQ������������������ǰ����gpio_chip��irq_chip�Ƕ����ģ������໥�������ṩ����

gpiod_to_irq()���Ը������ĺ�������ָ��ĳ��GPIO line��IRQ�����в���Ҫ��IRQʹ��֮ǰ�����ã�Ҳ�ܹ�����ʹ�á�

��׼����Ӳ������ʹ����GPIO��irq_chip API�еĻص��������ܹ�������������׼������Ҫ���������ȵ��õ�gpiod_to_irq()��

���ǿ��԰�GPIO irqchips����������Ŀ¼�У�

+ �����ж�оƬ������ζ��GPIO chip��һ��ͨ�õ��ж�����ߣ���ͨ��chip���κα����õ�GPIO line������������ж�����߽�����������һ�����жϿ������У���򵥵��������ϵͳ���жϿ����������ģ����irqchip�����ģ�irqchip����GPIO�������е�bits����ȷ������һ��line�������жϡ�������irqchip������Ҫ���Ĵ�����ȷ�����Ǹ�line�����жϣ������ƺ�����Ҫͨ�����ĳЩbit��ȷ��������������ж�(��ʱ����ʽ�ģ������״̬�Ĵ���)������ͨ������Ҫ����һЩ���ã������Ե���ж�(������/�½��أ����Ǹ�/�͵�ƽ�ж�)

+ �ֲ��ж�оƬ����ζ��ÿһ��GPIO line��һ��ר�õ�irq line����һ�����жϿ�����������Ҫ��ѯGPIOӲ����ȷ���ĸ�line�������������б�Ҫ��������жϣ�ȷ���жϣ������������Ǳ�Ե���жȵ����á�

ʵʱ(Realtime)ע�����һ��ʵʱ���ݵ�GPIO������irqchip��ʵ���в�Ӧ��ʹ��spinlock_t��������˯�ߵ�API������PM runtime����

+ ʹ��raw_spinlock_t����spinlock_t
+ �������ʹ�ÿ�˯��API������ͨ��ʹ��.irq_bus_lock()��.irq_bus_unlock()�ص���������ɣ���Ϊ����irqchip��Ψһ����·���ص������������Ҫ�ʹ����ص�������

# Cascaded GPIO irqchips
����GPIO irqchipͨ����Ϊ���ࣺ
+ ��ʽ����GPIO irqchip(CHAINED CASCADED GPIO IRQCHIPS)����������ͨ����Ƕ����SOC�еġ�����ζ��GPIOs��һ������IRQ��������򣬴Ӹ�IRQ�����������͵���ϵͳ�жϿ�����������ʽ��ʽ�����á�����ζ��GPIO irqchip������������Ӹ�irqchip���ã�ͬʱ����IRQs���á�Ȼ��GPIO irqchip�����������жϴ�����������������˳����ú�����

    ```
    static irqreturn_t foo_gpio_irq(int irq, void *data);
    chained_irq_enter(...); 
    generic_handle_irq(...); 
    chained_irq_exit(...);
    ```

    ��ʽGPIO irqchipһ�㲻�����ýṹ���е�.can_sleep��־����Ϊ���е��¶�ֱ���ڷ����ڻص��У���������I2C������������ͨ�š�

    ʵʱע�������ע�⣬��ʽIRQ������򲻻���-RT��ǿ���̻߳�����ˣ�spinlock_t���κο�˯�ߵ�API����PM runtime����������ʽIRQ���������ʹ�á�

    �����Ҫ�Ļ�����ʽIRQ���������Ա�ת��Ϊͨ��IRQ�������(�������ת���Ļ�����ο�����)��ͨ�����ַ�ʽ������-RT�ϱ�Ϊ�̻߳���IRQ��������ڷ�-RT�ϱ�ΪӲIRQ�������(�������[3])��

    ����generic_handle_irq()Ӧ����IRQ���õ�ʱ����ã�����������ڱ�ǿ�Ƶ��̵߳�IRQ��������б�����ʱ��IRQ core�ᷢ����Թ��fake��raw lock���Ա��������������⣺
    ```
    raw_spinlock_t wa_lock;
    static irqreturn_t omap_gpio_irq_handler(int irq, void *gpiobank)
            unsigned long wa_lock_flags;
            raw_spin_lock_irqsave(&bank->wa_lock, wa_lock_flags);
            generic_handle_irq(irq_find_mapping(bank->chip.irq.domain, bit));
            raw_spin_unlock_irqrestore(&bank->wa_lock, wa_lock_flags);
    ```
+ ͨ����ʽGPIO IRQCHIP(GENERIC CHAINED GPIO IRQCHIPS)�����롰��ʽGPIO irqchips����ͬ�����ǲ�ʹ����ʽIRQ��������෴��GPIO IRQ����ͨ��IRQ�������ִ�еģ�ͨ��IRQ�����������request_irq()���õġ�Ȼ��GPIO irqchip�����������жϴ�����������������˳����ú�����
    ```
    static irqreturn_t gpio_rcar_irq_handler(int irq, void *dev_id)
        for each detected GPIO IRQ
            generic_handle_irq(...);
    ```
    ʵʱ(Realtime)ע������������͵��жϴ�����򽫻���-RT�ϱ�ǿ���̻߳�����˵���IRQ����ʱ����generic_handle_irq()��IRQ core�ᷢ����Թ�����Բ��ú͡���ʽGPIO irqchips����ͬ�ı�ͨ������

+ Ƕ�׵��̻߳�GPIO irqchips(NESTED THREADED GPIO IRQCHIPS)������Ƭ��GPIO��չ������������λ������������һ�Ե�GPIO irqchip������I2C��SPI��

    ��Ȼ��Щ������Ҫ��������ͨ��������IRQ״̬���������ƵĶ�������Щͨ�ſ��ܻᵼ��������IRQ��������˵�IRQ����ʱ�������ڿ���IRQ�жϴ�������н��д����෴�أ�������Ҫ����һ���̣߳�Ȼ�����θ�IRQ line��ֱ�������������ж�Ϊֹ���������������ص����������жϴ�������е������ƵĶ�����
    ```
    static irqreturn_t foo_gpio_irq(int irq, void *data)
    ...
    handle_nested_irq(irq);
    ```
    �̻߳�GPIO irqchips���ص������ǽ�struct GPIOоƬ�ϵ�.can_sleep��־����Ϊtrue���������оƬ�ڷ���GPIOʱ���ܻ����ߡ�

    ����irqchips�����;���ʵʱ�ԣ���Ϊ�����Ѿ����Ǳ�������������˯�������ĵġ�

# Infrastructure helpers for GPIO irqchips
���ڰ������ú͹���GPIO irqchips�����irqdomain����Դ����ص�������ͨ��ѡ��Kconfig symbol GPIOLIB_IRQCHIP������IRQCHIP�����ͬʱѡ����IRQ_DOMAIN_HIERARCHY ������ṩ��νṹ�������򡣴󲿷ֵĴ��뿪����gpiolib�ṩ��ǰ�����жϺ�GPIO line֮���ӳ����1��1�ģ�
```
GPIO line offset	Hardware IRQ
0	0
1	1
2	2
��	��
ngpio-1	ngpio-1
```
�����ЩGPIO lineû����Ӧ��IRQ����ôgpio_irq_chip�е�λ����valid_mask�ͱ�־need_valid_mask������ĳЩline��ʹ����irq������Ч��

���ð�����������ȷ�ʽ�ǣ������gpio_chip֮ǰ����дgpio_chip�еĽṹ��gpio_irq_chip���������������gpiolib��������ʣ��GPIO���ܵ�ͬʱ���ö����irq_chip��������һ��ʹ��gpio_irq_chip�ļ����жϴ������ĵ������ӣ�
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
��������Ҳ֧��ʹ�÷ֲ��жϿ�����������������£������������£�
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
�������������ַ�ʽ�ǳ������ƣ������㲻��ҪΪIRQ�ṩһ��parent handler��ȡ����֮����һ��parent irqdomian��һ��Ӳ����fwnode��һ��������child_to_parent_hwirq()���ڴ�һ���ӣ�Ҳ�������gpio_chip��Ӳ��irq���Ҹ�Ӳ��irq��ͨ��ͨ���鿴�ں����ֵ�ʵ���Ի���й�����ҵ�����Ƭ�εĽ��顣

�ɵ���gpio_chipע��֮�����irqchip�ķ�ʽ��Ȼ�ǿ��õģ�����������ͼ������һ�㣺
+ ��̭�ģ�gpiochip_irqchip_add()����ʽ����irqchip��ӵ�һ��gpiochip�С������оƬ��struct gpio_chip*���ݸ�����IRQ�ص����������Իص�������Ҫ��gpio_chipǶ�뵽�������state container�У�����ʹ��container_of()���һ��ָ��container��ָ�롣
+ gpiochip_irqchip_add_nested():���һ��Ƕ�׵ļ���irqchip��һ��gpiochip�У���������ڲ�ͬ���͵ļ���irqchip�����۵ġ�����irq�������̻߳����жϴ�������������֮�⣬���Ĺ���ԭ�����ʽirqchip��ȫһ����
+ gpiochip_set_nested_irqchip()��Ϊһ�����Ը���IRQ��gpio_chip����һ��Ƕ�׼�����irq����������Ϊ����IRQͨ��������������ʽ��������˽�������IRQ���Ϊ����һ����IRQ֮�⣬û�����������á�

�����Ҫ����Щ�����������IRQ�����ų�ĳЩGPIO line�����ǿ�����devm_gpiochip_add_data()��gpiochip_add_data() ������֮ǰ����gpiochip��.irq.need_valid_mask����������һ��.irq.valid_mask�������õ�λ����оƬ��GPIO line��λ����ͬ��ÿ��λ����line 0...n-1�������������ͨ�������������е�λ���ų�GPIO line��������������gpiochip_irqchip_add() ��gpiochip_irqchip_add_nested()����֮ǰ���á�

ʹ�ð��������ʱ����ע�����¼��㣺
+ ȷ��ָ������struct gpio_chip��صĳ�Ա���Ա�irq_chip���Գ�ʼ�������磺.dev��.can_sleepӦ�ñ���ȷ�����á�
+  Nominally set all handlers to handle_bad_irq() in the setup call and pass handle_bad_irq() as flow handler parameter in gpiochip_irqchip_add() if it is expected for GPIO driver that irqchip .set_type() callback will be called before using/enabling each GPIO IRQ. Then set the handler to handle_level_irq() and/or handle_edge_irq() in the irqchip .set_type() callback depending on what your controller supports and what is requested by the consumer.

# Locking IRQ usage
��ΪGPIO��irq_chip�Ƕ����ģ��ҿ��Է��ַ��ֲ�ͬ����֮��ĳ�ͻ������һ������IRQ��GPIO line����Ҫ������ģ���Ϊ��һ����� line�ϴ����ж�ʱû����ġ�

�����ϵͳ�ڲ����ھ�������һ������ʹ����Դ������ĳ��GPIO line�ͼĴ�����������Ҫ��gpiolib��ϵͳ�ڲ��ܾ�ĳЩ����������ʹ�������

����GPIO������ΪIRQ�źš������������ʱ���������������GPIOΪ����IRQ:
```
int gpiochip_lock_as_irq(struct gpio_chip *chip, unsigned int offset)
```
����ֹʹ��non-irq��ص�GPIO API��ֱ��GPIO IRQ�����ͷţ�
```
void gpiochip_unlock_as_irq(struct gpio_chip *chip, unsigned int offset)
```
��������GPIO����������һ��irqchipʱ������������һ����Ҫ��irqchip�е�.startup��shutdown�ص������ڱ����á�

��ʹ��gpiolib irqchip ���������ʱ�򣬻ص����Զ���ָ�ɡ�

# Real-Time compliance for GPIO IRQ chips
�κ� irqchips ���ṩ�߶���Ҫ���Ķ�����֧��ʵʱ��ռ��GPIO��ϵͳ�е�����irqchips��Ӧ���μ���һ�㣬�������ʵ��Ĳ��ԣ���ȷ��������ʵʱ�ġ�

������ע�������ĵ��е�ʵʱע�����

��������׼��һ��ʵʱ��������ʱ��Ҫ��ѭ�ļ���
+ ȷ��spinlock_tû��irq_chip������ʹ��
+ ȷ��û����irq_chip��ʹ�ÿ�˯�ߵ�API���������ʹ�ÿ�˯�ߵ�API������ͨ��ʹ��.irq_bus_lock()��.irq_bus_unlock()�ص�������
+ ��ʽGPIO irqchips��ȷ��spinlock_t�Ϳ�˯�ߵ�APIû������ʽIRQ���������ʹ�á�
+ ͨ����ʽGPIO irqchips��ע�� generic_handle_irq()�ĵ��ã���������Ӧ�Ľ��������
+ ��ʽGPIO irqchips��������ܾ���ʹ��ͨ��IRQ������򣬶�������ʽIRQ�������
+ regmap_mmio������ͨ������.disable_locking���Ҵ���GPIO�����е����Ͻ���regmap���ڲ�����
+ ʹ���ʵ����ں�ʵʱ�����������������������򣬰�����ƽ�ͱ�Եirq

# Requesting self-owned GPIO pins
��ʱ������GPIOоƬ����ͨ��gpiolib API�������Լ���GPIO�������Ǻ����õġ�GPIO��������ʹ�����º�����������ͷ���������
```
struct gpio_desc *gpiochip_request_own_desc(struct gpio_desc *desc,
                                            u16 hwnum,
                                            const char *label,
                                            enum gpiod_flags flags)

void gpiochip_free_own_desc(struct gpio_desc *desc)
```
��gpiochip_request_own_desc()���������������ͨ��gpiochip_free_own_desc()���ͷš�

��Щ��������С��ʹ�ã���Ϊ���ǲ���Ӱ��ģ��ʹ�ü�������Ҫʹ�ú����������ڵ������������gpio��������

+ [1] http://www.spinics.net/lists/linux-omap/msg120425.html
+ [2] https://lkml.org/lkml/2015/9/25/494
+ [3] https://lkml.org/lkml/2015/9/25/495