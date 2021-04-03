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
如上所示，底层驱动可以通过使用devres简化代码。复杂性被从较少维护的底层驱动转移到了维护较好的高层驱动。并且由于init失败路径与exit路径共享，两者可以得到更多的测试。请注意，当把当前的调用或分配转换为托管的devm_*版本时，你要检查内部操作（如分配内存）是否失败。

