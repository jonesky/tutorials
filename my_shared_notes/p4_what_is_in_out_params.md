p4_what_is_in_out_params.md

[TOC]

# 概要

in, out, inout 表示了 这些参数是有向的，

与之对应的是 无向 的， aka： 不加任何参数修饰的。

**什么是无向的参数：** 这些参数 由 控制层面 下发


#  何时 你 遇到  in, out, inout 修饰的参数


```C
control MatchActionPipe<H>(in bit<4> inputPort,
		inout H parsedHeaders,
		out bit<4> outputPort);
```


# in

这样的参数，表明他是 从 control panel 输入/下发的

我们不可以 修改 in 修饰的参数


# out 

这样的参数表明， 这是我们 data panel 要去 修改的，输出的。

我们能对 out 修饰的参数进行 修改。


# inout


表示可以读入，也可以写出。



