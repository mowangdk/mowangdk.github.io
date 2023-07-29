---
layout: post
title:  "weekly report"
date:   2023-07-22 22:30:08 +0800
categories: weeklyreport
---

# 读书

### 大话存储

磁盘发展阶段:

- 全整合阶段:   就是物理盘在 cpu 在同一台机器上, 最常见的就是一个笔记本 DAS 架构
- 磁盘外置阶段:  通过 SCSI 接口/PCI 接口接入的硬盘. 也就是外接硬盘, 同样属于 DAS 架构
- 外部独立阵列: 在电脑外部部署 raid 阵列, 尽管还是通过 SCSI 接口, 但是已经属于 SAN 架构了
- 网络化独立磁盘阵列: 主机通过 FC 访问磁盘阵列, 并且这个磁盘阵列可以被多个节点同时访问. 属于完全的 SAN 架构了
- 独立 NAS: 主机不需要任何适配器, 只需要有网卡即可访问, 属于痩主机阶段, 后端 nas 既可以是完全独立的机箱,也可以是网络存储阵列
- SAN 与 NAS的融合: 也就是现代云厂商的基本主机, 既可以挂载 SAN 块设备, 也可以挂载 nas 网络文件存储
- System On Chip: 当所有的计算单元, 存储单元都收缩到同一个 chip 上面, 资源和空间利用率都能得到极大提高 

# 社区

社区回了, 但是还没时间看, 准备今天看下


# 工作

### major, minor

这周查看一个 kubernetes 代码才知道, 即使是 nfs, overlay 这种挂载点也是有 major & minor 的, 本质上其实是每一个文件系统都会有一组. 至于这种没有实体设备的,则由os 虚拟化出一对设备来保证属性完整, 当然, 以 nfs 为例, 这个虚拟化的设备是按照 server 为粒度的, 每一个文件系统对应一个, 一个文件系统下面不同的子路径其实对应的是同一个设备号


### sealed, unsealed

跟外部公司开会的时候提到的, 指的是把某种东西封装起来, 算是一个习惯性用法吧

### kubelet 多次调用 NodePublishVolume 

在 kubelet 通过标准 NodeStageVolume & NodePublishVolume 挂载 volumes 之后, 因为触发了 desiredStateOfWorld.processPodVolumes 导致要调用 MarkRemountRequired 重新 remount 一次, 结果发现是有两个 pod 同时引用了同一个 volumes 导致, 当有新的 pod 加入到 DSOW 的时候,  就会尝试重新挂载这个 pod 引用的所有的 volumes

```
I0728 19:25:17.375314   16211 desired_state_of_world_populator.go:316] "Added volume to desired state" pod="deploy/xxx1" volumeName="data" volumeSpecName="d-zm0duzs3t3cs3xxxxxxx"
I0728 19:25:24.529542   16211 desired_state_of_world_populator.go:316] "Added volume to desired state" pod="deploy/xxx2" volumeName="data" volumeSpecName="d-zm0duzs3t3cs3xxxxxxx"

```