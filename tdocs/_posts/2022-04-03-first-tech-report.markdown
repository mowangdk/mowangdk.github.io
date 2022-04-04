---
layout: post
title:  "first-tech-report"
date:   2022-04-03 20:30:08 +0800
categories: weeklyreport
---

## 论文 

### [ceph](https://www.usenix.org/legacy/event/osdi06/tech/full_papers/weil/weil_html/index.html)


本周阅读了 ceph 当初的论文.这里简单总结一下. 首先 ceph 定位是一个分布式网络文件系统. 它的整体架构分为三部分

- client: 面向用户的客户端, 通过宿主机上 vfs 的统一接口对用户暴露出 POSIX 文件系统接口. 我们可以像操作本地文件一样操作 ceph 上的数据
- MDS(MetadataServer): ceph 的主要卖点就是在于这个独立的元数据服务, 它创新性的提出了将元数据和实际数据存储分离的方案, 好处我们稍后再提, 总之, ceph 的元数据是要单独向 MDS 进行请求获取的. 这个思想后续也被很多其他系统借鉴, 比如 google 的元数据服务Grafeas.
- OSD(ObjectServiceDevice): ceph 用对象存储设备进行内容数据的存储, 它将所有的设备组成了一个 cluster, 这个 cluster 本身有自运维的能力, 可以动态的扩容,缩容, 进行灾备,数据恢复功能, 这些都是独立的,无需 client 端感知的

读取数据的大概流程基本上就是, client 端首先从 MDS 获取目标目录的元数据，其中元数据会记录文件的位置， client 端收到数据位置的信息之后，直接访问 OSD 获取数据

### 好处

首先最重要的就是元数据和数据本身的分离, 经过统计, 元数据的请求差不多占用的磁盘整体负载的一半, 这样将两者分开可以有效的减缓数据盘的压力, 并且元数据的 server 可以单独进行扩容. 这里 MDS 用了一种 [动态子树分区](Dynamic metadata management for petabyte-scale file systems. In Proceedings of the 2004 ACM/IEEE Conference on Supercomputing (SC '04). ACM, Nov. 2004.") 的系统来存储数据目录结构. 这个算法主要是为了分配目录结构。
同样 OSD 集群也是可以单独进行扩容的， 同时 OSD 集群里面还内置了数据的多备份 & 热数据的迁移等功能。 这里简单的介绍下 OSD 存储数据的方式。 首先，它将一份文件分成了若干个Objects（用于支持同一个文件的并发读写）， 每个 Objects 使用算法都分配到不同的 PlacementGroups 里面， 之后， OSD 是以 PG 为单位进行数据的存储， 最后就是底层的 ObjectStorageDevice 的任务了， 它将每一块 PG 都存储到对应的一个块设备上，如果声明了数据需要多副本， 那么就将 对应的 PG 分别复制到不同的 Device 上。这三份数据会按照主备进行服务， 平时提供服务的主要是主 PG， 如果主 PG 挂掉或者是网络不可达， 备 PG 将会统一对外提供读写服务




## 基础

- NVMe
本周主要熟悉了一下 NVMe， 因为最近工作中总是遇到， 所以找个时间了解下， NVMe 本身是一种逻辑设备接口，用于访问通过PCIe 接口链接上来的非易失性存储设备。 它的出现主要是为了提高块设备的使用效率， 之前的逻辑设备接口（AHCI）. 主要是针对HHD设备设计的。总所周知这种设备的io效率非常低下（寻址）. 而 NVMe 则是专门为 SSD 固态设备设计的接口. 他可以充分的发挥 ssd 盘快速寻址以及并发度的优势。换句话说， 使用AHCI 的SSD 盘 完全没有发挥出它的本来性能.用于访问通过PCIe 接口链接上来的非易失性存储设备。
另， NVMe逻辑是存储在NVMe芯片上的，这个芯片是与存储设备绑定到一起。所以并不涉及 PCIe 接口的兼容性问题
