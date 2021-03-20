# 描述
SPI总线可以用SPI controller设备的一个节点和总线上每个SPI slave设备的一组子节点来描述。系统的SPI控制器可以被描述为使用SPI master模式还是SPI slave模式，但不能同时使用。

# 属性
-  nodename

    样式： "^spi(@.\*|-[0-9a-f])/\*$"
-  address-cells

    enum: [0, 1]

- size-cells

    const: 0

- cs-gpios

    作为片选的GPIO。

    如果使用了这个属性，片选的编号将自动随着max增加。

    举个例子，controller有4个片选线，cs-gpio将像下面这样：
    ```
    cs-gpios = <&gpio1 0 0>, <0>, <&gpio1 1 0>, <&gpio1 2 0>;
    ```
    之后片选会被配置，所以num_chipselect应该为4，并且得到如下映射：
    ```
    cs0 : &gpio1 0 0
    cs1 : native
    cs2 : &gpio1 1 0
    cs3 : &gpio1 2 0
    ```
    GPIO描述符的第二个flag可以是GPIO_ACTIVE_HIGH(0)或是GPIO_ACTIVE_LOW(1)。旧的设备树通常使用0.

    有一个特殊的规则集，它将cs-gpio的第二个flag与SPI从机中可选的spi-cs-high flag相结合。

    表中的每项定义了CS引脚的物理驱动方式(不考虑pinmux潜在的gpio反转)。
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
    注意：

    1. 应该打印一个关于极性反转的警告。这里最好避免，并将gpio定义为ACTIVE_LOW。
    2. 应该打印一个关于极性反转的警告。因为ACTIVE_LOW会被spi-cs-high覆盖，这通常需要被避免，用spi-cs-high+ACTIVE_HIGH代替。

- num-cs

    片选总数
- spi-slave

    SPI controller作为slave，而不是master
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
# 标准属性
## "^slave$":
 - 类型: object
 - 属性:
    - compatible：SPI设备Compatible
 - 必须的:
    - compatible

##  "^.*@[0-9a-f]+$":
 - 类型: object
 - 属性:
    - compatible：SPI设备的Compatible.
    - reg：设备使用的片选
        - 最小值: 0
        - 最大值: 256
    - spi-3wire：设备需要使用3-wire模式
    - spi-cpha：设备需要使用CPHA模式
    - spi-cpol：设备需要使用CPOL模式
    - spi-cs-high：设备需要片选高电平有效
    - spi-lsb-first：设备使用LSB优先模式
    - spi-max-frequency：最大SPI时钟速率，单位是HZ
    - spi-rx-bus-width：spi读时用到的总线宽度
        - enum: [0, 1, 2, 4, 8]
        - default: 1
    - spi-rx-delay-us：读操作之后的延时，单位是ms
    - spi-tx-bus-width：spi写时用到的总线宽度
         - enum: [0, 1, 2, 4, 8]
         - default: 1
    - spi-tx-delay-us:写操作之后的延时，单位是ms

 - 必须的:
    - compatible
    - reg
# 例子
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