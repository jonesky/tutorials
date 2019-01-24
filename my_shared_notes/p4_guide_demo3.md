p4_guide_demo3.md 


[TOC]

# 概要

demo3 实现 了 ECMP


# 笔记

## 数据定义阶段

为了实现 ecmp， 我们 定义自己的struct ： `fwd_metadata_t`


```
struct fwd_metadata_t {
    bit<16> hash1;
    bit<1>  nexthop_type;
    bit<10> ecmp_group_idx;
    bit<8>  ecmp_path_selector;
    bit<32> l2ptr;
    bit<24> out_bd;
}
```

然后用一个统一的数据结构： `metadata` 

包裹这个自定义的 `fwd_metadata_t`

```
struct metadata {
    fwd_metadata_t fwd_metadata;
}

```

## parser 阶段


## ingress 阶段


```C
control ingress(inout headers hdr,
                inout metadata meta,
                inout standard_metadata_t standard_metadata) {
    direct_counter(CounterType.packets) ipv4_da_lpm_stats;

//s1 hash 的MA 行为，ps： 这里的M match key 为空，所以是match everything 吧
    action compute_lkp_ipv4_hash() {
        hash(meta.fwd_metadata.hash1, HashAlgorithm.crc16,
             (bit<16>) 0, { hdr.ipv4.srcAddr,
                            hdr.ipv4.dstAddr,
                            hdr.ipv4.protocol },
             (bit<32>) 65536);
    }
    table compute_ipv4_hashes {
        // 无视key 
        actions = {
            compute_lkp_ipv4_hash;
        }
        default_action = compute_lkp_ipv4_hash;
    }

//s2 接下来根据 目标 ip 作为key， 设定一些数据流的 下一跳，可能是普通的l2 , 或许是ecmp 的下一跳
    // 下一跳 是 简单普通的 l2 jump
    action set_l2ptr_with_stat(bit<32> l2ptr) {
        // 开启计数器
        ipv4_da_lpm_stats.count();
        meta.fwd_metadata.nexthop_type = NEXTHOP_TYPE_L2PTR;
        meta.fwd_metadata.l2ptr = l2ptr;
    }
    // 下一跳 是 等价路由 后的jump
    action set_ecmp_group_idx(bit<10> ecmp_group_idx) {
        ipv4_da_lpm_stats.count();
        meta.fwd_metadata.nexthop_type = NEXTHOP_TYPE_ECMP_GROUP_IDX;
        meta.fwd_metadata.ecmp_group_idx = ecmp_group_idx;
    }
    action my_drop_with_stat() {
        ipv4_da_lpm_stats.count();
        mark_to_drop();
    }
    table ipv4_da_lpm {
        // ecmp 是根据  目标 ip 作为key 进行match 的。
        key = {
            hdr.ipv4.dstAddr: lpm;
        }
        actions = {
            set_l2ptr_with_stat;
            set_ecmp_group_idx;
            my_drop_with_stat;
        }
        default_action = my_drop_with_stat;
        counters = ipv4_da_lpm_stats;
    }
// s3 前一步设定了ecmp 的组（group） 现在可以在组内选择一个path的index ，path 选择的结果放在第一个参数 ecmp_path_selector 中
    action set_ecmp_path_idx(bit<8> num_paths) {
        hash(meta.fwd_metadata.ecmp_path_selector, HashAlgorithm.identity,
             (bit<16>) 0, { meta.fwd_metadata.hash1 }, (bit<32>)num_paths);
    }
    // 这里的set l2 ptr 其实是经过了 ecmp 优化的 L2 的下一跳
    action set_l2ptr(bit<32> l2ptr) {
        meta.fwd_metadata.nexthop_type = NEXTHOP_TYPE_L2PTR;
        meta.fwd_metadata.l2ptr = l2ptr;
    }
    // 如果在前一个match action 中，设定了ecmp group index ， 那么 就会在 这个表里进行match ecmp_group_idx
    table ecmp_group {
        key = {
            meta.fwd_metadata.ecmp_group_idx: exact;
        }
        actions = {
            set_ecmp_path_idx;
            set_l2ptr;
            @default_only NoAction;
        }
        default_action = NoAction();
        size = 32768;
    }
//s4 根据 组索引（group idx) 和 具体的path 索引（path_selector) ，现在进行具体的L2 下一跳的选择。
    table ecmp_path {
        key = {
            meta.fwd_metadata.ecmp_group_idx    : exact;
            meta.fwd_metadata.ecmp_path_selector: exact;
        }
        actions = {
            set_l2ptr;
            @default_only NoAction;
        }
        default_action = NoAction();
        size = 32768;
    }
// s5 设定bridge domain （觉得是 vlan ） 的属性（mac地址，出口端口），设定ttl - 1
    action set_bd_dmac_intf(bit<24> bd, bit<48> dmac, bit<9> intf) {
        meta.fwd_metadata.out_bd = bd;
        hdr.ethernet.dstAddr = dmac;
        standard_metadata.egress_spec = intf;
        hdr.ipv4.ttl = hdr.ipv4.ttl - 1;
    }
    table mac_da {
        key = {
            meta.fwd_metadata.l2ptr: exact;
        }
        actions = {
            set_bd_dmac_intf;
            my_drop;
        }
        default_action = my_drop;
    }

    apply {
        compute_ipv4_hashes.apply();
        ipv4_da_lpm.apply();
        // 首先看 下一跳的类型 ，不是明显的L 2 下一跳， 那么进行ecmp group 选择。
        if (meta.fwd_metadata.nexthop_type != NEXTHOP_TYPE_L2PTR) {
            ecmp_group.apply();
            // 如果还没有 明显 的  L2 下一跳，那么会使用 ecmp_path 这个table。
            if (meta.fwd_metadata.nexthop_type != NEXTHOP_TYPE_L2PTR) {
                ecmp_path.apply();
            }
        }
        // 最后生效 出口mac 变化的table
        mac_da.apply();
    }
}

```


