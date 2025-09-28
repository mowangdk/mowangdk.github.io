---
layout: post
title:  "weekly report"
date:   2024-07-15 22:30:08 +0800
categories: weeklyreport
---

# 读书

### ai存储

1. 计算就是对事物的求解, 古人用龟壳问卦和现人用计算机搜索, 本质上没有区别.
2. 内存墙, 在 1994 年被创造, 用来定义一个问题, 处理器的性能超过了内存带宽. 内存限制了处理器的性能

英伟达按照规模和网络模式区分了这几个特定的AI基础设施类型：
● Cloud：基于以太网（各家Cloud都有自己的优化）构建的，小规模GPU服务集群，注重弹性和多租户
● AI Cloud：基于支持以太网的Spectrum-X构建的面对广大用户的AI cloud，支持大语言模型等大规模AI业务的使用。并且支持弹性与多租户。 就是类似DGX A100/H100为实例的大量机器组成的松耦合的云
● AI Factory：基于Infiniband/NVLINK互联构建的，满足少量用户的专用AI集群，针对超大基础模型开发设计, 基本上就是高端玩家的基础设施, 内存网络带宽可以达到 TB/s 级别.


AI对于基础设施的workload主要以东西向流量和南北向流量为主，东西向更多的是GPU-to-GPU，而南北向则重点是GPU-to-Storage，也是关注的重点。
在训练过程中对于存储最主要的诉求是dataset的读取诉求以及checkpoint写入读取。这两种请求的IO模型差异是比较大的。AI模型的类型和数据样本的大小决定了训练时候的性能差异。

AI 存储数据特性
训练
- 数据集
  - 并发读取
- checkpoint
  - 多路并发写入
  - 大块顺序读取
推理
- 模型
  - 随机读取
- kv-cache
  - 当使用性能较低的 GPU 卡型的时候， 相关推理的数据存储在 GPU 的内存中可以显著提升效率，通过硬件进行进一步压缩的 token 可以减少传输数据量和使用量

### xattr

The xattr (extended attributes) command on devices or files in Linux is used to manage extended attributes (metadata) associated with files or directories. These extended attributes can be used to store additional metadata not covered by the standard attributes provided by the filesystem.

扩展属性（xattrs）通常存储在磁盘上的独立数据块中，并通过inode.i_file_acl*在inode中引用。扩展属性最早的应用是存储文件ACL（访问控制列表）和其他安全数据（如Selinux）。通过挂载选项"user_xattr"，用户可以存储扩展属性，条件是所有属性名称必须以"user"开头。不过，这种限制似乎在Linux 3.0及以后的版本中已经消失了。

setfattr should be used on files or directories, not directly on block devices like /dev/vdb. this is only for setfattr...


use cases

```
setfattr -n user.comment -v "This is a comment" example.txt
setfattr -d example.txt
setfattr -x user.comment example.txt # remove
getfattr -d example.txt # verify remove
```

---2025-09-13 22:30:08 +0800 addon
we can know that xattr has four types, specified in the fully qualified namespace.attribute base on https://man7.org/linux/man-pages/man7/xattr.7.html

- user.mime_type
- trusted.md5sum
- system.posix_acl_access
- security.selinux

# 工作

### SRIOV

#### lspci pf vs vf 输出
lspci -s bdf -vvvv 这个命令 pf 和 vf 的输出是不一致的. pf 中包含 capabilities, 里面显示的声明支持了 Single Root I/O Virtualization (SR-IOV). 并且同样可以了解当前 pf 支持挂载 vf 的数量

Initial VFs: 63, Total VFs: 63, Number of VFs: 63


#### sriov_drivers_autoprobe

/sys/bus/pci/devices/<bdf>/sriov_drivers_autoprobe, 当往这个文件里面写 0 的时候, 这个 bdf 设备被挂载到节点上的时候就不会默认绑定 driver, 可以直接挂载 vfio


#### 新命令

lspci -t: Show a tree-like diagram containing all buses, bridges, devices and connections between them.


### nvme


controller 存在 primary 和 secondary, nvme list-secondary /dev/pf 会列出 当前 pf 中所有的 secondary controller, 其中默认第一个是给 pf 的, 剩下的可以根据 Secondary Controller State 来判断 ns 中 vf 挂载的状态

```
Identify Secondary Controller List:
   NUMID       : Number of Identifiers           : 64
   SCEntry[0  ]:
................
     SCID      : Secondary Controller Identifier : 0x0000
     PCID      : Primary Controller Identifier   : 0x0000
     SCS       : Secondary Controller State      : 0x0001 (Online)
     VFN       : Virtual Function Number         : 0x0000
     NVQ       : Num VQ Flex Resources Assigned  : 0x0006
     NVI       : Num VI Flex Resources Assigned  : 0x0007
   SCEntry[1  ]:
................
     SCID      : Secondary Controller Identifier : 0x0001
     PCID      : Primary Controller Identifier   : 0x0000
     SCS       : Secondary Controller State      : 0x0000 (Offline)
     VFN       : Virtual Function Number         : 0x0001
     NVQ       : Num VQ Flex Resources Assigned  : 0x0000
     NVI       : Num VI Flex Resources Assigned  : 0x0000
   SCEntry[2  ]:
................
     SCID      : Secondary Controller Identifier : 0x0002
     PCID      : Primary Controller Identifier   : 0x0000
     SCS       : Secondary Controller State      : 0x0001 (Online)
     VFN       : Virtual Function Number         : 0x0002
     NVQ       : Num VQ Flex Resources Assigned  : 0x0001
     NVI       : Num VI Flex Resources Assigned  : 0x0002
   SCEntry[3  ]:
................
     SCID      : Secondary Controller Identifier : 0x0003
     PCID      : Primary Controller Identifier   : 0x0000
     SCS       : Secondary Controller State      : 0x0001 (Online)
     VFN       : Virtual Function Number         : 0x0003
     NVQ       : Num VQ Flex Resources Assigned  : 0x0001
     NVI       : Num VI Flex Resources Assigned  : 0x0002
```

卸载/挂载 sriov-nvme 设备一般有如下步骤

将 vf attach 到某一个 pf 的 某一个 namespace 上

nvme attach-ns /dev/nvme1 -n %d -c %s

-n: namespace_id
-c: nvme ctl_id

分配资源
nvme virt-mgmt /dev/nvme1 -c %d -r %d -a %d -n %d 

-c ctl_id
-a 8: 分配资源
-r 0: 分配 VQ Resources
-r 1: 分配 VI Resources （后端不支持分配中断）

online, 使能 vf

nvme virt-mgmt /dev/nvme1 -c %d -a 9

-c ctl_id
-a: 9 使能
-a: 7 关闭, 查看 man 命令即可


绑定 driver

sudo echo nvme > /sys/bus/pci/devices/<bdf>/driver_override
echo <bdf> >/sys/bus/pci/drivers_probe

查看 pf 指定 namespace 的基础信息, for example nguid

nvme id-ns /dev/nvme1 -n 2  -o normal