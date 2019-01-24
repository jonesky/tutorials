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

