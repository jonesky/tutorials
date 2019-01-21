@[toc]
![image](https://ws3.sinaimg.cn/large/005JrW9Kgy1fze8fhhsldj30jh0kzdg7.jpg)

# ref

https://github.com/p4lang/switch/blob/master/p4src/README.md


ps： 这个工程采用的 是P_14 的语法


# 从宏观入手，整一个交换机

https://github.com/p4lang/switch/blob/master/p4src/switch.p4

`switch.p4` 定义了标准的 `ingress_metadata_t` 以及 `egress_metadata_t `

并且引入 诸多的 依赖 文件 ：

```c
#include "switch_config.p4"
#ifdef OPENFLOW_ENABLE
#include "openflow.p4"
#endif /* OPENFLOW_ENABLE */
#include "port.p4"
#include "l2.p4"
#include "l3.p4"
#include "ipv4.p4"
#include "ipv6.p4"
#include "tunnel.p4"
#include "acl.p4"
#include "nat.p4"
#include "multicast.p4"
#include "nexthop.p4"
#include "rewrite.p4"
#include "security.p4"
#include "fabric.p4"
#include "egress_filter.p4"
#include "mirror.p4"
#include "int_transit.p4"
#include "hashes.p4"
#include "meter.p4"
#include "sflow.p4"
#include "qos.p4"
```

然后就立刻定义了 两个 `control block`

 1. control ingress  
 2. control egress

分别是入口 和  出口 逻辑

# 交换机的入口逻辑

## 首先处理端口

###  ingress第一步process_ingress_port_mapping（）

来自： https://github.com/p4lang/switch/blob/master/p4src/port.p4

### 进入`port.p4`

ln ：  186 的 control process_ingress_port_mapping

	control process_ingress_port_mapping {
	    apply(ingress_port_mapping);
	    apply(ingress_port_properties);
	}

调用

ln ： 150  的action

	action set_ifindex(ifindex, port_type) {
	    modify_field(ingress_metadata.ifindex, ifindex); // 设定端口 id
	    modify_field(ingress_metadata.port_type, port_type); // 设定端口 的类型， 可能是access， 或者 trunk
	}

也会调用 ln ： 165 的 action `action set_ingress_port_properties` :



![在这里插入图片描述](https://img-blog.csdnimg.cn/20190121115145948.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3BhdWxrZzEy,size_16,color_FFFFFF,t_70)

其中 `acl_metadata` 这些数据 是在其他？？ 地方定义的，可以根据我的的业务要求增加删除。

 总之，端口属性通过 查询 2张 表：
1. ingress_port_mapping
2. ingress_port_properties
完成了
a. 端口索引的赋值，端口类型的赋值
b. 端口属性的赋值： acl，  qos， meter

## 接下来处理 头部，会有与vlan tag 相关的

### 调用是： process_validate_outer_header();

同样是在 port.p4 中

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190121134151152.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3BhdWxrZzEy,size_16,color_FFFFFF,t_70)

他使能了这张表：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190121134220586.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3BhdWxrZzEy,size_16,color_FFFFFF,t_70)

可以看到这个表 的 MA 导致的Action 有许多种，我们可以一个个看看，从最简单的 `set_valid_outer_unicast_packet_untagged`

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190121134737826.png)

可以知道，对于没有 tag 标记的报文 ，我们仅仅 做了 报文类型->UNICAST 单播的赋值， 以及 二层类型->以太网类型  的赋值 。

接下来是  `set_valid_outer_unicast_packet_single_tagged`， 这是针对仅仅有一个vlan tag 的情况，aka ： single tag
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190121134903967.png)

可以看到： 同样是设定了 报文 的类型为： 单播UNICAST， 同样设定了 二层类型 为以太网类型，然后有个pcp （对应  vlan 标记中  PRI 优先级）

## 接下来处理全局配置

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190121135912882.png)

尚未找到定义。

## 接下来处理vlan

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190121135943282.png)

### 调用的是 port 中的内容

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190121140343569.png)
apply 的表是如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190121140712149.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3BhdWxrZzEy,size_16,color_FFFFFF,t_70)
可以看到的是： 解析vlan tag 如果有命中的话 ， MA 对应的Action 是：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190121140909259.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3BhdWxrZzEy,size_16,color_FFFFFF,t_70)

可以看到， 和传统的交换机的 做的action 一项，无非是一些  inner vlan ID， out vlan ID， 生成树，学习使能位IPv4/6 的单/多/广播属性设置。这些设置的来源是控制面板下发的表项。


## 接下来处理 生成树

## 接下来处理 qos

## 接下来处理 IPSG

## 接下来处理 INT

## 接下来处理 sflow

## 接下来处理 tunnel

## 接下来处理 storm

##  还会处理的有： FABRIC port， MPLS， nat， meter， hash， openflow， traffic class，

# 交换机的出口逻辑 todo 0121

- [ ] todo reading https://github.com/p4lang/switch/blob/master/p4src/switch.p4

## 辅助： 一些数据结构的原始定义


### vlan_tag_t
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190121140454556.png)


## 辅助： 术语表

![image](https://wx1.sinaimg.cn/large/005JrW9Kgy1fze8nrbydvj30kw02xa9z.jpg)
