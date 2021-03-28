# ƽ̨�豸
ƽ̨�豸��һ���豸������ϵͳ��ͨ����Ϊ������ʵ����֡���������ͳ�Ļ��ڶ˿�(IO)���豸��������Χ���ߵ������ţ��Լ�������SOC�Ĵ����������������ͨ���Ĺ�ͬ����ֱ�Ӵ�CPU����Ѱַ����������£�һ��ƽ̨�豸����ͨ������ĳ�����ߵ�һ�����ӣ������ļĴ�����Ȼ����ֱ��Ѱַ��

ƽ̨�豸������һ�����ƣ����������󶨣��Լ�һ����Դ�б����ַ��IRQ:
```
struct platform_device {
      const char      *name;
      u32             id;
      struct device   dev;
      u32             num_resources;
      struct resource *resource;
};
```
# ƽ̨����
ƽ̨����������ѭ��׼��������ģ��Լ��������/ö�ٲ��������������ⲿ���У����������ṩprobe������remove��������������֧��ʹ�ñ�׼Լ���ĵ�Դ����͹ػ�֪ͨ��
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
��ע�⣬probe()ͨ��Ӧ��ָ֤�����豸Ӳ���Ƿ�ȷʵ���ڣ���ʱƽ̨���ô����޷�ȷ����probe����ʹ���豸��Դ������ʱ�Ӻ��豸ƽ̨���ݡ�
```
int platform_driver_register(struct platform_driver *drv);
```
���ߣ����豸��֪�����Ȳ�εĳ�������£�probe()���̿��Է���init�����У��Լ����������������ʱ�ڴ�ռ�á�
```
int platform_driver_probe(struct platform_driver *drv,int (*probe)(struct platform_device *))
```
kernelģ����������ƽ̨������ɡ�platform core�ṩhelper��ע���ע��һϵ�е�������
```
int __platform_register_drivers(struct platform_driver * const *drivers,
                              unsigned int count, struct module *owner);
void platform_unregister_drivers(struct platform_driver * const *drivers,
                                 unsigned int count);
```
������е�һ������ע��ʧ�ܣ���ô��ĿǰΪֹע����������������෴��˳��ע������ע����һ������ĺ꣬�䴫��THIS_MODULE��Ϊ�����߲�����
```
#define platform_register_drivers(drivers, count)
```
# �豸ö��

��Ϊһ�������ض���ƽ̨(ͨ�����ض��ڰ���)��setup�����ע��ƽ̨�豸��
```
int platform_device_register(struct platform_device *pdev);

int platform_add_devices(struct platform_device **pdevs, int ndev);
```
ͨ���Ĺ����ǽ���ע����Щȷʵ���ڵ��豸��������һЩ����¶�����豸Ҳ�ᱻע�ᡣ�ٸ����ӣ�kernel���ܱ�������������������������һ������������������ܲ��ڰ����ϡ�����kernel���ܱ�����������һ�����ɿ�����(�ⲿ��һ������)һ�����������ӱ���û�������κ����衣

��һЩ����£������̼��ᵼ��һ��������������������Щ�豸�ı�����û����������ôͨ��ϵͳsetup����������ȷ�豸��Ψһ��ʽ��Ϊ�ض��İ���buildһ��kernel���������ض�����ӵ��ں���Ƕ��ʽ�Ͷ���ϵͳ�����кܳ�����

���������£���ƽ̨�豸��ص��ڴ��IRQ��Դ���������豸�����������������ӵ����ô���ͨ����ʹ���豸platform_data�ֶ��ṩ���ӵ���Ϣ��

Ƕ��ʽϵͳͨ����ҪΪƽ̨�豸�ṩһ������ʱ�ӡ���Щʱ��ͨ���Ǳ��ֹرյģ�ֱ��������Ҫ���ǣ��Խ�ʡ��Դ����ϵͳ����Ҳ����Щʱ�����豸������������������clk_get(&pdev->dev, clock_name)�Ϳ��Ը�����Ҫ������Щʱ�ӡ�

# ��ͳ�������豸̽��
��Щ��������û����ȫת��Ϊ��������ģ�ͣ���Ϊ���ǳе���һ������������Ľ�ɫ���������豸����ϵģ���ƽ̨�豸������ע�ᣬ������������ϵͳ������ʩע�ᡣ���������������ܱ��Ȳ�λ����Σ���Ϊ��Щ��λ���Ҫ���豸�Ĵ������豸���������ڲ�ͬ��ϵͳ����еġ�

��ô����Ψһ�ô�������������Щ�ɵ�ϵͳ��ƣ���Щ��ϵͳ�������IBM PCһ�����������׳���� "̽��Ӳ�� "ģʽ����Ӳ�����á����µ�ϵͳ�������Ѿ�����������ģʽ��������������Ϊ֧�ֵĶ�̬���ã�����PCI��USB���������������̼��ṩ���豸������x86�ϵ�PNPACPI��������ʲô���������������̫���໥ì�ܵ�ѡ�񣬼�ʹ�ǲ���ϵͳ�и��ݵĲ²�Ҳ�ᾭ�������Ӷ�����鷳��

���ַ������������ǲ������ġ������Ҫ�������������������볢��������֮�⣬���豸ö���Ƶ�һ�������ʵ�λ�á���������ͨ���ᱻ������Ϊ�������������Ѿ����� "����"��ģʽ����������ʹ����PNP����ƽ̨�豸���ô������豸�ڵ㡣

