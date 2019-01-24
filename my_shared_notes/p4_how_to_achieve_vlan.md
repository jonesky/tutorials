p4_how_to_achieve_vlan.md


# 尝试的示例代码


```C
/* -*- P4_16 -*- */
#include <core.p4>
#include <v1model.p4>

const bit<16> TYPE_IPV4 = 0x800;
// 为了支持vlan 解析
const bit<16> TYPE_VLAN = 0X8100;

/*************************************************************************
*********************** H E A D E R S  ***********************************
*************************************************************************/

typedef bit<9>  egressSpec_t;
typedef bit<48> macAddr_t;
typedef bit<16> mcast_group_t;
typedef bit<32> ip4Addr_t;

header ethernet_t {
    macAddr_t dstAddr;
    macAddr_t srcAddr;
    bit<16>   etherType;
}
// 在这里添加新的 头部, 用以 支持  vlan
header vlan_t {
	bit<3> pcp;// priority
	bit<1> cfi; // drop or not
	bit<12> vid; // which vlan this packet belongs to
	bit<16> etherType; // why it is here?
}

header ipv4_t {
    bit<4>    version;
    bit<4>    ihl;
    bit<8>    diffserv;
    bit<16>   totalLen;
    bit<16>   identification;
    bit<3>    flags;
    bit<13>   fragOffset;
    bit<8>    ttl;
    bit<8>    protocol;
    bit<16>   hdrChecksum;
    ip4Addr_t srcAddr;
    ip4Addr_t dstAddr;
}

struct metadata {

    bit<12>    changedVlanTag;
}

struct headers {
    ethernet_t   ethernet;
    vlan_t       vlan; // now we assume only one vlan tag here.
    ipv4_t       ipv4;
}

/*************************************************************************
*********************** P A R S E R  ***********************************
*************************************************************************/

parser MyParser(packet_in packet,
                out headers hdr,
                inout metadata meta,
                inout standard_metadata_t standard_metadata) {

    state start {
        transition parse_ethernet;
    }
    state parse_ethernet {
		packet.extract(hdr.ethernet);
		transition select(hdr.ethernet.etherType) {
            TYPE_IPV4: parse_ipv4;
		    TYPE_VLAN: parse_vlan;
		    default: accept;
		}
    }
    state parse_ipv4{
    	packet.extract(hdr.ipv4);
    	transition accept;
    }

    state parse_vlan{
    	packet.extract(hdr.vlan);
    	transition select(hdr.vlan.etherType) {
            TYPE_IPV4: parse_ipv4;
            default: accept;
        } //tdo vlan 中解析ipv4 就像解析 ipv4 报文一样,开始 解析vlan 报文; 即将进入ingress control
    }
}


/*************************************************************************
************   C H E C K S U M    V E R I F I C A T I O N   *************
*************************************************************************/

control MyVerifyChecksum(inout headers hdr, inout metadata meta) {   
    apply {  }
}


/*************************************************************************
**************  I N G R E S S   P R O C E S S I N G   *******************
*************************************************************************/

control MyIngress(inout headers hdr,
                  inout metadata meta,
                  inout standard_metadata_t standard_metadata) {
    // 一般的表 都会使用 这个 action
    action drop() {
        mark_to_drop();
    }

    // 假定需要处理vlan ,那么有这样的行为
    action vlan_forward(egressSpec_t portOrPorts, bit<12> changeTag) {
        //egressSpec_t 可能是一个单播的口, 也可能是多个 口
        standard_metadata.egress_spec = portOrPorts;
        // 多种tag 行为：
        // 1. untag 则把 tag 改成 0；
        // 2. 更改tag 则把tag 改成不是0 的；
        standard_metadata.changedVlanTag = changeTag;
    }

    table vlan_exact {
        key = {
        	hdr.vlan.vid: exact; // 我们要精确匹配每个vlan id
        	standard_metadata.ingress_port: exact; // 每个报文的入端口也是需要检查的， 比如入口是access口， 但是此入口进入的pkt 是带有 vlan tag 的，这不正常（PC 不会发送一个带有tag 的报文），于是匹配行为：drop
        }
        actions = {
            vlan_forward;
            drop;
            NoAction;
        }
        size = 4096; // 因为支持最多4094 个vlan??
        default_action = NoAction();
    }

    apply {
        if (hdr.vlan.isValid()) {
            vlan_exact.apply();
        }
    }

}

```
预期的vlan table 可能是这样的：

![image](https://ws1.sinaimg.cn/large/005JrW9Kgy1fzgkru9bwfj30ra05y74t.jpg)
```c

/*************************************************************************
****************  E G R E S S   P R O C E S S I N G   *******************
*************************************************************************/

control MyEgress(inout headers hdr,
                 inout metadata meta,
                 inout standard_metadata_t standard_metadata) {
                    // todo 0124 egress not-allow-range drop 
    apply {
        if (standard_metadata.changedVlanTag == 0) {
            // 需要untag
            hdr.vlan.setInvalid();// 可能不准确是这个方法，总之是类似使之失效的API
        }
        // 不需要untag 的情况，可能tag 有改变或者没有改变，直接转发
    }
}

/*************************************************************************
*************   C H E C K S U M    C O M P U T A T I O N   **************
*************************************************************************/

control MyComputeChecksum(inout headers hdr, inout metadata meta) {
     apply {

	update_checksum(
	    hdr.ipv4.isValid(),
            { hdr.ipv4.version,
	      hdr.ipv4.ihl,
              hdr.ipv4.diffserv,
              hdr.ipv4.totalLen,
              hdr.ipv4.identification,
              hdr.ipv4.flags,
              hdr.ipv4.fragOffset,
              hdr.ipv4.ttl,
              hdr.ipv4.protocol,
              hdr.ipv4.srcAddr,
              hdr.ipv4.dstAddr },
            hdr.ipv4.hdrChecksum,
            HashAlgorithm.csum16);

    }
}


/*************************************************************************
***********************  D E P A R S E R  *******************************
*************************************************************************/

control MyDeparser(packet_out packet, in headers hdr) {
    apply {
		packet.emit(hdr.ethernet);
        // 在有效的时候，发送 vlan hdr， 无效的时候，实际上do nothing
		packet.emit(hdr.vlan); // 如果 emit（hdr) 时， hdr 是invalid 的，此时就是个no-op 行为； 见 spec 15.1
		packet.emit(hdr.ipv4);
    }
}

/*************************************************************************
***********************  S W I T C H  *******************************
*************************************************************************/

V1Switch(
MyParser(),
MyVerifyChecksum(),
MyIngress(),
MyEgress(),
MyComputeChecksum(),
MyDeparser()
) main;
```
