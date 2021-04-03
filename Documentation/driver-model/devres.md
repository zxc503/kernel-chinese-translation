# 介绍
devres是在试图将libata转化为ioremap时出现的。每一个ioremap过的地址，需要被保持并且在驱动卸载的时候unmapped。举个例子，一个普通的SFF ATA控制器（即旧的PCI IDE）工作在native模式使用了5个PCI BARS，这些BARS都需要被维护。

与其他设备驱动一样，libata底层驱动在remove和probe fail过程中有很多的bug。没错，这可能是因为libata底层驱动的开发者是懒惰的一群人，但不是所有的底层驱动的开发者都是懒惰的吗？花了一整天时间摆弄脑残的硬件，并且没有说明文档或是有但却脑残的文档，如果最终能工作了，那最好了。

出于这样或那样的原因，底层驱动没有被关注，或是像核心代码一样经过测试，并且在驱动卸载和初始化失败时出现的bug不是很常见，所以没有被重视。初始化失败的路径更糟糕，因为他的执行路径更短，却需要处理多个入口点。

所以很多底层驱动在卸载的时候会出现资源泄漏的情况，而且在probe()失败时的执行路径也有一半是坏的的，这在probe发生错误时可能导致资源泄露，甚至导致ops。iomap增加了更多的内容，msi和msix也是如此。

# devres
devres是一个链表，这个链表存放与struct device相关联的任意大小的内存区域。每个devres条目关联了一个release方法。一个devres可以用多种方法被release。无论哪一种方法，所有的devres条目都将在驱动卸载的时候被release。在release时，devres相关联的release方法被执行，之后devres条目被释放。

为resource创建的管理接口，它们通常被使用了devres的设备驱动使用。举个例子，连续的DMA内存通过使用dma_alloc_coherent()方法获得。而管理接口的版本是dmam_alloc_coherent()。它与dma_alloc_coherent()基本相同，除了由它分配的DMA内存是受管理的，并且会在驱动卸载的时候自动被释放。实现如下所示：
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
如果驱动使用dmam_alloc_coherent()，内存区域将被保证无论在初始化半途的时候失败还是驱动卸载的时候都能被释放。如果大多数resources通过管理接口获得，一个驱动的init和exit代码就可以简单多了。init路径基本看起来像下面这样：
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
exit路径像这样：
```
my_remove_one()
{
      unregister_from_upper_layer(d);
      shutdown_my_hardware();
}
```
如上所示，底层驱动可以通过使用devres简化代码。复杂性被从较少维护的底层驱动转移到了维护较好的高层驱动。并且由于init失败路径与exit路径共享，两者可以得到更多的测试。

请注意，当把当前的调用或分配转换为托管的devm_\*版本时，你要检查内部操作（如分配内存）是否失败。管理的资源只与释放这些资源有关--所有其他所需的检查仍然由你负责。在某些情况下，这可能意味着引入了检查，而这些检查在没有转化成托管的devm_\*版本之前不是必须的。

# devres组
devres条目可以使用devres group组合起来。当一个组被release，所有包含的普通devres条目和嵌套的组都将被release。一个用法是，在失败是回滚一连串的resource获取操作。例如：
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
 由于资源获取失败通常意味着probe失败，所以在接口函数不应该对失败有副作用的中层驱动（例如libata核心层）中，上述结构通常很有用。对于底层驱动来说，在大多数情况下，只要返回错误代码就足够了。

 每个组由void *id标识。这既可以由devres_open_group()的id参数显式指定，也可以像上面的例子一样通过传递NULL到id从而被自动创建。在这两种情况下， devres_open_group()都返回组的id。返回的id可以被传递到其他devres方法中，以选择指定的group。如果传递null到这些方法，最后一个打开的group将被选中。

 例如，你可以执行以下操作：
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
# 细节
devres条目的生命周期从devres分配开始，到release/destroy结束(remove和freed)——在没有引用计数的情况下。

devres核心保证所有基础devres操作的原子性，并且支持单例的devres类型。除此之外，同步并发访问已分配的devres资源是调用者的责任。这通常不是问题，因为bus操作和resource分配已经做了这部分操作。

关于单列devres类型的例子，请阅读lib/devres.c中的pcim_iomap_table()。

如果给定了正确的gfp mask，所有的devres接口函数都可以在没有上下文的情况下被调用。

# 开销
每个devres的记录信息与请求的数据区一起分配。在调试选项关闭的情况下，记录信息占用32位机中占用16直接，在64位机中占用24字节。如果单链表被使用，记录信息可以减少到2个指针(32位机8字节，64位机16字节)。

每个devres组占用8个指针。使用单链表可以减少到6个。

带有两个端口的ahci控制器的内存空间开销在32bit机器上经过naive转换后在300到400字节之间。


# 管理接口列表
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