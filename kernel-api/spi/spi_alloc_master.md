# ����
spi_alloc_master

����SPI master controller

# ���
struct spi_master * spi_alloc_master (struct device * dev,unsigned size);

# ����
- dev

    the controller������ʹ��platform_bus��

- size

    ��Ҫallocate��������˽������(�ѱ���ʼ��Ϊ0)��allocate�ڴ��ָ��洢�ڷ����豸��driver_data�ֶ��У�������spi_master_get_devdata�������ָ�롣

# ������

������

# ����
�����������SPI master controller����ʹ�ã�controller������Ψһֱ�ӽӴ�оƬ�Ĵ����ġ�����������ڵ���spi_register_master֮ǰ���ڷ���spi_master�ķ�ʽ��

����ӿ���˯�ߵ��������е��á�

��������ĵ����߸���ָ�����ߺţ������ڵ���spi_register_master֮ǰ��ʼ��master�ķ���������Ӧ�ã�������豸��������֮�󣩵���spi_master_put����ֹ�ڴ�й©��

# ����
�ɹ�ʱ����SPI master�ṹ�壬ʧ�ܷ���NULL