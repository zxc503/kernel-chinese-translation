# ����
SPI���߿�����SPI controller�豸��һ���ڵ��������ÿ��SPI slave�豸��һ���ӽڵ���������ϵͳ��SPI���������Ա�����Ϊʹ��SPI masterģʽ����SPI slaveģʽ��������ͬʱʹ�á�

# ����
-  nodename

    ��ʽ�� "^spi(@.\*|-[0-9a-f])/\*$"
-  address-cells

    enum: [0, 1]

- size-cells

    const: 0

- cs-gpios

    ��ΪƬѡ��GPIO��

    ���ʹ����������ԣ�Ƭѡ�ı�Ž��Զ�����max���ӡ�

    �ٸ����ӣ�controller��4��Ƭѡ�ߣ�cs-gpio��������������
    ```
    cs-gpios = <&gpio1 0 0>, <0>, <&gpio1 1 0>, <&gpio1 2 0>;
    ```
    ֮��Ƭѡ�ᱻ���ã�����num_chipselectӦ��Ϊ4�����ҵõ�����ӳ�䣺
    ```
    cs0 : &gpio1 0 0
    cs1 : native
    cs2 : &gpio1 1 0
    cs3 : &gpio1 2 0
    ```
    GPIO�������ĵڶ���flag������GPIO_ACTIVE_HIGH(0)����GPIO_ACTIVE_LOW(1)���ɵ��豸��ͨ��ʹ��0.

    ��һ������Ĺ��򼯣�����cs-gpio�ĵڶ���flag��SPI�ӻ��п�ѡ��spi-cs-high flag���ϡ�

    ���е�ÿ�����CS���ŵ�����������ʽ(������pinmuxǱ�ڵ�gpio��ת)��
    ```
    device node     | cs-gpio       | CS pin state active | Note
    ================+===============+=====================+=====
    spi-cs-high     | -             | H                   |
    -               | -             | L                   |
    spi-cs-high     | ACTIVE_HIGH   | H                   |
    -               | ACTIVE_HIGH   | L                   | 1
    spi-cs-high     | ACTIVE_LOW    | H                   | 2
    -               | ACTIVE_LOW    | L                   |s

    ```
    ע�⣺

    1. Ӧ�ô�ӡһ�����ڼ��Է�ת�ľ��档������ñ��⣬����gpio����ΪACTIVE_LOW��
    2. Ӧ�ô�ӡһ�����ڼ��Է�ת�ľ��档��ΪACTIVE_LOW�ᱻspi-cs-high���ǣ���ͨ����Ҫ�����⣬��spi-cs-high+ACTIVE_HIGH���档

- num-cs

    Ƭѡ����
- spi-slave

    SPI controller��Ϊslave��������master
# allOf
```
  - if:
      not:
        required:
          - spi-slave
    then:
      properties:
        "#address-cells":
          const: 1
    else:
      properties:
        "#address-cells":
          const: 0
```
# ��׼����
## "^slave$":
 - ����: object
 - ����:
    - compatible��SPI�豸Compatible
 - �����:
    - compatible

##  "^.*@[0-9a-f]+$":
 - ����: object
 - ����:
    - compatible��SPI�豸��Compatible.
    - reg���豸ʹ�õ�Ƭѡ
        - ��Сֵ: 0
        - ���ֵ: 256
    - spi-3wire���豸��Ҫʹ��3-wireģʽ
    - spi-cpha���豸��Ҫʹ��CPHAģʽ
    - spi-cpol���豸��Ҫʹ��CPOLģʽ
    - spi-cs-high���豸��ҪƬѡ�ߵ�ƽ��Ч
    - spi-lsb-first���豸ʹ��LSB����ģʽ
    - spi-max-frequency�����SPIʱ�����ʣ���λ��HZ
    - spi-rx-bus-width��spi��ʱ�õ������߿��
        - enum: [0, 1, 2, 4, 8]
        - default: 1
    - spi-rx-delay-us��������֮�����ʱ����λ��ms
    - spi-tx-bus-width��spiдʱ�õ������߿��
         - enum: [0, 1, 2, 4, 8]
         - default: 1
    - spi-tx-delay-us:д����֮�����ʱ����λ��ms

 - �����:
    - compatible
    - reg
# ����
```
    spi@f00 {
        #address-cells = <1>;
        #size-cells = <0>;
        compatible = "fsl,mpc5200b-spi","fsl,mpc5200-spi";
        reg = <0xf00 0x20>;
        interrupts = <2 13 0 2 14 0>;
        interrupt-parent = <&mpc5200_pic>;

        ethernet-switch@0 {
            compatible = "micrel,ks8995m";
            spi-max-frequency = <1000000>;
            reg = <0>;
        };

        codec@1 {
            compatible = "ti,tlv320aic26";
            spi-max-frequency = <100000>;
            reg = <1>;
        };
    };
```