## egress 阶段

```C
// 很明显， 出口仅仅需要把报文的src mac 进行变更。
// 
control egress(inout headers hdr,
               inout metadata meta,
               inout standard_metadata_t standard_metadata)
{
	// 把报文的src mac 进行变更
    action rewrite_mac(bit<48> smac) {
        hdr.ethernet.srcAddr = smac;
    }
    table send_frame {
        key = {
            meta.fwd_metadata.out_bd: exact;
        }
        actions = {
            rewrite_mac;
            my_drop;
        }
        default_action = my_drop;
    }

    apply {
        send_frame.apply();
    }
}

```

## deparser 阶段


```C
// 将头部 reassemble
control DeparserImpl(packet_out packet, in headers hdr) {
    apply {
        packet.emit(hdr.ethernet);
        packet.emit(hdr.ipv4);
    }
}


```


// 0124

# 实验


## 编译：

![image](https://wx2.sinaimg.cn/large/005JrW9Kly1fzhidg893rj30hp035jrf.jpg)


## 加载编译产物


![image](https://wx4.sinaimg.cn/large/005JrW9Kly1fzhj3ch9ogj30pu011glj.jpg)


## 开启控制层 与 数据层的通信

![image](https://ws1.sinaimg.cn/large/005JrW9Kly1fzhj50qw4qj30be029t8m.jpg)



## 控制层进行实际的下发

```

Control utility for runtime P4 table manipulation
RuntimeCmd: table_set_default compute_ipv4_hashes compute_lkp_ipv4_hash
Setting default action of compute_ipv4_hashes
action:              compute_lkp_ipv4_hash
runtime data:        
RuntimeCmd: table_add ipv4_da_lpm set_l2ptr_with_stat 10.1.0.1/32 => 58
Adding entry to lpm match table ipv4_da_lpm
match key:           LPM-0a:01:00:01/32
action:              set_l2ptr_with_stat
runtime data:        00:00:00:3a
Entry has been added with handle 0
RuntimeCmd: table_add mac_da set_bd_dmac_intf 58 => 9 02:13:57:ab:cd:ef 2
Adding entry to exact match table mac_da
match key:           EXACT-00:00:00:3a
action:              set_bd_dmac_intf
runtime data:        00:00:09	02:13:57:ab:cd:ef	00:02
Entry has been added with handle 0
RuntimeCmd: table_add send_frame rewrite_mac 9 => 00:11:22:33:44:55
Adding entry to exact match table send_frame
match key:           EXACT-00:00:09
action:              rewrite_mac
runtime data:        00:11:22:33:44:55
Entry has been added with handle 0
RuntimeCmd: table_add ipv4_da_lpm set_l2ptr_with_stat 10.1.0.200/32 => 81
Adding entry to lpm match table ipv4_da_lpm
match key:           LPM-0a:01:00:c8/32
action:              set_l2ptr_with_stat
runtime data:        00:00:00:51
Entry has been added with handle 1
RuntimeCmd: table_add mac_da set_bd_dmac_intf 81 => 15 08:de:ad:be:ef:00 1
Adding entry to exact match table mac_da
match key:           EXACT-00:00:00:51
action:              set_bd_dmac_intf
runtime data:        00:00:0f	08:de:ad:be:ef:00	00:01
Entry has been added with handle 1
RuntimeCmd: table_add send_frame rewrite_mac 15 => ca:fe:ba:be:d0:0d
Adding entry to exact match table send_frame
match key:           EXACT-00:00:0f
action:              rewrite_mac
runtime data:        ca:fe:ba:be:d0:0d
Entry has been added with handle 1
RuntimeCmd: table_add mac_da set_bd_dmac_intf 101 => 22 08:de:ad:be:ef:00 3
Adding entry to exact match table mac_da
match key:           EXACT-00:00:00:65
action:              set_bd_dmac_intf
runtime data:        00:00:16	08:de:ad:be:ef:00	00:03
Entry has been added with handle 2
RuntimeCmd: table_add send_frame rewrite_mac 22 => ca:fe:ba:be:d0:0d
Adding entry to exact match table send_frame
match key:           EXACT-00:00:16
action:              rewrite_mac
runtime data:        ca:fe:ba:be:d0:0d
Entry has been added with handle 2
RuntimeCmd: table_add ipv4_da_lpm set_ecmp_group_idx 11.1.0.1/32 => 67
Adding entry to lpm match table ipv4_da_lpm
match key:           LPM-0b:01:00:01/32
action:              set_ecmp_group_idx
runtime data:        00:43
Entry has been added with handle 2
RuntimeCmd: table_add ecmp_group set_ecmp_path_idx 67 => 3
Adding entry to exact match table ecmp_group
match key:           EXACT-00:43
action:              set_ecmp_path_idx
runtime data:        03
Entry has been added with handle 0
RuntimeCmd: table_add ecmp_path set_l2ptr 67 0 => 81
Adding entry to exact match table ecmp_path
match key:           EXACT-00:43	EXACT-00
action:              set_l2ptr
runtime data:        00:00:00:51
Entry has been added with handle 0
RuntimeCmd: table_add ecmp_path set_l2ptr 67 1 => 58
Adding entry to exact match table ecmp_path
match key:           EXACT-00:43	EXACT-01
action:              set_l2ptr
runtime data:        00:00:00:3a
Entry has been added with handle 1
RuntimeCmd: table_add ecmp_path set_l2ptr 67 2 => 101
Adding entry to exact match table ecmp_path
match key:           EXACT-00:43	EXACT-02
action:              set_l2ptr
runtime data:        00:00:00:65
Entry has been added with handle 2
RuntimeCmd: 
```

## 添加数据层面时候的 回应


```

myp4@myp4-VM:~/p4-guide/demo3$ sudo simple_switch --log-console -i 0@veth2 -i 1@veth4 -i 2@veth6 -i 3@veth8 -i 4@veth10 -i 5@veth12 -i 6@veth14 -i 7@veth16 demo3.p4_16.json
[sudo] password for myp4: 
Thrift port was not specified, will use 9090
Calling target program-options parser
[11:46:30.729] [bmv2] [D] [thread 8746] Set default default entry for table 'ingress.compute_ipv4_hashes': ingress.compute_lkp_ipv4_hash - 
[11:46:30.731] [bmv2] [D] [thread 8746] Set default default entry for table 'ingress.ipv4_da_lpm': ingress.my_drop_with_stat - 
[11:46:30.731] [bmv2] [D] [thread 8746] Set default default entry for table 'ingress.ecmp_group': NoAction - 
[11:46:30.731] [bmv2] [D] [thread 8746] Set default default entry for table 'ingress.ecmp_path': NoAction - 
[11:46:30.731] [bmv2] [D] [thread 8746] Set default default entry for table 'ingress.mac_da': my_drop - 
[11:46:30.731] [bmv2] [D] [thread 8746] Set default default entry for table 'egress.send_frame': my_drop - 
Adding interface veth2 as port 0
[11:46:30.731] [bmv2] [D] [thread 8746] Adding interface veth2 as port 0
Adding interface veth4 as port 1
[11:46:30.744] [bmv2] [D] [thread 8746] Adding interface veth4 as port 1
Adding interface veth6 as port 2
[11:46:30.744] [bmv2] [D] [thread 8746] Adding interface veth6 as port 2
Adding interface veth8 as port 3
[11:46:30.744] [bmv2] [D] [thread 8746] Adding interface veth8 as port 3
Adding interface veth10 as port 4
[11:46:30.745] [bmv2] [D] [thread 8746] Adding interface veth10 as port 4
Adding interface veth12 as port 5
[11:46:30.745] [bmv2] [D] [thread 8746] Adding interface veth12 as port 5
Adding interface veth14 as port 6
[11:46:30.745] [bmv2] [D] [thread 8746] Adding interface veth14 as port 6
Adding interface veth16 as port 7
[11:46:30.746] [bmv2] [D] [thread 8746] Adding interface veth16 as port 7
Thrift server was started
[11:47:01.550] [bmv2] [T] [thread 8761] bm_get_config
[15:19:52.837] [bmv2] [T] [thread 8761] bm_set_default_action
[15:19:52.837] [bmv2] [D] [thread 8761] Set default entry for table 'ingress.compute_ipv4_hashes': ingress.compute_lkp_ipv4_hash - 
[15:21:31.300] [bmv2] [T] [thread 8761] bm_table_add_entry
[15:21:31.315] [bmv2] [D] [thread 8761] Entry 0 added to table 'ingress.ipv4_da_lpm'
[15:21:31.315] [bmv2] [D] [thread 8761] Dumping entry 0
Match key:
* hdr.ipv4.dstAddr    : LPM       0a010001/32
Action entry: ingress.set_l2ptr_with_stat - 3a,

[15:23:51.524] [bmv2] [T] [thread 8761] bm_table_add_entry
[15:23:51.524] [bmv2] [D] [thread 8761] Entry 0 added to table 'ingress.mac_da'
[15:23:51.524] [bmv2] [D] [thread 8761] Dumping entry 0
Match key:
* meta.fwd_metadata.l2ptr: EXACT     0000003a
Action entry: ingress.set_bd_dmac_intf - 9,21357abcdef,2,

[15:24:10.786] [bmv2] [T] [thread 8761] bm_table_add_entry
[15:24:10.786] [bmv2] [D] [thread 8761] Entry 0 added to table 'egress.send_frame'
[15:24:10.786] [bmv2] [D] [thread 8761] Dumping entry 0
Match key:
* meta.fwd_metadata.out_bd: EXACT     000009
Action entry: egress.rewrite_mac - 1122334455,

[15:26:15.442] [bmv2] [T] [thread 8761] bm_table_add_entry
[15:26:15.456] [bmv2] [D] [thread 8761] Entry 1 added to table 'ingress.ipv4_da_lpm'
[15:26:15.456] [bmv2] [D] [thread 8761] Dumping entry 1
Match key:
* hdr.ipv4.dstAddr    : LPM       0a0100c8/32
Action entry: ingress.set_l2ptr_with_stat - 51,

[15:26:27.988] [bmv2] [T] [thread 8761] bm_table_add_entry
[15:26:27.988] [bmv2] [D] [thread 8761] Entry 1 added to table 'ingress.mac_da'
[15:26:27.988] [bmv2] [D] [thread 8761] Dumping entry 1
Match key:
* meta.fwd_metadata.l2ptr: EXACT     00000051
Action entry: ingress.set_bd_dmac_intf - f,8deadbeef00,1,

[15:26:48.984] [bmv2] [T] [thread 8761] bm_table_add_entry
[15:26:48.985] [bmv2] [D] [thread 8761] Entry 1 added to table 'egress.send_frame'
[15:26:48.985] [bmv2] [D] [thread 8761] Dumping entry 1
Match key:
* meta.fwd_metadata.out_bd: EXACT     00000f
Action entry: egress.rewrite_mac - cafebabed00d,

[15:30:10.297] [bmv2] [T] [thread 8761] bm_table_add_entry
[15:30:10.297] [bmv2] [D] [thread 8761] Entry 2 added to table 'ingress.mac_da'
[15:30:10.297] [bmv2] [D] [thread 8761] Dumping entry 2
Match key:
* meta.fwd_metadata.l2ptr: EXACT     00000065
Action entry: ingress.set_bd_dmac_intf - 16,8deadbeef00,3,

[15:32:28.577] [bmv2] [T] [thread 8761] bm_table_add_entry
[15:32:28.577] [bmv2] [D] [thread 8761] Entry 2 added to table 'egress.send_frame'
[15:32:28.577] [bmv2] [D] [thread 8761] Dumping entry 2
Match key:
* meta.fwd_metadata.out_bd: EXACT     000016
Action entry: egress.rewrite_mac - cafebabed00d,

[15:32:47.651] [bmv2] [T] [thread 8761] bm_table_add_entry
[15:32:47.651] [bmv2] [D] [thread 8761] Entry 2 added to table 'ingress.ipv4_da_lpm'
[15:32:47.651] [bmv2] [D] [thread 8761] Dumping entry 2
Match key:
* hdr.ipv4.dstAddr    : LPM       0b010001/32
Action entry: ingress.set_ecmp_group_idx - 43,

[15:32:55.763] [bmv2] [T] [thread 8761] bm_table_add_entry
[15:32:55.763] [bmv2] [D] [thread 8761] Entry 0 added to table 'ingress.ecmp_group'
[15:32:55.763] [bmv2] [D] [thread 8761] Dumping entry 0
Match key:
* meta.fwd_metadata.ecmp_group_idx: EXACT     0043
Action entry: ingress.set_ecmp_path_idx - 3,

[15:33:15.268] [bmv2] [T] [thread 8761] bm_table_add_entry
[15:33:15.268] [bmv2] [D] [thread 8761] Entry 0 added to table 'ingress.ecmp_path'
[15:33:15.268] [bmv2] [D] [thread 8761] Dumping entry 0
Match key:
* meta.fwd_metadata.ecmp_group_idx    : EXACT     0043
* meta.fwd_metadata.ecmp_path_selector: EXACT     00
Action entry: ingress.set_l2ptr - 51,

[15:34:22.488] [bmv2] [T] [thread 8761] bm_table_add_entry
[15:34:22.488] [bmv2] [D] [thread 8761] Entry 1 added to table 'ingress.ecmp_path'
[15:34:22.488] [bmv2] [D] [thread 8761] Dumping entry 1
Match key:
* meta.fwd_metadata.ecmp_group_idx    : EXACT     0043
* meta.fwd_metadata.ecmp_path_selector: EXACT     01
Action entry: ingress.set_l2ptr - 3a,

[15:34:27.864] [bmv2] [T] [thread 8761] bm_table_add_entry
[15:34:27.864] [bmv2] [D] [thread 8761] Entry 2 added to table 'ingress.ecmp_path'
[15:34:27.864] [bmv2] [D] [thread 8761] Dumping entry 2
Match key:
* meta.fwd_metadata.ecmp_group_idx    : EXACT     0043
* meta.fwd_metadata.ecmp_path_selector: EXACT     02
Action entry: ingress.set_l2ptr - 65,
```

## Scapy 发包

![image](https://ws1.sinaimg.cn/large/005JrW9Kly1fzhqsbn6auj30fu0b0t98.jpg)


## 看看我们交换机如何处理这些发送出来的包


```

[15:42:42.652] [bmv2] [D] [thread 8752] [0.0] [cxt 0] Processing packet received on port 0
[15:42:42.652] [bmv2] [D] [thread 8752] [0.0] [cxt 0] Parser 'parser': start
[15:42:42.652] [bmv2] [D] [thread 8752] [0.0] [cxt 0] Parser 'parser' entering state 'start'
[15:42:42.652] [bmv2] [D] [thread 8752] [0.0] [cxt 0] Extracting header 'ethernet'
[15:42:42.652] [bmv2] [D] [thread 8752] [0.0] [cxt 0] Parser state 'start': key is 0800
[15:42:42.652] [bmv2] [T] [thread 8752] [0.0] [cxt 0] Bytes parsed: 14
[15:42:42.652] [bmv2] [D] [thread 8752] [0.0] [cxt 0] Parser 'parser' entering state 'parse_ipv4'
[15:42:42.652] [bmv2] [D] [thread 8752] [0.0] [cxt 0] Extracting header 'ipv4'
[15:42:42.652] [bmv2] [D] [thread 8752] [0.0] [cxt 0] Parser state 'parse_ipv4' has no switch, going to default next state
[15:42:42.652] [bmv2] [T] [thread 8752] [0.0] [cxt 0] Bytes parsed: 34
[15:42:42.652] [bmv2] [D] [thread 8752] [0.0] [cxt 0] Verifying checksum 'cksum': true
[15:42:42.652] [bmv2] [D] [thread 8752] [0.0] [cxt 0] Verifying checksum 'cksum_0': true
[15:42:42.652] [bmv2] [D] [thread 8752] [0.0] [cxt 0] Parser 'parser': end
[15:42:42.652] [bmv2] [D] [thread 8752] [0.0] [cxt 0] Pipeline 'ingress': start
[15:42:42.652] [bmv2] [T] [thread 8752] [0.0] [cxt 0] Applying table 'ingress.compute_ipv4_hashes'
[15:42:42.652] [bmv2] [D] [thread 8752] [0.0] [cxt 0] Looking up key:

[15:42:42.652] [bmv2] [D] [thread 8752] [0.0] [cxt 0] Table 'ingress.compute_ipv4_hashes': miss
[15:42:42.652] [bmv2] [D] [thread 8752] [0.0] [cxt 0] Action entry is ingress.compute_lkp_ipv4_hash - 
[15:42:42.652] [bmv2] [T] [thread 8752] [0.0] [cxt 0] Action ingress.compute_lkp_ipv4_hash
[15:42:42.652] [bmv2] [T] [thread 8752] [0.0] [cxt 0] demo3.p4_16.p4(96) Primitive hash(meta.fwd_metadata.hash1, HashAlgorithm.crc16, ...
[15:42:42.652] [bmv2] [T] [thread 8752] [0.0] [cxt 0] Applying table 'ingress.ipv4_da_lpm'
[15:42:42.652] [bmv2] [D] [thread 8752] [0.0] [cxt 0] Looking up key:
* hdr.ipv4.dstAddr    : 0a0100c8

[15:42:42.652] [bmv2] [D] [thread 8752] [0.0] [cxt 0] Table 'ingress.ipv4_da_lpm': hit with handle 1
[15:42:42.652] [bmv2] [D] [thread 8752] [0.0] [cxt 0] Dumping entry 1
Match key:
* hdr.ipv4.dstAddr    : LPM       0a0100c8/32
Action entry: ingress.set_l2ptr_with_stat - 51,

[15:42:42.652] [bmv2] [D] [thread 8752] [0.0] [cxt 0] Action entry is ingress.set_l2ptr_with_stat - 51,
[15:42:42.652] [bmv2] [T] [thread 8752] [0.0] [cxt 0] Action ingress.set_l2ptr_with_stat
[15:42:42.652] [bmv2] [T] [thread 8752] [0.0] [cxt 0] demo3.p4_16.p4(42) Primitive 0; ...
[15:42:42.652] [bmv2] [T] [thread 8752] [0.0] [cxt 0] demo3.p4_16.p4(112) Primitive meta.fwd_metadata.l2ptr = l2ptr
[15:42:42.652] [bmv2] [T] [thread 8752] [0.0] [cxt 0] demo3.p4_16.p4(190) Condition "meta.fwd_metadata.nexthop_type != NEXTHOP_TYPE_L2PTR" (node_4) is false
[15:42:42.652] [bmv2] [T] [thread 8752] [0.0] [cxt 0] Applying table 'ingress.mac_da'
[15:42:42.652] [bmv2] [D] [thread 8752] [0.0] [cxt 0] Looking up key:
* meta.fwd_metadata.l2ptr: 00000051

[15:42:42.652] [bmv2] [D] [thread 8752] [0.0] [cxt 0] Table 'ingress.mac_da': hit with handle 1
[15:42:42.652] [bmv2] [D] [thread 8752] [0.0] [cxt 0] Dumping entry 1
Match key:
* meta.fwd_metadata.l2ptr: EXACT     00000051
Action entry: ingress.set_bd_dmac_intf - f,8deadbeef00,1,

[15:42:42.652] [bmv2] [D] [thread 8752] [0.0] [cxt 0] Action entry is ingress.set_bd_dmac_intf - f,8deadbeef00,1,
[15:42:42.652] [bmv2] [T] [thread 8752] [0.0] [cxt 0] Action ingress.set_bd_dmac_intf
[15:42:42.652] [bmv2] [T] [thread 8752] [0.0] [cxt 0] demo3.p4_16.p4(171) Primitive meta.fwd_metadata.out_bd = bd
[15:42:42.652] [bmv2] [T] [thread 8752] [0.0] [cxt 0] demo3.p4_16.p4(172) Primitive hdr.ethernet.dstAddr = dmac
[15:42:42.652] [bmv2] [T] [thread 8752] [0.0] [cxt 0] demo3.p4_16.p4(173) Primitive standard_metadata.egress_spec = intf
[15:42:42.652] [bmv2] [T] [thread 8752] [0.0] [cxt 0] demo3.p4_16.p4(174) Primitive hdr.ipv4.ttl = hdr.ipv4.ttl - 1
[15:42:42.652] [bmv2] [D] [thread 8752] [0.0] [cxt 0] Pipeline 'ingress': end
[15:42:42.652] [bmv2] [D] [thread 8752] [0.0] [cxt 0] Egress port is 1
[15:42:42.652] [bmv2] [D] [thread 8754] [0.0] [cxt 0] Pipeline 'egress': start
[15:42:42.652] [bmv2] [T] [thread 8754] [0.0] [cxt 0] Applying table 'egress.send_frame'
[15:42:42.652] [bmv2] [D] [thread 8754] [0.0] [cxt 0] Looking up key:
* meta.fwd_metadata.out_bd: 00000f

[15:42:42.652] [bmv2] [D] [thread 8754] [0.0] [cxt 0] Table 'egress.send_frame': hit with handle 1
[15:42:42.652] [bmv2] [D] [thread 8754] [0.0] [cxt 0] Dumping entry 1
Match key:
* meta.fwd_metadata.out_bd: EXACT     00000f
Action entry: egress.rewrite_mac - cafebabed00d,

[15:42:42.652] [bmv2] [D] [thread 8754] [0.0] [cxt 0] Action entry is egress.rewrite_mac - cafebabed00d,
[15:42:42.652] [bmv2] [T] [thread 8754] [0.0] [cxt 0] Action egress.rewrite_mac
[15:42:42.652] [bmv2] [T] [thread 8754] [0.0] [cxt 0] demo3.p4_16.p4(205) Primitive hdr.ethernet.srcAddr = smac
[15:42:42.652] [bmv2] [D] [thread 8754] [0.0] [cxt 0] Pipeline 'egress': end
[15:42:42.652] [bmv2] [D] [thread 8754] [0.0] [cxt 0] Deparser 'deparser': start
[15:42:42.652] [bmv2] [D] [thread 8754] [0.0] [cxt 0] Updating checksum 'cksum'
[15:42:42.652] [bmv2] [D] [thread 8754] [0.0] [cxt 0] Updating checksum 'cksum_0'
[15:42:42.652] [bmv2] [D] [thread 8754] [0.0] [cxt 0] Deparsing header 'ethernet'
[15:42:42.652] [bmv2] [D] [thread 8754] [0.0] [cxt 0] Deparsing header 'ipv4'
[15:42:42.652] [bmv2] [D] [thread 8754] [0.0] [cxt 0] Deparser 'deparser': end
[15:42:42.652] [bmv2] [D] [thread 8757] [0.0] [cxt 0] Transmitting packet of size 99 out of port 1
[15:42:49.386] [bmv2] [D] [thread 8752] [1.0] [cxt 0] Processing packet received on port 0
[15:42:49.386] [bmv2] [D] [thread 8752] [1.0] [cxt 0] Parser 'parser': start
[15:42:49.386] [bmv2] [D] [thread 8752] [1.0] [cxt 0] Parser 'parser' entering state 'start'
[15:42:49.386] [bmv2] [D] [thread 8752] [1.0] [cxt 0] Extracting header 'ethernet'
[15:42:49.386] [bmv2] [D] [thread 8752] [1.0] [cxt 0] Parser state 'start': key is 0800
[15:42:49.386] [bmv2] [T] [thread 8752] [1.0] [cxt 0] Bytes parsed: 14
[15:42:49.386] [bmv2] [D] [thread 8752] [1.0] [cxt 0] Parser 'parser' entering state 'parse_ipv4'
[15:42:49.386] [bmv2] [D] [thread 8752] [1.0] [cxt 0] Extracting header 'ipv4'
[15:42:49.386] [bmv2] [D] [thread 8752] [1.0] [cxt 0] Parser state 'parse_ipv4' has no switch, going to default next state
[15:42:49.386] [bmv2] [T] [thread 8752] [1.0] [cxt 0] Bytes parsed: 34
[15:42:49.386] [bmv2] [D] [thread 8752] [1.0] [cxt 0] Verifying checksum 'cksum': true
[15:42:49.386] [bmv2] [D] [thread 8752] [1.0] [cxt 0] Verifying checksum 'cksum_0': true
[15:42:49.386] [bmv2] [D] [thread 8752] [1.0] [cxt 0] Parser 'parser': end
[15:42:49.386] [bmv2] [D] [thread 8752] [1.0] [cxt 0] Pipeline 'ingress': start
[15:42:49.386] [bmv2] [T] [thread 8752] [1.0] [cxt 0] Applying table 'ingress.compute_ipv4_hashes'
[15:42:49.386] [bmv2] [D] [thread 8752] [1.0] [cxt 0] Looking up key:

[15:42:49.386] [bmv2] [D] [thread 8752] [1.0] [cxt 0] Table 'ingress.compute_ipv4_hashes': miss
[15:42:49.386] [bmv2] [D] [thread 8752] [1.0] [cxt 0] Action entry is ingress.compute_lkp_ipv4_hash - 
[15:42:49.386] [bmv2] [T] [thread 8752] [1.0] [cxt 0] Action ingress.compute_lkp_ipv4_hash
[15:42:49.386] [bmv2] [T] [thread 8752] [1.0] [cxt 0] demo3.p4_16.p4(96) Primitive hash(meta.fwd_metadata.hash1, HashAlgorithm.crc16, ...
[15:42:49.386] [bmv2] [T] [thread 8752] [1.0] [cxt 0] Applying table 'ingress.ipv4_da_lpm'
[15:42:49.386] [bmv2] [D] [thread 8752] [1.0] [cxt 0] Looking up key:
* hdr.ipv4.dstAddr    : 0a0100c8

[15:42:49.386] [bmv2] [D] [thread 8752] [1.0] [cxt 0] Table 'ingress.ipv4_da_lpm': hit with handle 1
[15:42:49.386] [bmv2] [D] [thread 8752] [1.0] [cxt 0] Dumping entry 1
Match key:
* hdr.ipv4.dstAddr    : LPM       0a0100c8/32
Action entry: ingress.set_l2ptr_with_stat - 51,

[15:42:49.386] [bmv2] [D] [thread 8752] [1.0] [cxt 0] Action entry is ingress.set_l2ptr_with_stat - 51,
[15:42:49.386] [bmv2] [T] [thread 8752] [1.0] [cxt 0] Action ingress.set_l2ptr_with_stat
[15:42:49.386] [bmv2] [T] [thread 8752] [1.0] [cxt 0] demo3.p4_16.p4(42) Primitive 0; ...
[15:42:49.386] [bmv2] [T] [thread 8752] [1.0] [cxt 0] demo3.p4_16.p4(112) Primitive meta.fwd_metadata.l2ptr = l2ptr
[15:42:49.386] [bmv2] [T] [thread 8752] [1.0] [cxt 0] demo3.p4_16.p4(190) Condition "meta.fwd_metadata.nexthop_type != NEXTHOP_TYPE_L2PTR" (node_4) is false
[15:42:49.386] [bmv2] [T] [thread 8752] [1.0] [cxt 0] Applying table 'ingress.mac_da'
[15:42:49.386] [bmv2] [D] [thread 8752] [1.0] [cxt 0] Looking up key:
* meta.fwd_metadata.l2ptr: 00000051

[15:42:49.386] [bmv2] [D] [thread 8752] [1.0] [cxt 0] Table 'ingress.mac_da': hit with handle 1
[15:42:49.386] [bmv2] [D] [thread 8752] [1.0] [cxt 0] Dumping entry 1
Match key:
* meta.fwd_metadata.l2ptr: EXACT     00000051
Action entry: ingress.set_bd_dmac_intf - f,8deadbeef00,1,

[15:42:49.386] [bmv2] [D] [thread 8752] [1.0] [cxt 0] Action entry is ingress.set_bd_dmac_intf - f,8deadbeef00,1,
[15:42:49.386] [bmv2] [T] [thread 8752] [1.0] [cxt 0] Action ingress.set_bd_dmac_intf
[15:42:49.386] [bmv2] [T] [thread 8752] [1.0] [cxt 0] demo3.p4_16.p4(171) Primitive meta.fwd_metadata.out_bd = bd
[15:42:49.386] [bmv2] [T] [thread 8752] [1.0] [cxt 0] demo3.p4_16.p4(172) Primitive hdr.ethernet.dstAddr = dmac
[15:42:49.386] [bmv2] [T] [thread 8752] [1.0] [cxt 0] demo3.p4_16.p4(173) Primitive standard_metadata.egress_spec = intf
[15:42:49.386] [bmv2] [T] [thread 8752] [1.0] [cxt 0] demo3.p4_16.p4(174) Primitive hdr.ipv4.ttl = hdr.ipv4.ttl - 1
[15:42:49.386] [bmv2] [D] [thread 8752] [1.0] [cxt 0] Pipeline 'ingress': end
[15:42:49.386] [bmv2] [D] [thread 8752] [1.0] [cxt 0] Egress port is 1
[15:42:49.386] [bmv2] [D] [thread 8754] [1.0] [cxt 0] Pipeline 'egress': start
[15:42:49.386] [bmv2] [T] [thread 8754] [1.0] [cxt 0] Applying table 'egress.send_frame'
[15:42:49.386] [bmv2] [D] [thread 8754] [1.0] [cxt 0] Looking up key:
* meta.fwd_metadata.out_bd: 00000f

[15:42:49.386] [bmv2] [D] [thread 8754] [1.0] [cxt 0] Table 'egress.send_frame': hit with handle 1
[15:42:49.386] [bmv2] [D] [thread 8754] [1.0] [cxt 0] Dumping entry 1
Match key:
* meta.fwd_metadata.out_bd: EXACT     00000f
Action entry: egress.rewrite_mac - cafebabed00d,

[15:42:49.386] [bmv2] [D] [thread 8754] [1.0] [cxt 0] Action entry is egress.rewrite_mac - cafebabed00d,
[15:42:49.386] [bmv2] [T] [thread 8754] [1.0] [cxt 0] Action egress.rewrite_mac
[15:42:49.386] [bmv2] [T] [thread 8754] [1.0] [cxt 0] demo3.p4_16.p4(205) Primitive hdr.ethernet.srcAddr = smac
[15:42:49.386] [bmv2] [D] [thread 8754] [1.0] [cxt 0] Pipeline 'egress': end
[15:42:49.386] [bmv2] [D] [thread 8754] [1.0] [cxt 0] Deparser 'deparser': start
[15:42:49.386] [bmv2] [D] [thread 8754] [1.0] [cxt 0] Updating checksum 'cksum'
[15:42:49.386] [bmv2] [D] [thread 8754] [1.0] [cxt 0] Updating checksum 'cksum_0'
[15:42:49.386] [bmv2] [D] [thread 8754] [1.0] [cxt 0] Deparsing header 'ethernet'
[15:42:49.386] [bmv2] [D] [thread 8754] [1.0] [cxt 0] Deparsing header 'ipv4'
[15:42:49.386] [bmv2] [D] [thread 8754] [1.0] [cxt 0] Deparser 'deparser': end
[15:42:49.386] [bmv2] [D] [thread 8757] [1.0] [cxt 0] Transmitting packet of size 99 out of port 1














[15:44:25.452] [bmv2] [D] [thread 8752] [2.0] [cxt 0] Processing packet received on port 0
[15:44:25.452] [bmv2] [D] [thread 8752] [2.0] [cxt 0] Parser 'parser': start
[15:44:25.452] [bmv2] [D] [thread 8752] [2.0] [cxt 0] Parser 'parser' entering state 'start'
[15:44:25.452] [bmv2] [D] [thread 8752] [2.0] [cxt 0] Extracting header 'ethernet'
[15:44:25.452] [bmv2] [D] [thread 8752] [2.0] [cxt 0] Parser state 'start': key is 0800
[15:44:25.452] [bmv2] [T] [thread 8752] [2.0] [cxt 0] Bytes parsed: 14
[15:44:25.452] [bmv2] [D] [thread 8752] [2.0] [cxt 0] Parser 'parser' entering state 'parse_ipv4'
[15:44:25.452] [bmv2] [D] [thread 8752] [2.0] [cxt 0] Extracting header 'ipv4'
[15:44:25.452] [bmv2] [D] [thread 8752] [2.0] [cxt 0] Parser state 'parse_ipv4' has no switch, going to default next state
[15:44:25.452] [bmv2] [T] [thread 8752] [2.0] [cxt 0] Bytes parsed: 34
[15:44:25.452] [bmv2] [D] [thread 8752] [2.0] [cxt 0] Verifying checksum 'cksum': true
[15:44:25.452] [bmv2] [D] [thread 8752] [2.0] [cxt 0] Verifying checksum 'cksum_0': true
[15:44:25.452] [bmv2] [D] [thread 8752] [2.0] [cxt 0] Parser 'parser': end
[15:44:25.452] [bmv2] [D] [thread 8752] [2.0] [cxt 0] Pipeline 'ingress': start
[15:44:25.452] [bmv2] [T] [thread 8752] [2.0] [cxt 0] Applying table 'ingress.compute_ipv4_hashes'
[15:44:25.452] [bmv2] [D] [thread 8752] [2.0] [cxt 0] Looking up key:

[15:44:25.452] [bmv2] [D] [thread 8752] [2.0] [cxt 0] Table 'ingress.compute_ipv4_hashes': miss
[15:44:25.452] [bmv2] [D] [thread 8752] [2.0] [cxt 0] Action entry is ingress.compute_lkp_ipv4_hash - 
[15:44:25.452] [bmv2] [T] [thread 8752] [2.0] [cxt 0] Action ingress.compute_lkp_ipv4_hash
[15:44:25.452] [bmv2] [T] [thread 8752] [2.0] [cxt 0] demo3.p4_16.p4(96) Primitive hash(meta.fwd_metadata.hash1, HashAlgorithm.crc16, ...
[15:44:25.452] [bmv2] [T] [thread 8752] [2.0] [cxt 0] Applying table 'ingress.ipv4_da_lpm'
[15:44:25.452] [bmv2] [D] [thread 8752] [2.0] [cxt 0] Looking up key:
* hdr.ipv4.dstAddr    : 0a0100c8

[15:44:25.452] [bmv2] [D] [thread 8752] [2.0] [cxt 0] Table 'ingress.ipv4_da_lpm': hit with handle 1
[15:44:25.452] [bmv2] [D] [thread 8752] [2.0] [cxt 0] Dumping entry 1
Match key:
* hdr.ipv4.dstAddr    : LPM       0a0100c8/32
Action entry: ingress.set_l2ptr_with_stat - 51,

[15:44:25.452] [bmv2] [D] [thread 8752] [2.0] [cxt 0] Action entry is ingress.set_l2ptr_with_stat - 51,
[15:44:25.452] [bmv2] [T] [thread 8752] [2.0] [cxt 0] Action ingress.set_l2ptr_with_stat
[15:44:25.452] [bmv2] [T] [thread 8752] [2.0] [cxt 0] demo3.p4_16.p4(42) Primitive 0; ...
[15:44:25.452] [bmv2] [T] [thread 8752] [2.0] [cxt 0] demo3.p4_16.p4(112) Primitive meta.fwd_metadata.l2ptr = l2ptr
[15:44:25.452] [bmv2] [T] [thread 8752] [2.0] [cxt 0] demo3.p4_16.p4(190) Condition "meta.fwd_metadata.nexthop_type != NEXTHOP_TYPE_L2PTR" (node_4) is false
[15:44:25.452] [bmv2] [T] [thread 8752] [2.0] [cxt 0] Applying table 'ingress.mac_da'
[15:44:25.452] [bmv2] [D] [thread 8752] [2.0] [cxt 0] Looking up key:
* meta.fwd_metadata.l2ptr: 00000051

[15:44:25.452] [bmv2] [D] [thread 8752] [2.0] [cxt 0] Table 'ingress.mac_da': hit with handle 1
[15:44:25.452] [bmv2] [D] [thread 8752] [2.0] [cxt 0] Dumping entry 1
Match key:
* meta.fwd_metadata.l2ptr: EXACT     00000051
Action entry: ingress.set_bd_dmac_intf - f,8deadbeef00,1,

[15:44:25.452] [bmv2] [D] [thread 8752] [2.0] [cxt 0] Action entry is ingress.set_bd_dmac_intf - f,8deadbeef00,1,
[15:44:25.452] [bmv2] [T] [thread 8752] [2.0] [cxt 0] Action ingress.set_bd_dmac_intf
[15:44:25.452] [bmv2] [T] [thread 8752] [2.0] [cxt 0] demo3.p4_16.p4(171) Primitive meta.fwd_metadata.out_bd = bd
[15:44:25.452] [bmv2] [T] [thread 8752] [2.0] [cxt 0] demo3.p4_16.p4(172) Primitive hdr.ethernet.dstAddr = dmac
[15:44:25.452] [bmv2] [T] [thread 8752] [2.0] [cxt 0] demo3.p4_16.p4(173) Primitive standard_metadata.egress_spec = intf
[15:44:25.452] [bmv2] [T] [thread 8752] [2.0] [cxt 0] demo3.p4_16.p4(174) Primitive hdr.ipv4.ttl = hdr.ipv4.ttl - 1
[15:44:25.452] [bmv2] [D] [thread 8752] [2.0] [cxt 0] Pipeline 'ingress': end
[15:44:25.452] [bmv2] [D] [thread 8752] [2.0] [cxt 0] Egress port is 1
[15:44:25.452] [bmv2] [D] [thread 8754] [2.0] [cxt 0] Pipeline 'egress': start
[15:44:25.452] [bmv2] [T] [thread 8754] [2.0] [cxt 0] Applying table 'egress.send_frame'
[15:44:25.452] [bmv2] [D] [thread 8754] [2.0] [cxt 0] Looking up key:
* meta.fwd_metadata.out_bd: 00000f

[15:44:25.452] [bmv2] [D] [thread 8754] [2.0] [cxt 0] Table 'egress.send_frame': hit with handle 1
[15:44:25.452] [bmv2] [D] [thread 8754] [2.0] [cxt 0] Dumping entry 1
Match key:
* meta.fwd_metadata.out_bd: EXACT     00000f
Action entry: egress.rewrite_mac - cafebabed00d,

[15:44:25.452] [bmv2] [D] [thread 8754] [2.0] [cxt 0] Action entry is egress.rewrite_mac - cafebabed00d,
[15:44:25.452] [bmv2] [T] [thread 8754] [2.0] [cxt 0] Action egress.rewrite_mac
[15:44:25.452] [bmv2] [T] [thread 8754] [2.0] [cxt 0] demo3.p4_16.p4(205) Primitive hdr.ethernet.srcAddr = smac
[15:44:25.452] [bmv2] [D] [thread 8754] [2.0] [cxt 0] Pipeline 'egress': end
[15:44:25.452] [bmv2] [D] [thread 8754] [2.0] [cxt 0] Deparser 'deparser': start
[15:44:25.452] [bmv2] [D] [thread 8754] [2.0] [cxt 0] Updating checksum 'cksum'
[15:44:25.452] [bmv2] [D] [thread 8754] [2.0] [cxt 0] Updating checksum 'cksum_0'
[15:44:25.452] [bmv2] [D] [thread 8754] [2.0] [cxt 0] Deparsing header 'ethernet'
[15:44:25.452] [bmv2] [D] [thread 8754] [2.0] [cxt 0] Deparsing header 'ipv4'
[15:44:25.452] [bmv2] [D] [thread 8754] [2.0] [cxt 0] Deparser 'deparser': end
[15:44:25.452] [bmv2] [D] [thread 8757] [2.0] [cxt 0] Transmitting packet of size 99 out of port 1
```

# 辅助

## 选择ecmp 的 path 的时候, ecmp 的选路的具体实现是hash 出 0,1,2：

![image](https://wx2.sinaimg.cn/large/005JrW9Kly1fzhos6kljzj30dc02jglh.jpg)

可以看到 三个 : 67 X  的配对

他们对应的就是以下的两个key 

![image](https://ws4.sinaimg.cn/large/005JrW9Kly1fzhotzkpqvj30bt02d744.jpg)



意思就是， 经过hash 后，会随机得到三个path 的选择值：0，1，2

作为一个动态的key ，在我们静态add 到table 中的entry 中进行选择，

从而得到一个 ecmp 的路径。




