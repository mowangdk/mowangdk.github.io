---
layout: post
title:  "weekly report"
date:   2025-06-06 22:30:08 +0800
categories: weeklyreport
---


# 读书

### Efficient Linux at the Command Line

还是跟其他的一些command line 有一些不同的， 有一些命令确实能够提效， 不过在现在的这个年代，对于敲代码的提效好像不是那么重要了, 各种的terminal都集成了 ai 智能终端了， 



# 社区

### volume populator

简单来说就是原先的 ```pvc.spec.dataSource``` 的替代， 这回不止支持 VolumeSnapshot， 甚至可以扩展为任意的 CRD, 其实个人看来可能跟社区的 image volume mount 可以有类似之处，但是扩展性可比默认提供的好多了


https://kubernetes.io/blog/2025/05/08/kubernetes-v1-33-volume-populators-ga/



### mutable pv affinity

enhencement #5382 顺带看了下 CSI 协议的相关内容，先大致过一遍吧，还有也顺带过了一遍 csinode allocatable count 的内容， 也是参考这个 KEP 写的 #5382


# 工作

### protobuf IDL(Interface Definition Language)

https://protobuf.com/docs/language-spec

### CRD PB

https://kubernetes.io/zh-cn/docs/reference/using-api/api-concepts/#protobuf-encoding-compatibility

因为内置的 core 类型都是直接由 golang struct 代码来生成的， 是可以直接生成pb 的protobuf 的, 但是如果是 CRD 定义， 因为不存在 apiserver 端的 protobuf decode，所以只能用json 通用编码， 这个原因在下面 cbor 的kep里面有提及， 看起来社区从前年开始就计划了，也是经过了很长时间

现在出来了一个 CBOR 的编码（https://www.rfc-editor.org/rfc/rfc8949），貌似可以支持.
kep: https://github.com/kubernetes/enhancements/tree/master/keps/sig-api-machinery/4222-cbor-serializer

注：
模糊测试 (fuzz testing)，或称为模糊测试，是一种软件测试技术，通过向计算机程序提供意外、格式错误或随机的数据（即“非受信输入”）来发现软件缺陷、安全漏洞或其他问题 1。

其主要目标是：

发现程序漏洞：模糊测试可以揭露意外异常、拒绝服务 (denial of service) 1、内存错误、数据竞争、未定义行为 1、内存损坏 1 和远程代码执行 1 等问题。
处理非受信输入：当项目涉及反序列化、解码和处理非受信输入时，模糊测试尤其有用 1。
提高软件健壮性：它有助于减少高保障软件中的错误 1。
例如，在 Kubernetes 的一个 Go 包中，FuzzDecodeAllocations 函数被设计为一个模糊测试目标，用于发现因输入导致解码时分配过大内存而引发程序崩溃的情况 2。像 Atheris 这样的覆盖引导式模糊测试工具，则通过最大化代码覆盖率来寻找更多潜在问题 1。

### utab 

/run/mount/utab 文件是 Linux 系统中由 mount 命令管理的一个文件，主要用于记录由普通用户通过 user 选项挂载的文件系统信息 1。其目的是为了让挂载该文件系统的用户能够再次卸载它 1。例如，当用户 user1 挂载设备时，mount 命令会将类似 SRC=/dev/sdb1 TARGET= ROOT=/ OPTS=user=user1 的信息写入 /run/mount/utab 1。

这个文件通常不会默认存在，只有当普通用户执行了带有 user 选项的挂载操作时，mount 命令才会在需要时创建并写入该文件 1。由于 root 用户本身就拥有挂载和卸载文件系统的权限，因此其挂载操作不会将信息写入 /run/mount/utab 1。mount 命令自身会进行用户ID（UID）检测，以确保操作权限 2。



### PDB

它的核心作用是限制在自愿性中断（Voluntary Disruptions）期间，某个应用（Pod 集合）能够同时不可用的 Pod 数量。在 Kubernetes 集群中，为了进行维护、升级或缩容等操作，有时需要将节点上的 Pod 驱逐（evict）走。这些操作是“自愿性”的，因为它们是由集群管理员或自动化系统主动触发的。

如果没有 PDB，当你对一个节点执行 kubectl drain（排空节点）操作时，该节点上的所有 Pod 都会被强制驱逐。如果你的某个应用的所有 Pod 都恰好运行在这个节点上，或者同时被驱逐的 Pod 数量过多，那么你的应用就可能暂时完全不可用，导致服务中断。PDB 的目的就是为了避免这种情况，确保在这些自愿性中断发生时，你的应用始终保持在至少一定数量的可用 Pods。
