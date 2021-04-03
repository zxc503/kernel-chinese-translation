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
������ʾ���ײ���������ͨ��ʹ��devres�򻯴��롣�����Ա��ӽ���ά���ĵײ�����ת�Ƶ���ά���Ϻõĸ߲���������������initʧ��·����exit·���������߿��Եõ�����Ĳ��ԡ���ע�⣬���ѵ�ǰ�ĵ��û����ת��Ϊ�йܵ�devm_*�汾ʱ����Ҫ����ڲ�������������ڴ棩�Ƿ�ʧ�ܡ�

