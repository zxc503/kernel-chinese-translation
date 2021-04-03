# ����
devres������ͼ��libataת��Ϊioremapʱ���ֵġ�ÿһ��ioremap���ĵ�ַ����Ҫ�����ֲ���������ж�ص�ʱ��unmapped���ٸ����ӣ�һ����ͨ��SFF ATA�����������ɵ�PCI IDE��������nativeģʽʹ����5��PCI BARS����ЩBARS����Ҫ��ά����

�������豸����һ����libata�ײ�������remove��probe fail�������кܶ��bug��û�����������Ϊlibata�ײ������Ŀ������������һȺ�ˣ����������еĵײ������Ŀ����߶���������𣿻���һ����ʱ���Ū�Բе�Ӳ��������û��˵���ĵ������е�ȴ�Բе��ĵ�����������ܹ����ˣ�������ˡ�

����������������ԭ�򣬵ײ�����û�б���ע����������Ĵ���һ���������ԣ�����������ж�غͳ�ʼ��ʧ��ʱ���ֵ�bug���Ǻܳ���������û�б����ӡ���ʼ��ʧ�ܵ�·������⣬��Ϊ����ִ��·�����̣�ȴ��Ҫ��������ڵ㡣

���Ժܶ�ײ�������ж�ص�ʱ��������Դй©�������������probe()ʧ��ʱ��ִ��·��Ҳ��һ���ǻ��ĵģ�����probe��������ʱ���ܵ�����Դй¶����������ops��iomap�����˸�������ݣ�msi��msixҲ����ˡ�

# devres
devres��һ�����������������struct device������������С���ڴ�����ÿ��devres��Ŀ������һ��release������һ��devres�����ö��ַ�����release��������һ�ַ��������е�devres��Ŀ����������ж�ص�ʱ��release����releaseʱ��devres�������release������ִ�У�֮��devres��Ŀ���ͷš�

Ϊresource�����Ĺ���ӿڣ�����ͨ����ʹ����devres���豸����ʹ�á��ٸ����ӣ�������DMA�ڴ�ͨ��ʹ��dma_alloc_coherent()������á�������ӿڵİ汾��dmam_alloc_coherent()������dma_alloc_coherent()������ͬ���������������DMA�ڴ����ܹ���ģ����һ�������ж�ص�ʱ���Զ����ͷš�ʵ��������ʾ��
```
struct dma_devres {
      size_t          size;
      void            *vaddr;
      dma_addr_t      dma_handle;
};

static void dmam_coherent_release(struct device *dev, void *res)
{
      struct dma_devres *this = res;

      dma_free_coherent(dev, this->size, this->vaddr, this->dma_handle);
}

dmam_alloc_coherent(dev, size, dma_handle, gfp)
{
      struct dma_devres *dr;
      void *vaddr;

      dr = devres_alloc(dmam_coherent_release, sizeof(*dr), gfp);
      ...

      /* alloc DMA memory as usual */
      vaddr = dma_alloc_coherent(...);
      ...

      /* record size, vaddr, dma_handle in dr */
      dr->vaddr = vaddr;
      ...

      devres_add(dev, dr);

      return vaddr;
}
```
�������ʹ��dmam_alloc_coherent()���ڴ����򽫱���֤�����ڳ�ʼ����;��ʱ��ʧ�ܻ�������ж�ص�ʱ���ܱ��ͷš���������resourcesͨ������ӿڻ�ã�һ��������init��exit����Ϳ��Լ򵥶��ˡ�init·������������������������
```
my_init_one()
{
      struct mydev *d;

      d = devm_kzalloc(dev, sizeof(*d), GFP_KERNEL);
      if (!d)
              return -ENOMEM;

      d->ring = dmam_alloc_coherent(...);
      if (!d->ring)
              return -ENOMEM;

      if (check something)
              return -EINVAL;
      ...

      return register_to_upper_layer(d);
}
```
exit·����������
```
my_remove_one()
{
      unregister_from_upper_layer(d);
      shutdown_my_hardware();
}
```
������ʾ���ײ���������ͨ��ʹ��devres�򻯴��롣�����Ա��ӽ���ά���ĵײ�����ת�Ƶ���ά���Ϻõĸ߲���������������initʧ��·����exit·���������߿��Եõ�����Ĳ��ԡ�

