---
layout: post
title:  "weekly report"
date:   2022-05-14 22:30:08 +0800
categories: weeklyreport
---

# 读书

这周还是继续读 Understanding Linux Network Internals 这本书, 这两周开始读第二章的内容, 第二章整体介绍了网络中重要的数据结构(也只有两个, sk_buffer & net_divice). sk_buffer 主要是 socket 使用的数据结构,  net_device 是网络设备使用的数据结构(NIC or VLAN). 并且将这些数据结常用的方法也介绍了一遍. 其实对编程人员来说还是 sk_buffer 比较实用, 因为我们打交道最多的还是 socket , network device 这层我们日常还是接触不到的. 简单拿 sk_buffer 的方法介绍一下, 发一个 tcp package 的时候是如何进行数据填充的, 首先, 我们按照从物理层到 tcp 层所有协议头所占用的最大容量之和来在 buffer 中预留空间, 然后开始填入 payload 数据. 填写完数据之后开始按照 tcp header -> ethernet header 的顺序往之前预留的 buffer 里面 push 协议头数据. 除了这个之外第二章还包含了很多的图示说明,可以更好的理解相关的内核操作, 虽然知道了这些其实对我之后的编程没有什么实质性的帮助, 但是了解这些内核设计我认为还是很重要的, 至少之后在遇到类似的设计场景的时候可以参考.毕竟内核的设计基本上是最优的设计了

# 日常

### pod starting time optimize

最近工作的主要事情之一就是优化 pod 的启动时间, 主要涉及两个方面, 一个是管控侧 kcm 的逻辑优化, 另一个是节点侧 kubelet 的逻辑优化, 对比了 k8s 1.20的社区代码和现在的社区代码(1.24),发现 kcm 上的代码没有特别大的变动,只是改了一下文件路径. kubelet 上倒是做了一些优化,来加速 pod 上面的持久卷的绑定速度. 简单整理了一下调用流程关系, 后续附上

![csi mount costs](/assets/img/rose.jpg)

# 社区

本周社区没有进展, 活太多了, 效率也不是很高, 尽管有几个可以做的东西,但是考虑到工作量还是没有接, 先放着吧