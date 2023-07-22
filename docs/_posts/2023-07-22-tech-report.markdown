---
layout: post
title:  "weekly report"
date:   2023-07-22 22:30:08 +0800
categories: weeklyreport
---

# 读书

### 大话存储

继续读 nas, 这里提到了一个 SAN 和 NAS 的区别, 作者更倾向于 SAN 包含 NAS, 我也认同, 因为 SAN 是一个存储阵列,至于是否为这个存储阵列加一个集中式的文件系统, 这个就完全看客户的需求了


SAN vs NAS 的访问路径

![SAN vs NAS](/assets/img/ebs-vs-nas.png)

nas 架构的路径在虚拟目录层和文件系统层通信的时候, 用以太网络和 TCP/IP 协议代替了内存, 这样做不仅增加了大量的 CPU 的指令周期(TCP/IP逻辑和网卡驱动), 用了低速的传输介质(以太网vs内存速度) . 在 SAN 方式下, 虽然路径中比 NAS 多了一次 FC 访问, 但是 FC 整体的协议比 tcp 轻量, 再加上 FC 逻辑是在 硬件 driver 上执行, 如果NAS没有后端磁盘瓶颈, 并且没有快于内存的网络访问方式, NAS 速度永远无法超过 SAN 架构


# 社区

本周没有进展, 我今天问下吧


# 工作

### kworker zombie

本周找了下内核同学帮忙共同看下, 也没有看出来什么结论, 可以通过 strace 看到的是 container-shim 进程一直在等信号返回. shim 进程本身所在的 bundle json 日志也没有什么特别的输出(/run/containerd/io.containerd.runtime.v2.task/)

### chmod

本周在评审中查了下相关 chmod 相关的逻辑, chmod 是不会处理目录下的软链的. 相关说明见 man chmod


### 硬链接
硬链接只能引用同一个文件系统的文件, inode 是没办法迁移的, chown 自然也不存在硬链接的问题


### tmpfs limit
Tmpfs , 在 mount 的时候可以通过 size option 来指定 tmpfs 的大小, 默认为除去 swap 空间物理内存的一半

https://www.kernel.org/doc/Documentation/filesystems/tmpfs.txt


