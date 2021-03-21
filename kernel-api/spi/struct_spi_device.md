# ����
struct spi_device

SPI slave�豸��master�����

# ����
```
struct spi_device {
  struct device dev;
  struct spi_master * master;
  u32 max_speed_hz;
  u8 chip_select;
  u8 mode;
    #define SPI_CPHA	0x01
    #define SPI_CPOL	0x02
    #define SPI_MODE_0	(0|0)
    #define SPI_MODE_1	(0|SPI_CPHA)
    #define SPI_MODE_2	(SPI_CPOL|0)
    #define SPI_MODE_3	(SPI_CPOL|SPI_CPHA)
    #define SPI_CS_HIGH	0x04
    #define SPI_LSB_FIRST	0x08
    #define SPI_3WIRE	0x10
    #define SPI_LOOP	0x20
  u8 bits_per_word;
  int irq;
  void * controller_state;
  void * controller_data;
  const char * modalias;
}; 
```
# ��Ա
- dev

    �豸������ģ�ͱ�ʾ��

- master

    �豸ʹ�õ�SPI controller

- max_speed_hz

    ��chipʹ�õ�����ʱ�����ʡ������豸���������ģ�spi_transfer.speed_hzÿ�δ���ʱ���Ը�����ֵ

- chip_select

    Ƭѡ����������master�����chip

- mode

    spiģʽ�������������������ķ�ʽ���ɱ��豸���������ġ�ƬѡĬ�ϵġ�active low�����Ա�����(ͨ��ָ��SPI_CS_HIGH)�����紫����ÿ���ֵ� "MSB���� "Ĭ��ֵһ��(ͨ��ָ��SPI_LSB_FIRST)
- bits_per_word

    ���ݴ������һ�����߶��words��word�Ĵ�Сͨ����8��12bits���ڴ��е��ִ�С�������ֽڵ�����������20λ����ʹ��32λ�����ֳ�����ͨ���豸����������ı䣬����Ĭ��ֵ(0)����ʾ�ֳ���8bit��spi_transfer.bits_per_word������ÿ�δ���ʱ�������ֵ��

- irq

    ���������Ǳ����ݸ�request_irq�����֣����ڽ����豸�жϡ�

- controller_state

    controller������ʱ״̬

- controller_data

    controller�İ弶���壬����FIFO��ʼ������������board_info.controller_data��

- modalias

    ����豸ʹ�õ��������ƣ����Ǳ��������������������ε�sysfs�ġ�modalias�����ԣ��Լ������Ȳ�ε�uevents�С�

# ����
spi_device������SPI slave��CPU�洢��֮�佻�����ݡ���dev��platform_data���ڱ����������豸����Ϣ����Щ��Ϣ���豸��protocol driver���ã����Ƕ��豸��controllerû�á�һ��������һ����ʶ�����ڱ�ʾchip֮��ϸ΢�Ĳ�ͬ����һ�������ǹ�������ض��İ����������оƬ�����ŵġ