������ˣ�������һЩAPI��֧����������������������ʹ����Щ���ã�����ʹ�����ִ����Ȳ��ȱ�ݵ�������

```
struct platform_device *platform_device_alloc(const char *name, int id);
```
�����ʹ��platform_device_alloc()����̬�ķ���һ���豸��Ȼ����resources��platform_device_register()��������ʼ������һ�����õĽ������ͨ����ʹ�ã�
```
struct platform_device *platform_device_register_simple(
                const char *name, int id,
                struct resource *res, unsigned int nres);
```
�����ʹ��platform_device_register_simple()һ���������ע���豸��

# �豸������������
platform_device.dev.bus_id���豸�Ĺ淶���ơ��������������ֹ����ģ�
- platform_device.name ,��Ҳ����������ƥ�䡣
- platform_device.id �豸ʵ���ţ�������"-1 "��ʾֻ��һ����

������������������һ��ʹ�õģ�name=serial/id=0����ʾbus_id"serial.0"��name=serial/id=3 ��ʾbus_id ��serial.3�������Ƕ�ʹ����Ϊserial��platform_driver����name=my_rtc/id=-1��bus_id����"my_rtc"(û��ʵ����)������ʹ�ý���"my_rtc"��platform_driver.

����������������������Զ�ִ�еģ����ҵ��������豸֮���ƥ��֮��������probe()�ͻᱻִ�С���probe()ִ�гɹ�֮���������豸�ͻ���ͨ��һ��������һ�������ֲ�ͬ�ķ�����Ѱ������ƥ�䡣

- ÿ��һ���豸��ע��󣬸����ߵ������ͻ����Ƿ�ƥ�䡣 ƽ̨�豸Ӧ����ϵͳ�����ǳ����ھͱ�ע�ᡣ
- ��ʹ��platform_driver_register()ע��һ������ʱ�����и�������δ�󶨵��豸��Ӧ������Ƿ�ƥ�䡣��������ͨ��������������ע�ᣬ����ͨ��ģ����ء�
- ʹ��platform_driver_probe()ע�������������ʹ��platform_driver_register()һ������ͬ���ǣ�ʹ��platform_driver_probe()������������豸ע�ᣬ�����Ժ�Ͳ��ᱻprobe����(���Ǻõģ���Ϊ�˽ӿڽ����ڷ��Ȳ���豸��)

# ���ڵ�ƽ̨�豸����������
���ڵ�platform�ӿڣ���ϵͳ����������Ϊƽ̨�豸�����ṩplatform data����δ��뽨����early_param()�����н���֮�ϣ����Ժ����ִ�С�

����: ע�ᡰearlyprintk�� �����ڴ��ڿ���̨�������ֲ���

1. ע������ƽ̨�豸data

      �ܹ�����ʹ�ú���early_platform_add_devices()ע��ƽ̨�豸��data�������ڴ��ڿ���̨�������У����dataӦ���Ǵ��ڵ�Ӳ�����á������ʱ���ע����豸��֮�󽫱���ƽ̨����ƥ�䡣

2. �����ں�������

      �ܹ��������parse_early_param()�������ں˵������У��⽫ִ������ƥ���early_param()�ص����⽫ִ������ƥ���early_param()�ص����û�ָ��������ƽ̨�豸���ڴ�ʱ��ע�ᡣ�����ڴ��ڿ���̨�������У��û��������ں���������ָ���˿�Ϊ��earlyprintk=serial.0�������С�earlyprintk����string���͡�����serial����ƽ̨���������֣�0��ƽ̨�豸��id�����id��-1����ô���id�����Ա����ԡ�

3. ��װ����ĳһ�������ƽ̨����

      �ܹ��������ͨ��ʹ�ú���early_platform_driver_register_all()��ѡ���Ե�ǿ��ע������ĳ�������������ƽ̨���������û��ڲ���2ָ�����豸�����ȼ�������һ�����豸��������豻��������������ʡ���ˣ���Ϊ���ڵĴ�����������Ӧ���Ǳ����õģ������û����ں���������ָ���˶˿ڡ�

4. ����ƽ̨����ע��

      ʹ��early_platform_init()�����ƽ̨�������ڲ���2��3���Զ�ע�ᡣ���������������У�Ӧ��ʹ��early_platform_init("earlyprintk", &platform_driver)��

5. probe����ĳ���������ƽ̨����

      �ܹ��������early_platform_driver_probe()��ƥ����ע�������ƽ̨�豸(��ĳ���������)����ע�������ƽ̨������ƥ�䵽���豸�ᱻprobed()����һ�����������������ڼ���κ�ʱ��ִ�С����ڴ��ڵ����ӣ�����ִ�п��ܻ���á�

6. �����ڵ�ƽ̨����probe()��

      �����������ڼ䣬�������������Ҫ�ر�С�ģ����������ڴ������ж�ע�᷽�档probe()�����еĴ������ʹ��is_early_platform_device()�����������������ƽ̨�豸ʱ�����û����ڳ����ƽ̨�豸ʱ���á����ڴ���������ʱ��ִ�� register_console()��


## ������Ϣ��μ�<linuxplatform_device.h>��