��ע�⣬���ѵ�ǰ�ĵ��û����ת��Ϊ�йܵ�devm_\*�汾ʱ����Ҫ����ڲ�������������ڴ棩�Ƿ�ʧ�ܡ��������Դֻ���ͷ���Щ��Դ�й�--������������ļ����Ȼ���㸺����ĳЩ����£��������ζ�������˼�飬����Щ�����û��ת�����йܵ�devm_\*�汾֮ǰ���Ǳ���ġ�

# devres��
devres��Ŀ����ʹ��devres group�����������һ���鱻release�����а�������ͨdevres��Ŀ��Ƕ�׵��鶼����release��һ���÷��ǣ���ʧ���ǻع�һ������resource��ȡ���������磺
```
if (!devres_open_group(dev, NULL, GFP_KERNEL))
       return -ENOMEM;

 acquire A;
 if (failed)
       goto err;

 acquire B;
 if (failed)
       goto err;
 ...

 devres_remove_group(dev, NULL);
 return 0;

err:
 devres_release_group(dev, NULL);
 return err_code;
 ```
 ������Դ��ȡʧ��ͨ����ζ��probeʧ�ܣ������ڽӿں�����Ӧ�ö�ʧ���и����õ��в�����������libata���Ĳ㣩�У������ṹͨ�������á����ڵײ�������˵���ڴ��������£�ֻҪ���ش��������㹻�ˡ�

 ÿ������void *id��ʶ����ȿ�����devres_open_group()��id������ʽָ����Ҳ���������������һ��ͨ������NULL��id�Ӷ����Զ�������������������£� devres_open_group()���������id�����ص�id���Ա����ݵ�����devres�����У���ѡ��ָ����group���������null����Щ���������һ���򿪵�group����ѡ�С�

 ���磬�����ִ�����²�����
 ```
 int my_midlayer_create_something()
{
      if (!devres_open_group(dev, my_midlayer_create_something, GFP_KERNEL))
              return -ENOMEM;

      ...

      devres_close_group(dev, my_midlayer_create_something);
      return 0;
}

void my_midlayer_destroy_something()
{
      devres_release_group(dev, my_midlayer_create_something);
}
```
# ϸ��
devres��Ŀ���������ڴ�devres���俪ʼ����release/destroy����(remove��freed)������û�����ü���������¡�

devres���ı�֤���л���devres������ԭ���ԣ�����֧�ֵ�����devres���͡�����֮�⣬ͬ�����������ѷ����devres��Դ�ǵ����ߵ����Ρ���ͨ���������⣬��Ϊbus������resource�����Ѿ������ⲿ�ֲ�����

���ڵ���devres���͵����ӣ����Ķ�lib/devres.c�е�pcim_iomap_table()��

�����������ȷ��gfp mask�����е�devres�ӿں�����������û�������ĵ�����±����á�

# ����
ÿ��devres�ļ�¼��Ϣ�������������һ����䡣�ڵ���ѡ��رյ�����£���¼��Ϣռ��32λ����ռ��16ֱ�ӣ���64λ����ռ��24�ֽڡ����������ʹ�ã���¼��Ϣ���Լ��ٵ�2��ָ��(32λ��8�ֽڣ�64λ��16�ֽ�)��

ÿ��devres��ռ��8��ָ�롣ʹ�õ�������Լ��ٵ�6����

���������˿ڵ�ahci���������ڴ�ռ俪����32bit�����Ͼ���naiveת������300��400�ֽ�֮�䡣


# ����ӿ��б�
### **CLOCK**
devm_clk_get() devm_clk_get_optional() devm_clk_put() devm_clk_bulk_get() devm_clk_bulk_get_all() devm_clk_bulk_get_optional() devm_get_clk_from_childl() devm_clk_hw_register() devm_of_clk_add_hw_provider() devm_clk_hw_register_clkdev()

### **DMA**
dmaenginem_async_device_register() dmam_alloc_coherent() dmam_alloc_attrs() dmam_free_coherent() dmam_pool_create() dmam_pool_destroy()

### **DRM**
devm_drm_dev_alloc()

