---
layout: post
title:  "weekly report"
date:   2023-11-26 22:30:08 +0800
categories: weeklyreport
---


# 读书

### 大话存储

本周继续快照相关. 之前看到了基于文件系统的快照. 现在要做基于物理卷的快照. 它比文件系统实现起来要简单一些. 因为 lun/volume 在底层文件系统上的物理位置是恒定的.不像文件系统一样可以随意分布. 其元数据的地址也是恒定的. 在制作快照的过程中需要复制的元数据链就更小. 比较困难的是后面. 因为要做 CoFW or RoFW. 并且要记载编辑数据对应的实际物理指针. 相当于把文件系统的一部分功能搬到卷层来实现了.

CoFW: 因为首次更新要复制所有的数据, 所以首次更新时底层 io 压力较大
RoFW: 因为是保留元数据, 所以每次读请求的时候都需要 redirect io 到新的地址. 后面读请求的压力较大

### rust 程序设计

今天看了下所有权, 因为之前有大概了解过, 所以这里看的很快. 比较好玩的一点是作者拿 python, c++, rust 这三种语言在传递指针的时候内存分布做了一些对比, 简单贴个图

![python vs c++ vs rust](/assets/img/arrays_memory_model.JPG)

当上述数组所有权进行转移的时候, python 是直接增加引用技术, c++ 则是直接复制相关元数. rust 则是转移所有权. 这么一看确实 rust 是要更加优雅一些, 但是的确也增加了很多复杂度, 尤其是涉及到程序控制流里面.一不留神就可能引用没有所有权的变量. 的确是要注意一些

# 社区

本周 CSI 协议相关的 SLA 证书批下来了, 意味着之后可以直接给 CSI 协议提 pr 了. 虽然说后面的工作可能不是很多, 不过能推上去也挺好

# 工作

### mount 失败

```
Warning  FailedMount  10m                  kubelet  Unable to attach or mount volumes: unmounted volumes=[etcd-data], unattached volumes=[xxx]: timed out waiting for the condition
Warning  FailedMount  3m31s (x4 over 17m)  kubelet  Unable to attach or mount volumes: unmounted volumes=[etcd-data], unattached volumes=[xxx]: timed out waiting for the condition
Warning  FailedMount  2m50s (x4 over 21m)  kubelet  MountVolume.MountDevice failed for volume "d-xxx" : rpc error: code = Internal desc = mount failed: exit status 32
Mounting command: mount
Mounting arguments: -t ext4 -o shared,defaults /dev/vdc /var/lib/kubelet/plugins/kubernetes.io/csi/pv/d-5xxx/globalmount
Output: mount: /var/lib/kubelet/plugins/kubernetes.io/csi/pv/d-xxxx/globalmount: mount(2) system call failed: Structure needs cleaning.
```

神奇的报错, 格式化之后直接挂载失败, 一开始以为是加密盘的问题, 后面反馈是硬件上的问题. 还是需要继续追下


