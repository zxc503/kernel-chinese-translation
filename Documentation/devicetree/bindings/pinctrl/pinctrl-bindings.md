控制引脚复用和参数配置(例如上下拉/三态/驱动)的硬件模块被指定为pinctrl。每个pinctrl必须在设备树中作为一个节点，就像其他硬件模块。

那些信号受引脚配置影响的硬件必须被指定为client设备。再一次，每一个client设备必须在设备树中作为一个节点，就像其他硬件设备。

为了使一个client设备正常运行，某个pinctrl必须设置具体的引脚配置。一些client设备需要一个单独的静态配置，例如在初始化期间设置引脚配置。有一些client设备需要在运行时重新配置引脚，例如在设备处于非活动状态时配置为三态。因此每个client设备可以定义一组命名的state。这些state的数量和名字由client设备自身的绑定定义。

这个文件中定义的通用pinctrl绑定为client设备提供了一个基础框架，用来将这些state名称映射到这些state所使用的引脚配置。

注意，pinctrl本身也可以作为自身的client设备。例如，当驱动加载的时候，一个pinctrl可能会配置成自身的"active"state。这将允许在一个地方表示一个板子的静态引脚配置，而不是将其分割在多个客户端设备节点上。是否要这样做取决于板子设备树的作者，以及板子上使用的client设备的绑定需求，即他们是否需要某些特定的命名state，用于动态引脚配置。

# Pinctrl client devices
对于每个独立的client设备，每个pin state被指定了整数ID。这些整数从0开始并且是连续的。对于每个state ID，存在一个唯一的属性用来定义引脚配置。每一个state必须分配一个名称。当名称被使用时，存在另一个属性映射，从这些名称映射到整数ID。

每个client设备自身的绑定决定了其设备树节点中必须定义的state集，以及是否定义必须提供的state ID集，或者是否定义必须提供的state名称集。

需要的属性：</br>
+ pinctrl-0: phandles列表，每一项指向一个引脚配置节点。相关的引脚配置必须作为其配置的pinctrl的子节点。可能在这个列表中存在多个项目，这样就可以配置多个pinctrl，或者可以从多个节点为单个pinctrl建立一个状态，每个节点都是整体配置的一部分。关于这些引脚配置节点的格式相关的细节请查看下一小节。</br>

    在一些情况下，定义一个空的state是很有用的。当在SOC中使用一个普通的IP block，而不通过pinctrl时，或者pinctrl不对硬件模块产生影响时，这个state可能会被请求。如果IP block的绑定需要某个pin state存在时，这些state仍然需要被定义，但是是空的。

可选的属性：</br>
+ pinctrl-1：phandles列表，每一项指向pinctrl内的一个引脚配置节点</br>
...</br>
+ pinctrl-n：phandles列表，每一项指向pinctrl内的一个引脚配置节点</br>
+ pinctrl-names：名称列表，用于指定state。列表条目0定义了state ID 0对应的名称，列表条目1定义了ID 1对应的名称，以此类推。</br>

例如：
```
/* For a client device requiring named states */
	device {
		pinctrl-names = "active", "idle";
		pinctrl-0 = <&state_0_node_a>;
		pinctrl-1 = <&state_1_node_a &state_1_node_b>;
	};

	/* For the same device if using state IDs */
	device {
		pinctrl-0 = <&state_0_node_a>;
		pinctrl-1 = <&state_1_node_a &state_1_node_b>;
	};

	/*
	 * For an IP block whose binding supports pin configuration,
	 * but in use on an SoC that doesn't have any pin control hardware
	 */
	device {
		pinctrl-names = "active", "idle";
		pinctrl-0 = <>;
		pinctrl-1 = <>;
	};
```

# Pin controller devices
必须的属性：查看pinctrl驱动相关的文档

可选的属性：
+ pinctrl-cells:除了pinctrl设备实例中的索引外，pinctrl单元的数量。
+ pinctrl-use-default:布尔值。指示OS可以使用默认引脚配置引导。这允许使用不包含pinctrl驱动的OS。这个属性既可以设置为pinctrl的全局属性，也可以在子节点中为单个pin group设置。

pinctrl设备需要包含client设备涉及到的引脚设置节点。

例如:
```
	pincontroller {
		... /* Standard DT properties for the device itself elided */

		state_0_node_a {
			...
		};
		state_1_node_a {
			...
		};
		state_1_node_b {
			...
		};
	}
```
每一个引脚配置子节点的内容是由每个pinctrl设备的绑定决定的。节点内容不存在通用的标准。pinctrl框架只提供pinctrl驱动使用的通用的辅助绑定，

引脚配置节点不需要直接作为pinctrl设备的子节点。它们可以也作为孙节点，比如，这是否合法，以及子节点和中间的父节点是否有相互作用，都完全由各个pinctrl设备定义。