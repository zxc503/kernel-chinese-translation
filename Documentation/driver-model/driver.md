# �豸����
�μ�kernel doc�е�device_driver�ṹ��

# ����
�豸���������Ǿ�̬����Ľṹ����ʹ������ϵͳ�д���һ������֧�ֶ���豸��������ṹ��device_driver���ǽ�������Ϊһ�������ʾ(������һ���ض����豸ʵ��)��

# ��ʼ��
�����������ٳ�ʼ��name��bus�ֶΡ���������Ҫ��ʼ��devclass�ֶ�(������ʱ)�����Կ���Ҫ��ȡ�ʵ����ڲ���ϵ��������Ҫ�����ܳ�ʼ������Ļص���������ʹÿ���ص��������ǿ�ѡ�ġ�

# ����
�����������ṹ��device_driver�����Ǿ�̬����ġ�������һ������eepro100���������ӡ���������Ϊ���裬������������������Ѿ���ȫת��Ϊ��model��
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
������������ܱ���ȫת������model����Ϊ���Ǵ�����busӵ��һ���ض���bus�Ľṹ��������ṹ�����޷�ͨ�û����ض���bus���ֶΡ�

�����������:�豸ID�ṹ��һ������ͨ���ᶨ��һϵ���豸ID(����֧�ֵ�)���豸ID�ṹ�ĸ�ʽ�ͱȽ��豸ID��������ȫ���ض���bus�ġ������Ƕ���Ϊ�ض���bus����Ŀ���������Ͱ�ȫ�ԣ�������Ǳ����ض���bus�Ľṹ��

�ض���bus��������Ҫ�������Ķ����а���һ��ͨ�õĽṹ��device_driver����������
```
struct pci_driver {
       const struct pci_device_id *id_table;
       struct device_driver     driver;
};
```
һ�������ض���bus���ֶεĶ��壬������(�ٴ�ʹ��eepro100������Ϊ����)��
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
����ܻᷢ��Ƕ��ṹ���ʼ�����﷨��ɵ�����ѿ����������ǵ�ĿǰΪֹ�������ҵ�����õķ�����

# ע��
```
int driver_register(struct device_driver *drv);
```
����������ʱע������ṹ�塣����û���ض���bus���ֶ�(��û���ض���bus�����ṹ)�����������ǿ���ʹ��driver_register()���Ҵ���һ��ָ��device_driver�����ָ�롣

Ȼ�������������ӵ��һ���ض���bus�Ľṹ������Ҫʹ����pci_driver_register()֮��ķ�����������ע�ᡣ

# ���ɵ���������
ͨ������wrapper���������Ը����׵ع��ɵ��µ�model������������ȫ����ͨ�ýṹ����bus wrapper��д�ֶΡ����ڻص����������߿��Զ���ͨ�õĻص�������������ת�����ض���bus�Ļص������ϡ�

����������ֻ����ʱ�ġ�Ϊ�˻�������е�class��Ϣ������������ν����޸ġ���Ϊ������ת�����µ�model���Լ��ٻ����ṹ�ĸ����Ժʹ����С���������������Ϣʱ�������ת����

# ����
һ��һ������ע�ᣬ�Ϳ��Է�����������ͨ���ֶΣ����������豸�б�
```
int driver_for_each_dev(struct device_driver *drv, void *data,
                        int (*callback)(struct device *dev, void *data));
```
�豸�ֶ���һ���б�����������Ѿ��󶨵��������豸��LDM�����ṩһ��helper���������������������Ƶ��豸�����helper��ÿ�νڵ��������ʱ��������������ÿ���豸��������ʱ��������ʵ������ü�����

# sysfs
��һ���豸��ע�ᣬ�ͻ�����busĿ¼�´���һ��sysfsĿ¼�������Ŀ¼�У��������Ե���һ���ӿڵ��û��ռ䣬����ȫ�ַ�Χ�ڿ��������Ĳ��������翪�������ڵ�debug�����

���Ŀ¼��δ�����ܽ�����Ϊһ�� "devices"Ŀ¼�����Ŀ¼������symlinks�����ӵ���֧�ֵ��豸��Ŀ¼��

# �ص�����

- int (*probe) (struct device *dev)
       probe()���������������б����õģ�������bus��rwsem�����Լ���������󶨵��豸�ϡ�����ͨ��ʹ��container_of()������dev"ת�����ض���bus�����ͣ���probe()�����������ж�����ˡ���Щ�ض���bus��"dev"ͨ���ṩ�豸��resource data������pci_dev.resource[]��platform_device.resources������dev->platform_data֮�⣬��Щresource dataҲ��������ʼ��������

       ��������ص����������ض����������߼������ڽ���������󶨵������豸����Щ�߼�������֤�豸�Ƿ���ڣ��Ƿ����������Դ���汾�����������ݽṹ�ܷ񱻷���ͳ�ʼ�����Լ�Ӳ���ܷ񱻳�ʼ��������ͨ������һ��ָ������״̬��ָ�룬ͨ��ʹ��dev_set_drvdata()���������ɹ����������豸ʱ��probe()�᷵��0������������ģ�ʹ��뽫�������������򵽸��豸�ϵĲ���.

       ���������probe()���ܻ᷵��һ������errnoֵ����ʾ��������û�а󶨵�����豸������������£���Ӧ���ͷ��������������Դ��


- void (*sync_state)(struct device *dev)
       sync_state()����һ���豸ֻ�ܱ�����һ�Ρ��������������������豸�Ѿ���probe��ʱ��������������á��豸���������б���ͨ���������Ӹ��豸���������豸�����ӣ���õġ�

       ��һ�γ��Ե���sync_state()����late_initcall_sync()�ڼ䷢���ģ���ʹ�̼�������������ʱ���໥�����豸���ڵ�һ�γ��Ե���sync_state()�ڼ䣬��������豸���������ڴ�ʱ���Ѿ�probe�ɹ��ˣ�sync_state()���ϱ����á�����ڵ�һ�γ����ڼ䣬�������豸�����ߣ���Ҳ����Ϊ�ǡ����е��豸�������Ѿ���probe��������sync_state()Ҳ�ᱻ������á�

       ������״γ���Ϊ�豸����sync_state()ʱ����Ҫ������û��probe�ɹ���sync_state()���ûᱻ�ӳ٣�ֻ����һ������������probe�ɹ�ʱ�Ż����³��ԡ����������ķ���һ�����������߻�û�б�probeʱ��sync_state()���û��ٴα��ӳ١�

       һ�� sync_state()�ĵ��������ǣ����ں˸ɾ�����ش�bootloader�нӹ��豸�Ĺ����ٸ����ӣ����һ���豸��bootloader����Ϊһ���ض���Ӳ�����ã���ô�豸������������Ҫ����������ã�ֱ�����е��������豸�����Ѿ�probe�ˡ�һ�����е��������豸����probe�ˣ��豸����������ͬ���豸��Ӳ��״̬����ƥ�������豸��������״̬�����Խ�sync_state()��

       �ٸ����Ե����ӣ�����regulator֮���resource���Դ�sync_state()�����档sync_state()ͬ���Ը��ӵ�resource�����ã�����IOMMUs���ٸ����ӣ�IOMMUs�������������(�豸,���ַ��IOMMU����ӳ��)������Ҫ�������ǵ�ӳ��̶������������ϣ�ֱ�����������߶��Ѿ�probe�ˡ�

       ��Ȼsync_state()�ĵ��������ǣ����ں˸ɾ�����ش�bootloader�нӹ��豸�Ĺ�������sync_state()���÷������ڴˡ����κ���Ҫ������������probe֮���ȡ������Ķ���ʱ��������ʹ�����������

- int (*remove) (struct device *dev); 
       remove������������������豸�İ󶨡����豸����ϵͳ�������Ƴ�������ģ�鱻ж�ء��������ڼ䣬������������£�������������ܱ����á�

       �����������豸�Ƿ���ڡ����������Ҫ�ͷ�Ϊ����豸�ر������κ���Դ�����磬driver_data�ֶ��е����ж�����
       ����豸�����ڣ�Ӧ�þ�Ĭ�豸�����������ڵ͹���״̬��

- int (*suspend) (struct device *dev, pm_message_t state)
       suspend��������ʹ�豸���ڵ͹���״̬

- int (*resume) (struct device *dev)
       resume����ʹ�豸�ӵ͹���״̬�ָ���

# ����
```
struct driver_attribute {
        struct attribute        attr;
        ssize_t (*show)(struct device_driver *driver, char *buf);
        ssize_t (*store)(struct device_driver *, const char *buf, size_t count);
};
```
�豸��������ͨ��sysfsĿ¼�������ԡ�<br/>
��������ʹ��DRIVER_ATTR_RW��DRIVER_ATTR_RO�����������ԣ��乤��ԭ����DEVICE_ATTR_RW��DEVICE_ATTR_RO����ͬ��

���ӣ�
```
DRIVER_ATTR_RW(debug);
```
���൱��������
```
struct driver_attribute driver_attr_debug;
```
Ȼ�����ʹ��������������������Ŀ¼����Ӻ�ɾ������:
```
int driver_create_file(struct device_driver *, const struct driver_attribute *);
void driver_remove_file(struct device_driver *, const struct driver_attribute *);
```