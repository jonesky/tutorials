# P4 Tutorial

可以直接跳到下方中文部分

## Introduction

Welcome to the P4 Tutorial! We've prepared a set of exercises to help
you get started with P4 programming, organized into several modules:

1. Introduction and Language Basics
* [Basic Forwarding](./exercises/basic)
* [Basic Tunneling](./exercises/basic_tunnel)

2. P4Runtime and the Control Plane
* [P4Runtime](./exercises/p4runtime)

3. Monitoring and Debugging
* [Explicit Congestion Notification](./exercises/ecn)
* [Multi-Hop Route Inspection](./exercises/mri)

4. Advanced Behavior
* [Source Routing](./exercises/source_routing)
* [Calculator](./exercises/calc)
* [Load Balancing](./exercises/load_balance)


# 中文部分by tt


p4_steps.md

[TOC]

# install

建议直接使用ova


## 使用微云里的ova 虚拟机导入vbox，直接可用


link： https://share.weiyun.com/5q791LD

![](assets/p4_obtain_required_sw-183d6b7a.png)

![](assets/p4_obtain_required_sw-3d705a49.png)


# 一些实例

参考： https://github.com/p4lang/tutorials

## 内容有：

1. Introduction and Language Basics
- Basic Forwarding
- Basic Tunneling
2. P4Runtime and the Control Plane
- P4Runtime
3. Monitoring and Debugging
- Explicit Congestion Notification
- Multi-Hop Route Inspection
4. Advanced Behavior
- Source Routing
- Calculator
- Load Balancing


# 实践

## 实践basic tunneling

ps： 所有步骤可以参考 tutorial ：

https://github.com/p4lang/tutorials/tree/master/exercises/basic_tunnel

![](assets/p4_runtime_intro-e4089c90.png)

- In h2's xterm, start the server:`bash ./receive.py`
- 然后，首先测试 without tunneling: 在h1 使用：`./send.py 10.0.2.2 "P4 is cool, and you" `

![](assets/p4_runtime_intro-d6ed7144.png)


没有启动h3的receive 的时候，h1 可以发送，当然没错，开启了h3的xterm 终端，并开启receive.py，如下：

![](assets/p4_runtime_intro-57d5ccb3.png)

## 接下来测试带有我们自己tunnel tag 的报文

- h1 send  h2 ：  `./send.py 10.0.2.2 "P4 is cool" --dst_id 2`
![](assets/p4_runtime_intro-2307f675.gif)

- h1 send to h2 by send to h3 :  `./send.py 10.0.3.3 "P4 is cool" --dst_id 2`


![](assets/p4_runtime_intro-91acc7fe.gif)
