---
layout: post
title:  "weekly report"
date:   2023-08-26 22:30:08 +0800
categories: weeklyreport
---

# 读书

### 大话存储

由于 ip-san 和 fc-san 各有各的优点, ip-san 由于其通用性被底层用户所喜爱. fs-san 由于其稳定性和传输速率被高级客户所喜爱, 所以两家在考虑融合协议. 

 首先书中用了汽车和火车比喻为网络中链路层的不同. 汽车相当于tcp/ip, 虽然可以到的地方多, 但是沿途会受到红绿灯和堵车的影响, 没有办法走的特别快, 火车走铁道, 相当于 fc 协议, 需要提前在两地中间架设铁道. 运输速度快. 但是费时费力费钱. 这个比喻还是很经典了. 至于三种协议融合方式的话  协议融合分为三种方式, 调用, 隧道, 映射. 说实话刚开始的时候不太明白这些意思. 不过幸亏后面给解释了一下. 像是 tcp 使用 ip 来进行路由, 这种就叫做调用. 因为两种协议功能没有冲突(虽然也是将 tcp package 全部包起来再加上 ip header 进行传递) 隧道则是两个协议在某种功能上都有实现.但是一个还是要经由另一个协议进行传递, fc & ip 就是这种. 至于最后一种 map 则是将前一种协议完全映射到后一种协议, 这种方式尽管 payload 的大小降低了很多. 但是成本太高, 每个协议都要自定义. 实在是没有意义, 当然 vpn 这种就是 ip 映射到 ip 的例子. 将内网 ip 包当做外网 ip 的payload 进行传递. 当然内网的 ip 包是可以做加密和压缩的. 这些都可以做到防篡改的

# 社区

subtree updated PR merged. 

# 工作

### sriov

又要跟 sriov 打交道了, 所以这周又仔细的研读了下 sriov 协议. 首先这个 ko 功能的作用是用于将一个插入到物理机的物理设备映射到多个虚拟机上.首先他会创建一个 Physical Function 的设备, 这个设备跟完整的网络/存储设备一样. 具备所有功能. 同时也具备管理其他虚拟出来的设备的功能. 其次就是 Virtual Function, 这个 VirtualFunction 是 sriov 管理的.可以独立的直通到用户创建的虚拟机中, 但是这个设备只具有 io 的功能, 所以他并不具备其他的功能, 假如这个 sriov 设备是一个网卡, 那么 pf 将在宿主机上存在, 每创建一个虚拟机,就在这个组里面创建对应的 vf 设备, 然后将这个 vf 设备直通到虚拟机中. 这样就可以实现多个虚拟机共享一个物理设备的功能.

这个 VF 正如前面所说, 是没有 permanent unique MAC address 的.并且每次重启, VF 的 macAddress 都会改变. 但是 PF 是有的

bdf: bus device function, use to describe pci or pcie devices

 
> PCI Bus number in hexadecimal, often padded using a leading zeros to two or four digits
> A colon (:)
> PCI Device number in hexadecimal, often padded using a leading zero to two digits . Sometimes this is also referred to as the slot number.
> A decimal point (.)
> PCI Function number in hexadecimal


举个例子, 下面这个 bdf 就是 Bus 0, Device 2, Function 0:

00:02.0

使用如下命令查看 bdf 设备有多少个 virtual-functions

```
cat /sys/bus/pci/devices/0000\:00\:02.0/sriov_numvfs
```


### csi nodeexpandvolume

这周突然发现, 如果直接改了一个已经绑定的 pv 的 storagecapacity 会触发 kubelet 调用节点上 csi.NodeExpandVolume 接口扩容用户的文件系统. 还挺神奇的一个点.
