# ����
struct spi_master

SPI master Controller�Ľӿ�

# ����
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
# ��Ա
- dev

    ���������豸�ӿ�

- list
    
    ���ӵ�ȫ�ֵ�spi_master list

- bus_num

    �弶��ʶ��(ͨ���ض���SOC)������ָ����SPI������

- num_chipselect

    Ƭѡ�������ֲ�ͬ��SPI slave������Ƭѡ��Ŵ�0��num_chipselects��ÿ��slave��һ��Ƭѡ�źţ���ͨ������ÿ��Ƭѡ�����ӵ�һ��slave��
- dma_alignment

    SPI��������DMA buffer alignment��Լ��

- mode_bits

    controller����������־

- bits_per_word_mask

    һ��mask��ָʾ����֧�ֵ�bits_per_word��ֵ��λn��ʾ����֧��ÿ��word n+1bits����������ˣ�SPI core���ܾ��κβ�֧�ֵ�bits_per_word���䡣���û���ã����ֵ�ͱ����ԣ��ɸ��Ե�������������֤

- min_speed_hz

    ���֧�ֵĴ����ٶ�

- max_speed_hz

    ���֧�ֵĴ����ٶ�

- flags

    ������ص�����Լ������

- max_transfer_size

    ���������ڷ���һ��spi_device��������С�����ܷ���0����ôĬ�ϵ�SIZE_MAX���ᱻʹ��

- bus_lock_spinlock

    ����SPI bus������������

- bus_lock_mutex

    ����SPI bus�����Ļ�����

- bus_lock_flag

    ����ָʾSPI bus�Ѿ����������ڶ�ռ��ʹ��

- setup

    ���������ڸ����豸��SPI controllerʹ�õ��豸ģʽ��ʱ��ģʽ��protocol������ܻ���ô��������������޷�ʶ���֧�ֵ�ģʽ����˲�������ʧ�ܡ����������޸����õ��豸�ϵĴ�����𣬷�����ô��������ǰ�ȫ�ġ�

- transfer

    ���������һ��message��controller�Ĵ������

- cleanup

    ��������release()ʱ�����ã������ͷ�spi_controller�ṩ��memory

- can_dma

    ����������master�Ƿ�֧��DMA

- queued

    ��־λ�����master�Ƿ��ṩ���õ�message����

- kworker

    ����message pump���̵߳Ľṹ��

- kworker_task

    ָ�룬ָ������message pump��kworker�̵߳�task�ṹ��

- pump_messages

    work�ṹ�壬���ڰ���work��message pump

- queue_lock

    ������������ͬ��message���еķ���

- queue

    message ����

- cur_msg

    ��ǰ���ڴ��͵���Ϣ

- idling

    ��־λ���豸�Ƿ�������״̬

- busy

    ��־λ��message pump�Ƿ�æ

- running

    ��־λ��message pump�Ƿ�������

- rt

    ��־λ�������Ƿ����ó���ʵʱ��������

- auto_runtime_pm

    coreӦ��ȷ����Ӳ��׼����ʱ��runtime PM�����Ѿ������У�ͨ��ʹ�ø��豸��Ϊspidev��

- cur_msg_prepared

    ��־λ��trueʱ��spi_prepare_message��������������ǰ�����е���Ϣ��

- cur_msg_mapped

    ��־λ����Ϣ�Ƿ��ѱ�ӳ�䵽DMA

- xfer_completion

    ��transfer_one_messageʹ��

- max_dma_len

    ���DMA����

- prepare_transfer_hardware

    �������������������֪��ϵͳһ��message�����Ӷ����д�����������ϵͳ����������׼������Ӳ����

- transfer_one_message

    ��������ϵͳ�������������䵥��message��ͬʱ���ڴ��ڼ䵽��Ĵ�������Ŷӡ�������������message���������spi_finalize_current_message��������ϵͳ���ܷ���һ������Ϣ��

- unprepare_transfer_hardware

    ��������������������ϵͳ��ǰû�ж�����û�и������Ϣ�ˣ�������ϵͳ����֪ͨ����������Ӳ����Ϣ�ˡ�

- prepare_message

    ����������controller�Դ��䵥����Ϣ��������DMAӳ�䡣�ú������߳������ĵ��á�

- unprepare_message
    
    ����prepare_message�Ĳ���

- spi_flash_read

    ���������������ṩ���ٽӿڵ�spi controller����ȡflash�豸��

- flash_read_supported

    �����������ж�controller�Ƿ�֧��flash��

- set_cs

    ��������������Ƭѡ���߼��㣬���ܻ���ж������ĵ��á�

- transfer_one
    ���������䵥��spi_transfer.���������ɹ�����0��������ڴ��䷵��1�����������������transfer���������spi_finalize_current_transfer��������ϵͳ���ܷ���һ������Ϣ��ע�⣺transfer_one��transfer_one_message���໥�ų�ģ������߶�������ʱ��ͨ����ϵͳ����������transfer_one�ص���

- handle_err

    ��������ϵͳ������������������transfer_one_message��ͨ��ʵ���г��ֵĴ���

- cs_gpios

    ���飬����Ƭѡ�ߵ�GPIO�����顣ÿ��CS����Ϊ�����е�һ��������Ƭѡ�߲���GPIO(��SPI controller��������)����ֵ������-ENOENT��

- statistics

    ��Ϊspi_transfer��ͳ��

- dma_tx

    DMA����ͨ������core dmaengine helpersʹ��

- dma_rx

    DMA����ͨ������core dmaengine helpersʹ��

- dummy_rx

    ���飬����ȫ˫���豸��dummy����buffer

- dummy_tx

    ���飬��ȫ˫���豸��dummy����buffer


- fw_translate_cs

    ��������̼�ʹ�õı�ŷ�����linux�����ı�ŷ�����ͬ�������ѡ�Ĺ��ӿ��Ա�ʹ����������֮��ת����

# ����
ÿ��SPI master controller������һ������spi_deviceͨ�š��⽫��Ϊһ��С�����ߣ����ǹ���MOSI MISO SCK��ȴʹ�ò�ͬƬѡ��ÿ���豸���Ա�������ʹ�ò�ͬ��ʱ�����ʣ���Ϊֱ��Ƭѡ��ѡ�У���Щ�����ź��߶��ᱻ���ԡ�

SPI����������������ͨ��spi_message������й������Щ�豸�ķ��ʣ���CPU�ڴ��SPI���豸֮�临�����ݡ��������Ŷӵȴ���ÿһ����������Ϣ�����������������ʱ������Ϣ����ɺ�����