[TOC]

# ref

https://github.com/anonymity12/p4-guide/blob/master/demo2/README.md


# NOTE

## DEMO2  要做什么

和demo1 相同，不过就是加上了 一个 prefix 的match count

![](assets/p4_guide_demo2-3fbd94c5.png)


注意观察 demo 2 的 代码，我们认为，table 有一个 counters 的字段

![](assets/p4_guide_demo2-729f3081.png)

 我们可以通过cli 读出这个字段，命令是 ： `counter_read ipv4_da_lpm_stats 0`

 ## DEMO2 的实验过程
   
 按照readme 指导

 1. 编译

 命令是 :

 `p4c --target bmv2 --arch v1model demo2.p4_16.p4`

 ![](assets/p4_guide_demo2-61814e1a.png)

 2. 开启simple switch 并加载编译好的数据层面的json 文件（你可以理解json为配置，回调，合同，约定）

 命令是: `sudo simple_switch --log-console -i 0@veth2 -i 1@veth4 -i 2@veth6 -i 3@veth8 -i 4@veth10 -i 5@veth12 -i 6@veth14 -i 7@veth16 demo2.p4_16.json
`

 ![](assets/p4_guide_demo2-b72d9c18.png)

 3. 开启cli 进行 表项的下发

 命令是 :

 ```
 table_set_default ipv4_da_lpm my_drop
table_set_default mac_da my_drop
table_set_default send_frame my_drop

table_add ipv4_da_lpm set_l2ptr 10.1.0.1/32 => 58
table_add mac_da set_bd_dmac_intf 58 => 9 02:13:57:ab:cd:ef 2
table_add send_frame rewrite_mac 9 => 00:11:22:33:44:55

table_add ipv4_da_lpm set_l2ptr 10.1.0.200/32 => 81
table_add mac_da set_bd_dmac_intf 81 => 15 08:de:ad:be:ef:00 4
table_add send_frame rewrite_mac 15 => ca:fe:ba:be:d0:0d

 ```

 ![](assets/p4_guide_demo2-f066ae73.png)

 4. CLI 下读取counter

 命令是 ： `counter_read ipv4_da_lpm_stats 0`

 ![](assets/p4_guide_demo2-f1ead772.png)

5. 发送一个pkt 以期待计数器的变化。

**PS**: scapy 回话是在以太网接口用来发送pkt 的(我们此时使用的虚拟以太网接口:`veth`)，值得注意的是，任何在以太网接口上进行报文收发的行为，都需要root权限，所以请记得使用`sudo scapy`

 命令是 ：

 ```
 sudo scapy

fwd_pkt1=Ether() / IP(dst='10.1.0.1') / TCP(sport=5793, dport=80)
drop_pkt1=Ether() / IP(dst='10.1.0.34') / TCP(sport=5793, dport=80)

# Send packet at layer2, specifying interface
sendp(fwd_pkt1, iface="veth2")
sendp(drop_pkt1, iface="veth2")

fwd_pkt2=Ether() / IP(dst='10.1.0.1') / TCP(sport=5793, dport=80) / Raw('The quick brown fox jumped over the lazy dog.')
sendp(fwd_pkt2, iface="veth2")

 ```

 scapy 在`veth2`上发送一个pkt

 ![](assets/p4_guide_demo2-ff63e293.png)

 simple switch 这边 立刻就处理完成

![](assets/p4_guide_demo2-d211fe11.png)

  **ps**: 从[p4-guide/demo2/the_log_when_demo2](https://github.com/anonymity12/p4-guide/blob/master/demo2/the_log_when_demo2) 可以看到完整的scapy 发送报文 导致simple switch（虚拟交换机/交换机模拟器）产生的 报文处理log

6. 确认计数值确实发生了变化

可以与4 中的0 相比，这里， 你看到了pkt=1，bytes = 54.

![](assets/p4_guide_demo2-c963c653.png)

 发送一个因为目的ip 不存在而drop 的pkt，以及一个带有raw data 的pkt

 完整的变化结果：

 ![](assets/p4_guide_demo2-8a79d808.png)


**PS**: pkt 的size 和raw data 有关，raw data:

```
The quick brown fox jumped over the lazy dog.

```

 是45 个字符，也就是45B，所以看到第三次的pkt size 是 99（54 + 45），统计看到的bytes 是从54 （`54 + 99 = 153`）跃迁到 153, 那么我们可以认为 ：

> 一个不带任何data 的pkt ，其size 是 54。带了多少字节的data ，就会让这个pkt 增加多少字节的总长度。
