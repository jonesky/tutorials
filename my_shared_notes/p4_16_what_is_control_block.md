p4_16_what_is_control_block.md

[TOC]

# 概要

p4_16 中的control block， p4-16-v1.1.0-spec.pdf chapter 13

其实它和c 语言 中的 函数很像，

比如能声明变量，创建一些对象（在 P4 里就是table）, 实例化外部对象。

但是 不能包含 loop， 所有的行为 都应该是DAG ：有向无环图

一个 control block 包含的有： match action , deparser， 以及其他形式的 包处理


# 前言

从 parser 中提取的bits ， 形成了多种多样的header， Control block 就是要操作这些 header

Control block 的行为就像是 指令式语言。

match-action 就在control block 进行

![image](https://ws2.sinaimg.cn/large/005JrW9Kgy1fzf47r5ju7j30f905jgly.jpg)

 数据层面提供 行为指令代码（aka P4 程序提供）， 控制层面提供 行为需要的辅助数据。这些下发的数据，由数据层面进行绑定。

# 13.1. Actions

  行为里的数据，（如果有的话）通过控制层面下发， 通过 数据层面 读取。

   在 action 里， 你不可以使用 switch  语句。 语法检查 允许这些，但是 语义检查 不允许。

  一些特定的目标硬件 甚至 不支持 条件语句，只能 是 直线的操作流


## 13.1.1. Invoking actions

调用一个action方式 可以 是  隐式 的，或者显式的。

### 什么是隐式的：

 使用table 

### 什么是显式的： 

明显调用  其他的 action 对象， 可能是从 一个  control block 中调用，也可能从另外一个 action 中调用； 

ps： 在 显式的调用中，action 的参数都需要显式 地 提供 ， 包括那些 无向 的参数，此时，无向的参数 表现的就像 有 `in` 修饰的参数（aka： 你只能读取，不能做修改）


# 13.2. Table

使用 ： match-action unit 称呼这个 table  似乎是 更合适的。

### MA 的工作流

![image](https://wx2.sinaimg.cn/large/005JrW9Kgy1fzf5v7cukkj30oc0fuq51.jpg)


### 基本的表 的属性包括：

1. key

2. actions

3. default action 如果你没有显式的申明这个default action 那么NoAction 会被编译器插入

4. size 表格的希望的size


除了上述的属性， 还有的是target 特定的属性，上述4个属性，其实默认带有`const`关键字的修饰， 这说明，控制层面不可以动态地改变他们。


### 13.2.1. Table properties

本小节具体讨table 的 6 个属性：

#### keys： 

key 的定义形式 是` e : m` 

其中 e  是 : 表达式 

m : 是 match kind  可能是 ternary, exact, lpm（这三者是p4 core 预定义的）, 有的机器甚至支持 regexp。

match kind 代表的可能是多种算法。

```

table Fwd {
	key = {
		ipv4header.dstAddress : ternary;
		ipv4header.version : exact;
	}
	...
}
```

ps： 每个key 可选一个 `@name` 注解，这令控制层面 可见其名字。

#### actions

典型的例子就是：

```
action Drop_action() {
	outCtrl.outputPort = DROP_PORT;
}
// EthernetAddress sourceMac 由控制层面下发 ，action 在数据层面可以认为是一种回调机制
action Rewrite_smac(EthernetAddress sourceMac) {
	headers.ethernet.srcAddr = sourceMac;
}
table smac {
	key = { outCtrl.outputPort : exact; }
	actions = {
		Drop_action;
		Rewrite_smac;
	}
}
```


##### 对于action 中的一些参数的约定


什么是 in ，out, inout 这些有向参数，可以看 [p4_what_is_in_out_params.md](./p4_what_is_in_out_params.md)

```
action a(in bit<32> x) { ...}
bit<32> z;
action b(inout bit<32> x, bit<8> data) { ...}
table t {
	actions = {
		// a; -- 非法，in修饰的 x 的参数 需要显式指定
		a(5); // 合法，在数据层面 把 5 绑定到 x 参数上
		b(z); // 合法， 将 z 绑定到 in（aka： inout） 修饰的b（） 的x 参数上
		// b(z, 3); -- 非法，无向的参数（ bit<8>  data） 不能被绑定上 3
		// b(); -- 非法，inout 的参数， 类似于 in 的参数， 一定需要 显式绑定
	}
}
```



#### default actions


实际上我们看到的default action 是这样的：（是带有 const 修饰的）

```
const default_action = Rewrite_smac(48w0xAA_BB_CC_DD_EE_FF);

```

注意， 某些 default action 需要给定参数


- [ ] todo 0122 理解下方代码 中 不同 参数 的意思


```
default_action = a(5); // OK - no control-plane parameters
// default_action = a(z); -- illegal, a's x parameter is already bound to 5
default_action = b(z,8w8); // OK - bind b's data parameter to 8w8
// default_action = b(z); -- illegal, b's data parameter is not bound
// default_action = b(x, 3); -- illegal: x parameter of b bound to x instead of z
```



#### entries


一般来说， entries 是 由 控制层面下发的。

但是我们可以在P4 中直接写一些 静态的entry， 这有利于一些算法的速度提升。

entry 只能 被读，不能被控制层面 移除， 修改

entry 是 不可变更的，aka： 它有一个默认的 `const` 修饰。但是后期的P4 可能支持 可变的 table entry

entry 的匹配按照代码里写的顺序， 并且在第一个匹配处停止。

实例：


```c

header hdr {
	bit<8> e;
	bit<16> t;
	bit<8> l;
	bit<8> r;
	bit<1> v;
}
struct Header_t {
	hdr h;
}
struct Meta_t {}

control ingress(inout Header_t h, inout Meta_t m,
				inout standard_metadata_t standard_meta) {
	action a() { standard_meta.egress_spec = 0; }
	action a_with_control_params(bit<9> x) { standard_meta.egress_spec = x; }
	table t_exact_ternary {
		key = {
			h.h.e : exact;
			h.h.t : ternary;
		}
		actions = {
			a;
			a_with_control_params;
		}
		default_action = a;
		// 添加 静态的entry ， 静态的entry 匹配，按照 声明的顺序， 就像 enum
		//  在第一个匹配处 停止
		const entries = {
			(0x01, 0x1111 &&& 0xF ) : a_with_control_params(1);
			(0x02, 0x1181 ) : a_with_control_params(2);
			(0x03, 0x1111 &&& 0xF000) : a_with_control_params(3);
			(0x04, 0x1211 &&& 0x02F0) : a_with_control_params(4);
			(0x04, 0x1311 &&& 0x02F0) : a_with_control_params(5);
			(0x06, _ ) : a_with_control_params(6);
		}
	}
}
```


#### size

可选的属性

对于那些table： 需要动态 标定尺寸 的，会把size 当做初始容量 

#### 附加属性


不同用处的表可能有不同的属性，比如： 稀疏的hash 表，稠密的表

编译器的后端，都需要知道一些表格的额外信息。

### 13.2.2. Match-action unit invocation


令一个  Match-action unit  生效的方法是 ： 使用`apply` 方法

apply 方法的内部 可以 理解为 返回了 一个 枚举变量（action list ） 和 一个结构体变量 （包含 一个 bool hit)

```c
if (ipv4_match.apply().hit) {
	// there was a hit
} else {
	// there was a miss
}

```


# 13.3. MA 流水线的抽象机器

调用 `return` 会从 当前的control block 返回

调用 `exit` 会从  所有的外层的control block 返回。

调用 `apply` 会像 上述描述的一样  进行匹配行为。



# 13.4 调用其他的control block


子control block 在被调用之前，需要被实例化

```
control Callee(inout IPv4 ipv4) { ...}
control Caller(inout Headers h) {
	Callee() instance; // instance of callee 1. 实例化
	apply {
		instance.apply(h.ipv4); // invoke control 2. 调用
	}
}
```
