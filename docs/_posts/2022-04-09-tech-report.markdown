---
layout: post
title:  "weekly report"
date:   2022-04-09 22:30:08 +0800
categories: weeklyreport
---

# 论文 

### [ceph](https://www.usenix.org/legacy/event/osdi06/tech/full_papers/weil/weil_html/index.html)


本周继续阅读了 ceph 当初的论文. 书接上回，每一个 MDS 都会维护一组文件节点， 每个树节点都用权重来代表当前节点的热点程度， 并且根据动态子树分区算法来调整每一个MDS负责的子树数量。 来确保整体的负载均衡

关于 OSD 上周已经说过了一些， 这里再总结一下， OSD主要关注的是数据的备份， 数据分布，数据安全， 数据故障恢复这些逻辑，说白了它的关注点并不是如何给client端提供数据（当然这个也很重要，不过并不是主要关注点， 因为大家都差不多）

最后就是对整个ceph集群做性能压测， 展现这个存储（包括 OSD集群和 MDS集群）在各种特殊情况下的性能值指标，以及最后的总结了和展望了一下未来的发展方向。

说实话这篇论文是很早之前的论文了， 因为之前一直听说ceph 怎么样怎么样， 但是从来没有详细了解过它，是我开始读这篇论文的初心。这个也是我正式开始读的第一篇论文（以前断断续续读过一些不过都记不太清了）, 之后会持续阅读业界存储相关的沦文 。


# 基础

### SATA & PCI & PCIe

由于上周看了下 NVMe 的基本情况， 这周着重看了下之前的协议演变， 首先， SATA & PCI & PCIe 都是在 NVMe之下的一层协议， 它定义了设备之间的方式，而NVMe主要定义了命令集和数字逻辑层。 SATA 和 PCI 协议是同级的, PCIe 是 PCI 的extension版本， 支持了点对点通信。而非 PCI的时钟通信， 这点大大提高了设备的效率. 最后还有M.2, U.2 这种物理接口

SATA 是 Serial AT  Attachments 的缩写， 其中 AT 是 AdvancedTechnology 的缩写， 由于这个词与IBM有着很深的渊源，所以统一简写为 AT 来解耦这个逻辑



# 日常

### 一个同事问的一个问题

一个同事问了一个之前我没有详细考虑过的问题，块设备挂载到容器之后容器为什么可以看到相应的块设备。
经过实验， 确认了在使用文件系统挂载块设备的方式下，在容器中是没有办法看到块设备。 因为块设备已经通过kubelet挂载到指定目录里了， 所谓在容器中看到块设备只是通过 mount/df 命令间接看到， 这里看到的同样是宿主机上面的块设备。
如果使用了裸设备方式挂载的情况下, 块设备则会通过 volumeDevice 中定义的块设备透出， 也无法直接看到宿主机上的块设备


# 社区

- 本周支持了社区 autoscaler 的一个 good first issue , 主要是为 某一个field 字段做校验。多亏了这个pr使得我对kubernetes quantity有了进一步的了解， cpu无法支持 m 单位一下的精准调度， 同样 内存也不支持1 byte一下的精准分配， 这些都是需要在validate 阶段进行校验的。 [pr](https://github.com/kubernetes/autoscaler/pull/4798)
- 多亏了一个客户反馈的问题， 使我这边发现了一下神奇的pvc/pv创建现象。 当pvc加了某些特殊的annotation的时候动态创建完成后会出现 pvc lost 和 pv available的现象 [issue](https://github.com/kubernetes/kubernetes/issues/109393)