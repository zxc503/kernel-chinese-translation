# ʲô��SPI��
��

# ˭ʹ��SPI����������ϵͳ��ʹ�ã�
��

# SPI������ʱ��ģʽ�������ģ�
��

# ��Щ�����Ľӿ�����ι����ģ�
<linux/spi/spi.h>ͷ�ļ�����kernel doc������ҪԴ���룬��Ӧ���Ķ��ں�API�ĵ�������ֻ��һ���������������õ���һ����ŵ��˽����˽�ϸ��֮ǰ��

SPI�������ǽ���I/O���С��Ը���SPI�豸���������ǰ�FIFO˳��ִ�У�����ͨ���ص������첽��������󡣻���һЩ�򵥵�ͬ����װ����������Щ���ã�����һ��������transaction���ͣ��緢��һ�����Ȼ���ȡ��Ӧ��

������SPI driver���ͣ�����
- Controller drivers��Controller���Ա�������SOC�ϣ�����ͨ��֧����/��ģʽ����Щ������Ҫ�Ӵ�Ӳ���Ĵ��������ҿ���ʹ��DMA�������ǿ��Գ�ΪPIO bitbangers������ʹ��GPIO��
- Protocol drivers����Щ����ͨ��������������Ϣ���Ա���SPI���ϵ���/���豸ͨ�š�

���磬һ��Protocol driver������MTD��ͨ�ţ������ݵ������洢��SPI flash����DataFlash���ϵ��ļ�ϵͳ����������һЩ��������������������Ƶ�ӿڣ���������������Ϊ��Ϊ����ӿڣ����������й����м���¶Ⱥ͵�ѹ������д�������ܹ���һ��Controller drivers��

����һ��spi_device�ṹ�壬������driver֮�䣬��װ��һ���������˵Ľӿڡ�

SPI��̽ӿ���һ����С�ĺ��ģ���Ҫ������ͨ��ʹ�ð弶�ĳ�ʼ�����룬��ͨ������ģ������contronller��protocol������SPI ������sysfs�еĶ��λ�ã�
- /sys/devices/.../CTLR  ����SPI������������ڵ�
- /sys/devices/.../CTLR/spiB.C  ������S"B",Ƭѡ"C"�ϵ�spi_device��ͨ��CTLR����
- /sys/bus/spi/devices/spiB.C  �����豸.../CTLR/spiB.C�ķ�������
- /sys/devices/.../CTLR/spiB.C/modalias ��ʶӦ���deviceһ��ʹ�õ�driver(�������Ȳ��)
- /sys/bus/spi/drivers/D  ����һ������spi*.*��driver
- /sys/class/spi_master/spiB һ���߼��ڵ�ķ������ӣ���ʵ�ʵ��豸�ڵ㣩��������Ϊ�������ߡ�B����SPI��������hold��״̬��ص��ࡣ���е�spiB.*�豸����һ������SPI���߶Σ���SCLK��MOSI��MISO
- /sys/devices/.../CTLR/slave һ�������ļ�������ΪSPI�ӿ�����(ȡ��ע��)ע����豸����SPI slave Handler��driver����д�뵽����ļ���ע����豸��дnull��ȡ��ע����豸��
- /sys/class/spi_slave/spiB һ���߼��ڵ�ķ������ӣ���ʵ�ʵ��豸�ڵ㣩��������Ϊ�������ߡ�B����SPI�ӿ�����hold��״̬��ص��ࡣ��ע���֮��һ��spiB.*�豸���������������������SPI���豸��������SPI���߶�

��ע�⣬����������״̬��ʵ��λ��ȡ�������Ƿ�������CONFIG_SYSFS_DEPRECATED����ʱ��Ψһ�����йص�״̬�����ߺţ�����"spiB "�е� "B"����������Щsysclass��Ŀֻ�Կ���ʶ���������á�
# �弶�ĳ�ʼ�������������SPI�豸
Linux��Ҫ�����Ϣ����ȷ����SPI�豸����Щ��Ϣͨ�����������صĴ����ṩ����ʹ��ĳЩ֧���Զ�����/ö�ٵ�оƬҲ����ˡ�

## ����Controller
��һ����Ϣ�Ǵ�����ЩSPI���������б����ڻ���SOC�İ��ӣ�controllerͨ����platform devices������controller��ҪһЩplatform_data���������������ṹ��platform_device�����һЩ��Դ�������һ���������������ַ��IRQ�š�

platformͨ�������"register SPI controller"��һ���������ܽ������������Գ�ʼ���������ã��Ա������ӵ�arch/.../mach-*/board-*.c�ļ����Թ�����ͬ�Ļ������������ô��롣������Ϊ�����soc���ж��SPI�Ŀ�������ͨ��ֻӦ���ú�ע�����������ʵ�ʿ��õĿ�������

�ٸ����ӣ�arch/.../mach-*/board-*.c�ļ����ܺ����������Ĵ��룺
```
#include <mach/spi.h>	/* for mysoc_spi_data */

	static struct mysoc_spi_data pdata __initdata = { ... };

	static __init board_init(void)
	{
		...
		/* ������ӽ���ʹ��SPI Controller #2 */
		mysoc_register_spi(2, &pdata);
		...
	}
```
�����ض���SOC��ʵ�ó�����������
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

            /* ���ǣ���ʼ������ģʽ���Ա�spi2�ź��ܹ�����ص������Ͽɼ���
             * �����İ����ϵ�bootloaders�����Ѿ�������Щ������
             * ���ǿ�������ͨ����Ҫlinux������Щ������
			 */
		}
		...
	}
```

��ע�⣬��ʹʹ����ͬ��SOC�����������ӵ�platform_dataҲ���ܲ�ͬ�� ���磬��һ������ϣ�SPI���ܻ�ʹ���ⲿʱ�ӣ�����һ�������SPI���������ڵ�ǰ���õ���ʱ�ӡ�

## �������豸
�ڶ�����Ϣ��Ŀ����ϴ�����ЩSPI���豸���б�ͨ����ҪһЩ�弶�����ݲ���ʹ������������������

ͨ��arch/.../mach-*/board-*.c�ļ������ṩһ���б��г���ÿ�������ϵ�SPI�豸�����ǣ�
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
ͬ������ע������ṩ�弶����Ϣ��ÿ��оƬ������Ҫ�������͡��������չʾ��һ��ͨ�õ�Լ��������������������SPIʱ�����ʣ���IRQ����������ӣ������ض���оƬ�����ƣ�����һ���������ϵĵ��ݾ������ӳ�ֵ��

����"controller_data"��һ����controller driver���õ���Ϣ��һ���������ض��������DMA�������ݻ���Ƭѡ�ص���������Щ��Ϣ��֮�󴢴���spi_device�С�

board_info��Ҫ�ṩ�㹻����Ϣ��ʹϵͳ�ڲ�����оƬ�������������¹������������鷳�Ŀ�����spi_device.mode�ֶ��е�SPI_CS_HIGHλ����Ϊ��һ������󡱽���Ƭѡ���豸���������ǲ����ܵģ����ǻ�����ʩ֪�����deselect��

Ȼ����İ弶��ʼ�����뽫��ע������б�SPI����ʵʩ�С��������Ժ�ע��SPI master controllerʱ����Щ��Ϣ�����ˡ�
```
spi_register_board_info(spi_board_info, ARRAY_SIZE(spi_board_info));
```
��������̬�弶��ʼ��һ�����㲻��ȡ��ע����Щ��Ϣ��

## �Ǿ�̬����
������(Developer boards)���Ʒ��(Product boards)������ѭ��ͬ�Ĺ���һ�������Ǵ���һ��Ǳ�ڵ������Ǿ�����Ҫ�Ȳ��SPI�豸�Ϳ�������

����������У��������Ҫʹ��spi_busnum_to_master()��Ѱ��spi_master�������ƺ���Ҫspi_new_device()���ṩ�Ѿ�������Ȳ���豸��board_info����Ȼ�ڰ����Ƴ�֮������Ҫ����spi_unregister_device()��

# ���дһ��SPI Protocol Driver��
�����SPI����Ŀǰ�����ں���������Ҳ֧���û��ռ���������������ǽ��������ں�������

SPI Protocol Driver�е�������ƽ̨�豸������
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
�������Ľ����Զ������������board_info��modaliasΪ��CHIP�����κ�SPI�豸�С����probe()������ܿ������������������㴴�����豸�����ڹ������ߵģ�������/sys/class/spi_master����
```
static int CHIP_probe(struct spi_device *spi)
	{
		struct CHIP			*chip;
		struct CHIP_platform_data	*pdata;

		/* ����������Ҫ�弶����: */
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
һ������probe(),�����ͻ�ʹ�ýṹ��"spi_message"����I/O����SPI�豸����remove()���ػ���probe()ʧ�ܵ�ʱ�������ᱣ֤���ٷ�����Ϣ��
- һ��spi_message��һϵ��Э�����������Ϊһ��ԭ������ִ�С�spi driver�������²�����
    + ��˫�����д��ʼ��ͨ��spi_driver����spi_transfer��������������
    + spi_driver����ʹ����һ��I/O buffer��ÿ��spi_transfer��װ��һ��buffer���ڲ�ͬ���䷽��֧��ȫ˫���Ͱ�˫�����䡣
    + spi_driver��ѡ�ض�����һ�����ӳ��ڴ���֮��ʹ��spi_transfer.delay_usecs�������ӳ١�
    + spi_driver�����ڴ�����ӳ�֮��Ƭѡ�Ƿ�inacive��ͨ��spi_transfer.cs_change��־�������á�
    + spi_driver��ʾ��һ����Ϣ�Ƿ����Ҳ���͵���ͬ���豸��ʹ��ԭ�Ӷ��е����һ�δ����е�spi_transfer.cs_change��־�������á�����Ǳ�ڵؽ�ԼƬѡdeslect��select����֮����������ġ�
- �����message�У�Ӧ��ѭ��׼��kernel���򣬲����ṩ"DMA��ȫ"��buffer������ʹ��DMA��controller drivers�Ͳ���ǿ�ƽ��ж���Ŀ���������Ӳ��Ҫ������ô����

�����Щbuffer���ʺ�ʹ�ñ�׼��dma_map_single()����������ʹ��spi_message.is_dma_mapped������controller driver���Ѿ��ṩ��һ����ص�DMA��ַ��

- ����I/O��ԭ������spi_async()���첽(asnyc)����������κ��������б����(irq�����������񣬵ȵ�)�����첽�������������ͨ��ʹ����message�ṩ�Ļص��������б��档�ڼ�⵽����֮��Ƭѡ�ᱻdeselct����spi_message�Ĵ�����̻ᱻȡ����
- ͬ��Ҳ����ͬ���İ�װ��������spi_sync(),�Ͱ�װ��spi_read()��spi_write(),spi_write_then_read().��Щ����ֻ���ڿ�˯�ߵ��������б����ã��������Ƕ���ͨ��spi_async()ʵ�ֵġ�
- spi_write_then_read()����������ʵ�ֵİ�װ����Ӧ�ý����������������ϣ���������ĸ��ƾͿ��Ա����ԡ���������֧��RPC��ʽ����������spi_w8r16��������һ����װ����������һ��8bit��������Ҷ�ȡ16bit����Ӧ��

һЩ�豸������Ҫ�޸�spi_deivce���ԣ����紫��ģʽ���ִ�С������ʱ�����ʡ���Щ����ͨ��spi_setup()��ɣ�ͨ���ڵ�һ�� I/O ���豸���֮ǰ�� probe() �е��á�Ȼ����spi_setup()������û��messageҪ������κ�ʱ�䱻���á�

spi_device�������ĵײ㣬�����ϲ���ܰ���sysfs(�ر��Ǵ���������)������㣬ALSA, ����, MTD���ַ��豸��ܣ�����������linux��ϵͳ��

ע�⣬�������������������ڴ����ͣ���Ϊ��SPI�豸������һ���֡�
 - I/O bufferʹ��ͨ����Linux���򣬲��ұ����ǡ�DMA��ȫ���ġ���ͨ����Ҫ�Ӷѻ��ǿ��е�ҳ����з���buffer����Ҫʹ��ջ����������������Ϊ��̬���ڴ档
 - spi_message��spi_transferԪ����(metadata)���ڽ���ЩI/O buffer������һ��protocol transaction����Щ�������κεط������䣬������Ϊ����ֻ����һ�ε����������ݽṹ��һ���֣���Ҫ��ʼ��Ϊ�㡣

 �����Ը�⣬spi_message_alloc()��spi_message_free()�������������Է���ط���һ�����ж�������spi_message�������ʼ��Ϊ�㡣

 # ���дһ��SPI Master controller Driver��
һ��SPI controller ͨ��ע����platform_bus�ϣ�дһ�����������豸�������漰����bus��

������������Ҫ�������ṩһ��spi_master��ʹ��spi_alloc_master()������һ��master��ʹ��spi_master_get_devdata()����ȡ����Ϊ�豸�����˽�����ݡ�
```
struct spi_master	*master;
	struct CONTROLLER	*c;

	master = spi_alloc_master(dev, sizeof *c);
	if (!master)
		return -ENODEV;

	c = spi_master_get_devdata(master);
```
�������򽫳�ʼ����spi_master���ֶΣ��������ߺ�(������ƽ̨�豸ID��ͬ)������������SPI�˺�SPIЭ�����������ĺ������������򻹻��ʼ��������ص��ڲ�״̬��

�����ʼ��spi_master֮��Ȼ��ʹ��spi_register_master()������spi_master��ϵͳ���������֡���ʱ��controller���豸�ڵ������Ԥ��������SPI�豸�������ã���ʱ����ģ�ͺ��Ľ������豸�󶨵������ϡ�

�������Ҫ�Ƴ�spi������������spi_unregister_master()��������spi_register_master()��

# �����߱��
�����߱�ź���Ҫ����ΪLinux��������ʶ��һ��������SPI���ߣ�����SCK��MOSI��MISO������Ч���ߺŴ�0��ʼ����SOC�ϣ����ߺ���Ҫ��оƬ�����̶���ı����ͬ�����磬������SPI2�����ߺž���2���������ӵ����������豸��spi_board_info����ʹ�������š�

�����û��������Ӳ����������ߺţ���������ĳ��ԭ���㲻�������䣬��ô�ṩһ���������ߺš� �����ͻᱻһ����̬����ĺ�����ȡ����Ȼ������Ҫ������Ϊ�Ǿ�̬���ã������ģ�

# SPI Master ����
- master->setup(struct spi_device *spi)
	
	���������豸ʱ�����ʡ�SPIģʽ���ִ�С������������Ըı� board_info �ṩ��Ĭ��ֵ��Ȼ�����spi_setup(spi)������������̡� ���ܵ���˯�ߡ�
	
	����ÿ��SPI�ӻ������Լ������üĴ���������Ҫ�����������ǣ���������������ܻ�������SPI�豸���ڽ��е�I/O��

- master->cleanup(struct spi_device *spi)

	����controller �������ܻ�ʹ�� spi_device.controller_state ������������豸��̬������״̬�����������������ȷ���ṩcleanup()�������ͷŸ�״̬��
- master->prepare_transfer_hardware(struct spi_master *master)
	
	�����������л��Ƶ��ã���������������message�����������źţ������ϵͳͨ�������˵���������������׼������Ӳ��������ܵ���˯�ߡ�

- master->unprepare_transfer_hardware(struct spi_master *master)

	�����������л��Ƶ��ã������������������в����д��������Ϣ���źţ��������п��ܷ���Ӳ�������磬ͨ����Դ������ã�������ܻᵼ��˯�ߡ�
	
- master->transfer_one_message(struct spi_master *master,struct spi_message *mesg)

	��ϵͳ������������������һ��message��ͬʱ���ڴ��ڼ䵽��Ĵ�������Ŷӡ�������������������Ϣ�����������spi_finalize_current_message()��������ϵͳ���ܷ�����һ����Ϣ������ܻᵼ��˯�ߡ�

- master->transfer_one(struct spi_master *master, struct spi_device *spi,struct spi_transfer *transfer)

	��ϵͳ������������������һ��transfer��ͬʱ���ڴ��ڼ䵽��Ĵ�������Ŷӡ�������������������������������spi_finalize_current_transfer()��������ϵͳ���ܷ�����һ��transfer��������ϵͳ���ܷ�����һ�δ��䡣����ܵ���˯�ߡ�ע�⣺transfer_one��transfer_one_message���໥�ų�ģ������߶�������ʱ��ͨ����ϵͳ�������transfer_one�ص���
	
	����ֵ:
	- ��ֵ������
	- 0���������
	- 1�����������

- master->set_cs_timing(struct spi_device *spi, u8 setup_clk_cycles,u8 hold_clk_cycles, u8 inactive_clk_cycles)

	�÷�������SPI client��������SPI master controller�����豸�ض���CS���á����ֺͷǻʱ��Ҫ��

# �����÷���
- master->transfer(struct spi_device *spi, struct spi_message *message)
	��һ�����ᵼ��˯�ߡ�����ְ���ǰ��Ŵ���ķ��������Ұ���complete()�ص������á�������ͨ����������������ɺ���������������ǿ��еģ�������Ҫ���������˷����������Ŷӿ����������ʵ����transfer_one_message()��(un)prepare_transfer_hardware()����˷�������ΪNULL��

# SPI ��Ϣ����
�������SPI��ϵͳ�ṩ�ı�׼�Ŷӻ������⣬ֻ��ʵ������ָ�����Ŷӷ�����ʹ����Ϣ���еĺô��Ǽ����˴����Ĵ��룬���ṩ�˴�����������ִ�еķ��������ڸ����ȼ�SPIͨ�ţ���Ϣ����Ҳ��������Ϊʵʱ���ȼ���

����ѡ��SPI��ϵͳ�еĶ��л��ƣ�����󲿷ֵ��������򽫹����������Ѿ������ĺ���transfer()�ṩ��IO���С�

������п����Ǵ������Եġ����磬һ��ֻ���ڷ��ʵ�Ƶ����������������ʹ��ͬ��PIO�Ϳ����ˡ�

