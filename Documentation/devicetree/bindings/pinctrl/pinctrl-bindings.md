�������Ÿ��úͲ�������(����������/��̬/����)��Ӳ��ģ�鱻ָ��Ϊpinctrl��ÿ��pinctrl�������豸������Ϊһ���ڵ㣬��������Ӳ��ģ�顣

��Щ�ź�����������Ӱ���Ӳ�����뱻ָ��Ϊclient�豸����һ�Σ�ÿһ��client�豸�������豸������Ϊһ���ڵ㣬��������Ӳ���豸��

Ϊ��ʹһ��client�豸�������У�ĳ��pinctrl�������þ�����������á�һЩclient�豸��Ҫһ�������ľ�̬���ã������ڳ�ʼ���ڼ������������á���һЩclient�豸��Ҫ������ʱ�����������ţ��������豸���ڷǻ״̬ʱ����Ϊ��̬�����ÿ��client�豸���Զ���һ��������state����Щstate��������������client�豸����İ󶨶��塣

����ļ��ж����ͨ��pinctrl��Ϊclient�豸�ṩ��һ��������ܣ���������Щstate����ӳ�䵽��Щstate��ʹ�õ��������á�

ע�⣬pinctrl����Ҳ������Ϊ�����client�豸�����磬���������ص�ʱ��һ��pinctrl���ܻ����ó������"active"state���⽫������һ���ط���ʾһ�����ӵľ�̬�������ã������ǽ���ָ��ڶ���ͻ����豸�ڵ��ϡ��Ƿ�Ҫ������ȡ���ڰ����豸�������ߣ��Լ�������ʹ�õ�client�豸�İ����󣬼������Ƿ���ҪĳЩ�ض�������state�����ڶ�̬�������á�

# Pinctrl client devices
����ÿ��������client�豸��ÿ��pin state��ָ��������ID����Щ������0��ʼ�����������ġ�����ÿ��state ID������һ��Ψһ���������������������á�ÿһ��state�������һ�����ơ������Ʊ�ʹ��ʱ��������һ������ӳ�䣬����Щ����ӳ�䵽����ID��

ÿ��client�豸����İ󶨾��������豸���ڵ��б��붨���state�����Լ��Ƿ�������ṩ��state ID���������Ƿ�������ṩ��state���Ƽ���

��Ҫ�����ԣ�</br>
+ pinctrl-0: phandles�б�ÿһ��ָ��һ���������ýڵ㡣��ص��������ñ�����Ϊ�����õ�pinctrl���ӽڵ㡣����������б��д��ڶ����Ŀ�������Ϳ������ö��pinctrl�����߿��ԴӶ���ڵ�Ϊ����pinctrl����һ��״̬��ÿ���ڵ㶼���������õ�һ���֡�������Щ�������ýڵ�ĸ�ʽ��ص�ϸ����鿴��һС�ڡ�</br>

    ��һЩ����£�����һ���յ�state�Ǻ����õġ�����SOC��ʹ��һ����ͨ��IP block������ͨ��pinctrlʱ������pinctrl����Ӳ��ģ�����Ӱ��ʱ�����state���ܻᱻ�������IP block�İ���Ҫĳ��pin state����ʱ����Щstate��Ȼ��Ҫ�����壬�����ǿյġ�

��ѡ�����ԣ�</br>
+ pinctrl-1��phandles�б�ÿһ��ָ��pinctrl�ڵ�һ���������ýڵ�</br>
...</br>
+ pinctrl-n��phandles�б�ÿһ��ָ��pinctrl�ڵ�һ���������ýڵ�</br>
+ pinctrl-names�������б�����ָ��state���б���Ŀ0������state ID 0��Ӧ�����ƣ��б���Ŀ1������ID 1��Ӧ�����ƣ��Դ����ơ�</br>

���磺
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
��������ԣ��鿴pinctrl������ص��ĵ�

��ѡ�����ԣ�
+ pinctrl-cells:����pinctrl�豸ʵ���е������⣬pinctrl��Ԫ��������
+ pinctrl-use-default:����ֵ��ָʾOS����ʹ��Ĭ����������������������ʹ�ò�����pinctrl������OS��������Լȿ�������Ϊpinctrl��ȫ�����ԣ�Ҳ�������ӽڵ���Ϊ����pin group���á�

pinctrl�豸��Ҫ����client�豸�漰�����������ýڵ㡣

����:
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
ÿһ�����������ӽڵ����������ÿ��pinctrl�豸�İ󶨾����ġ��ڵ����ݲ�����ͨ�õı�׼��pinctrl���ֻ�ṩpinctrl����ʹ�õ�ͨ�õĸ����󶨣�

�������ýڵ㲻��Ҫֱ����Ϊpinctrl�豸���ӽڵ㡣���ǿ���Ҳ��Ϊ��ڵ㣬���磬���Ƿ�Ϸ����Լ��ӽڵ���м�ĸ��ڵ��Ƿ����໥���ã�����ȫ�ɸ���pinctrl�豸���塣