### **GPIO**
devm_gpiod_get() devm_gpiod_get_array() devm_gpiod_get_array_optional() devm_gpiod_get_index() devm_gpiod_get_index_optional() devm_gpiod_get_optional() devm_gpiod_put() devm_gpiod_unhinge() devm_gpiochip_add_data() devm_gpio_request() devm_gpio_request_one() devm_gpio_free()

### **I2C**
devm_i2c_new_dummy_device()

### **IIO**
devm_iio_device_alloc() devm_iio_device_register() devm_iio_kfifo_allocate() devm_iio_triggered_buffer_setup() devm_iio_trigger_alloc() devm_iio_trigger_register() devm_iio_channel_get() devm_iio_channel_get_all()

### **INPUT**
devm_input_allocate_device()

### **IO region**
devm_release_mem_region() devm_release_region() devm_release_resource() devm_request_mem_region() devm_request_region() devm_request_resource()

### **IOMAP**
devm_ioport_map() devm_ioport_unmap() devm_ioremap() devm_ioremap_uc() devm_ioremap_wc() devm_ioremap_resource() : checks resource, requests memory region, ioremaps devm_ioremap_resource_wc() devm_platform_ioremap_resource() : calls devm_ioremap_resource() for platform device devm_platform_ioremap_resource_wc() devm_platform_ioremap_resource_byname() devm_platform_get_and_ioremap_resource() devm_iounmap() pcim_iomap() pcim_iomap_regions() : do request_region() and iomap() on multiple BARs pcim_iomap_table() : array of mapped addresses indexed by BAR pcim_iounmap()

### **IRQ**
devm_free_irq() devm_request_any_context_irq() devm_request_irq() devm_request_threaded_irq() devm_irq_alloc_descs() devm_irq_alloc_desc() devm_irq_alloc_desc_at() devm_irq_alloc_desc_from() devm_irq_alloc_descs_from() devm_irq_alloc_generic_chip() devm_irq_setup_generic_chip() devm_irq_sim_init()

### **LED**
devm_led_classdev_register() devm_led_classdev_unregister()

### **MDIO**
devm_mdiobus_alloc() devm_mdiobus_alloc_size() devm_mdiobus_register() devm_of_mdiobus_register()

### **MEM**
devm_free_pages() devm_get_free_pages() devm_kasprintf() devm_kcalloc() devm_kfree() devm_kmalloc() devm_kmalloc_array() devm_kmemdup() devm_krealloc() devm_kstrdup() devm_kvasprintf() devm_kzalloc()

### **MFD**
devm_mfd_add_devices()

### **MUX**
devm_mux_chip_alloc() devm_mux_chip_register() devm_mux_control_get()

### **NET**
devm_alloc_etherdev() devm_alloc_etherdev_mqs() devm_register_netdev()

### **PER-CPU MEM**
devm_alloc_percpu() devm_free_percpu()

### **PCI**
devm_pci_alloc_host_bridge() : managed PCI host bridge allocation devm_pci_remap_cfgspace() : ioremap PCI configuration space devm_pci_remap_cfg_resource() : ioremap PCI configuration space resource pcim_enable_device() : after success, all PCI ops become managed pcim_pin_device() : keep PCI device enabled after release

### **PHY**
devm_usb_get_phy() devm_usb_put_phy()

### **PINCTRL**
devm_pinctrl_get() devm_pinctrl_put() devm_pinctrl_register() devm_pinctrl_unregister()

### **POWER**
devm_reboot_mode_register() devm_reboot_mode_unregister()

### **PWM**
devm_pwm_get() devm_pwm_put()

### **REGULATOR**
devm_regulator_bulk_get() devm_regulator_get() devm_regulator_put() devm_regulator_register()

### **RESET**
devm_reset_control_get() devm_reset_controller_register()

### **RTC**
devm_rtc_device_register() devm_rtc_allocate_device() devm_rtc_register_device() devm_rtc_nvmem_register()

### **SERDEV**
devm_serdev_device_open()

### **SLAVE DMA ENGINE**
devm_acpi_dma_controller_register()

### **SPI**
devm_spi_register_master()

### **WATCHDOG**
devm_watchdog_register_device()