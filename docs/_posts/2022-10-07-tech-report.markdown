---
layout: post
title:  "weekly report"
date:   2022-10-07 22:30:08 +0800
categories: weeklyreport
---

大概鸽了差不多一个月... 之前因为各种事情被拌住了, 放完假开始更新

# 读书

### 论文

https://www.usenix.org/conference/osdi22/presentation/zhong

这篇论文算是读完了. 的确对我还是有很大的启发的, 看完之后也对目前业界存储 BDF 的方向有了一定的了解, 大概有如下几个方面.
- 加速I/O, 这个在网络方面比较常用, 主要是 XDP . 在 RX 路径上加 hook, 以便在内核层面直接将返回数据进行转发, 不必在返回到 userspace. 也减少了层次传递的损耗. 存储方面现在有一个 ExtFUSE, 因为 FUSE 是 userspace 的文件系统, 下发到 VFS 的请求会被 FUSE driver 转发给用户空间的 daemon 进行处理, 这个 ExtFUSE 允许用户使用 hook 在内核阶段处理数据, 使得FUSE 文件系统的数据不必返回给 UserSpace

- kernel-bypass. 这个intel SPDK 算是做到极致了, 但是正如论文中所说, SPDK 有性能(polling)上和安全上的隐患(绕过文件系统的 authorization). 同样上文中做的 XRP 也算是基于 NVMe driver Bypass 了部分的请求. 同样 SPDK 也是属于 DPDK 家族里面的一员

- near storage device compute. 这个就比较麻烦了. 因为依赖 storage device 内置/外置的芯片来做远端计算. 这个我估计暂时够呛, 而且在没有统一的标准之前只能单独针对硬件进行适配. 估计未来发展性不会很高

-  自定义扩展文件系统/操作系统. 这个很久之前就有了, 因为离我们也比较远, 暂时先放



### 大话存储

还在啃.

# 日常

### dpdk

全称为 data plane develop kits, 主要是为了将数据从内核链路上解放而做的一种工具, 由 intel 开发. 有了他我们可以从冗长的内核处理栈中解放 io.解决 C10K 的问题. dpdk 主要包含如下功能
- 内存池: 禁止掉 Userspace 的内存和 内核内存之间的 copy, 在切换的时候只做控制权转移, 这样可以省掉内存搬移的时间
- UIO: 通过拦截硬件中断.截断内核处理栈, 将设备符直接映射到用户空间, 由用户程序进行处理
等等.

相关缺点也很明显, 正如上面所说, dpdk 基本上是基于使用 polling 来代替中断机制的一种方法, 当系统负载不高的时候, polling 会持续占用 cpu 资源. 造成浪费. 当然, dpdk 进程超过 core(hyper-thread)的数量的时候, 也会照成明显的性能下降.


### spdk
全称为 storage plane develop kits, 是一系列的库或者是工具， 主要是提供了编写高性能，可扩展的用户状态的应用。它通过以下几种技术来获得高性能
- 将所有的操作都放到用户侧， 从而避免 systemcall ， 并且可以从用户侧开启 0 拷贝
- 通过轮训硬件， 而不是中断来进行提速
- 在IO 路径中避免使用锁，主要依赖信息传递

SPDK的核心是一个用户空间、轮询模式、异步、无锁的NVMe驱动程序。该程序直接在用户空间与 SSD 硬件进行访问，在此基础上同样提供了 iscsi，nvme-of, vhost 等一系列存储相关的处理工具


### IOMMU

IOMMU 是一种内存管理单元, 他将具有 DMA 能力的总线链接到内存. 也就是将主存的虚拟地址和设备的物理地址进行映射. 相对于直接访问硬盘(物理寻址). 他增加了相应的页表转换开销 & 页表管理开销.

![iommu](/assets/img/iommu.jpg)