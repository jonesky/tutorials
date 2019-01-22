
[TOC]

# c12 pkt parser



parser就是一个状态机，它自身在多种状态之间切换， 在不同的状态中，完成：


- 次态的select/ transition
- 对条件的verify
- 提取一些bits， 形成 一个 header 

![image](https://wx2.sinaimg.cn/large/005JrW9Kgy1fzedzxndovj30bc0buwew.jpg)

## 12.3. The Parser abstract machine

```Java

ParserModel {
  error parseError;
  onPacketArrival(packet p) {
    ParserModel.parseError = error.NoError;
    goto start;
  }
}
```

架构会决定parser 的 accept 状态 和  reject 状态的 实际行为。

比如

a。 有的架构 直接 drop 掉 reject 的包

b。 有的架构 会 送 到下一个block ，而且 附带 错误码



## 12.4. Parser states

parser  包含以下这些内容：
  1. 变量的声明
  2. 赋值语句
  3. 方法调用，包括：（校验）函数调用。
               计算checksum  的函数，
               状态迁移    


架构要给自己的parser    一些限制， 比如本地变量的数量，操作限制，比如乘法    

## 12.5. Transition statements


## 12.6. Select expressions


## 12.7. verify

verify 仅仅能在 parser 里进行invoke    

`extern void verify(in bool condition, in error err);
`


## 12.8. Data extraction

`packt_in` 代表进来的网络报文。在 `<core.p4>` 中定义

其行为如下：

```java
extern packet_in {
  void extract<T>(out T headerLvalue);
  void extract<T>(out T variableSizeHeader, in bit<32> varFieldSizeBits);
  T lookahead<T>();
  bit<32> length(); // This method may be unavailable in some architectures
  void advance(bit<32> bits);
}
```

其数据模型如下：

![image](https://ws3.sinaimg.cn/mw690/005JrW9Kgy1fzef812vwmj30io0a80ta.jpg)

---

使用`extract`需要注意 的是 `cut-through packet processing`: 在不知道报文总长度的时候，就开始处理报文首部。 此时进行 提取 ， 很有可能发生 failure

### 12.8.1. Fixed width extraction 固定长度提取


`void extract<T>(out T headerLeftValue);`

这里的入参 `headerLeftValue` 一直被原文强调为 左值， 是因为 这个参数是留待被赋值的

`extract` 的行为是： 从 packet_in 对象 中提取 出来 和 `headerLeftValue` 相同长度（比如 16bits） 的二进制数据
（比如： 1001010010101000）放置到  `headerLeftValue` 中。

从 extract 的实现来看， 内部维持的一个指针，也会因此 （因为：extract） 移动：

```C
void packet_in.extract<T>(out T headerLValue) {
  bitsToExtract = sizeofInBits(headerLValue);
  lastBitNeeded = this.nextBitIndex + bitsToExtract;
  ParserModel.verify(this.lengthInBits >= lastBitNeeded, error.PacketTooShort);
  headerLValue = this.data.extractBits(this.nextBitIndex, bitsToExtract);
  headerLValue.valid$ = true;
  if headerLValue.isNext$ {
    verify(headerLValue.nextIndex$ < headerLValue.size, error.StackOutOfBounds);
    headerLValue.nextIndex$ = headerLValue.nextIndex$ + 1;
  }
  // HERE !! 你看到指针移动了
  this.nextBitIndex += bitsToExtract;
}

```


### 12.8.2. Variable width extraction 变长长度提取

`void extract<T>(out T headerLvalue, in bit<32> variableFieldSize);`


### 12.8.3 lookahead 环视

这个和 `extract` 非常相似，除了它不会将内部指针移位，aka：仅仅就是看看，而不提取

![image](https://wx2.sinaimg.cn/large/005JrW9Kgy1fzefgxw6hpj30dl00tdfq.jpg)


### 12.8.4. Skipping bits

跳过 一些 bit ，aka： 不把它赋值给 某些 header



##  12.9. Header stacks

头栈

这个栈就是个数组， 此数组包含两个指针： next ， last

意思分别是： 下一个，上一个

我的理解其实是： 下一个 = 这一个， 上一个 = 就是上一个。

如下的代码使得我们可以根据前一个 mpls 的header 的bos 值 来决定是否需要继续处理 mpls 标签。（ps： bos 告诉我们：后面还有mpls 标签哦，你应该继续parse mpls 的内容）

```C
struct Pkthdr {
  Ethernet_h ethernet;
  Mpls_h[3] mpls;
  // other headers omitted
}
parser P(packet_in b, out Pkthdr p) {
  state start {
    b.extract(p.ethernet);
    transition select(p.ethernet.etherType) {
      0x8847: parse_mpls;
      0x0800: parse_ipv4;
    }
  }
  state parse_mpls {
    b.extract(p.mpls.next);
    transition select(p.mpls.last.bos) {
      0: parse_mpls; // This creates a loop
      1: parse_ipv4;
    }
  }
// other states omitted
}
```

## 12.10. Sub-parsers

嵌套子 parser
