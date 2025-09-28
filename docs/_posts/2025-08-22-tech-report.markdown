---
layout: post
title:  "weekly report"
date:   2025-08-08 22:30:08 +0800
categories: weeklyreport
---


# 读书

### NearOOM

Linux 划分了 high、low、min 三个内存水线。当系统剩余内存低于 low 水线后，内核会唤醒 kswapd 进程会被唤醒开始进行异步的内存回收，此时对系统没什么影响；系统剩余内存低于 min 水线后，内核会阻塞要分配的内存进程尝试尽可能地回收所有可回收的内存（主要是文件缓存以及一些内核结构体缓存），回收的过程中可能涉及到将文件缓存写入磁盘或遍历一些内核结构体，从而导致系统负载飙高、应用被阻塞。如果能成功回收内存并满足申请需求则不触发 OOM。

更糟糕的是，系统可能进入一种 Near-OOM 的活锁状态，即内核一边在尝试回收文件缓存；但是应用运行过程中从磁盘加载代码段等行为也在不断产生文件缓存，那么就会使整个系统负载持续飙高，甚至发生夯机。

为了应对 Near-OOM 现象，核心就是“快” OOM，在内核还在犹豫要不要 OOM 的时候，我们就替他做出决定！目前业界已有的方案主要是通过用户态提前杀死相关进程来提前释放内存，比如应用较为广泛的是 Facebook（Meta）推出的 oomd。oomd 目前已经集成于 systemd 中成为 systemd-oomd，且从 Ubuntu 22.04 开始集成于 Ubuntu 中。但是 oomd 方案存在以下问题：

- 与 cgroupV2 以及 Linux 内核的 PSI（Pressure Stall Information）特性深度绑定。但 cgroupV1 目前仍然是云计算中主流 cgroup 版本，且由于 PSI 功能有一定的性能开销，在大部分云计算场景中都是默认关闭的。

- 只支持以 cgroup 为粒度杀进程，配置 cgroup 级别的杀进程策略。

所以 oomd 在适用性和灵活性上仍有欠缺。

### kubernetes Node Swap

https://kubernetes.io/blog/2025/08/19/tuning-linux-swap-for-kubernetes-a-deep-dive/


#### Memory

Anonymous memory: This is memory that is not backed by a specific file on the disk, such as a program's heap and stack. From the application's perspective this is private memory, and when the kernel needs to reclaim these pages, it must write them to a dedicated swap device.

File-backed memory: This memory is backed by a file on a filesystem. This includes a program's executable code, shared libraries, and filesystem caches. When the kernel needs to reclaim these pages, it can simply discard them if they have not been modified ("clean"). If a page has been modified ("dirty"), the kernel must first write the changes back to the file before it can be discarded.



# 社区


# 工作

### 系统盘扩容
本周试了一下系统盘扩容，因为系统盘默认会存在 boot 分区，所以一个系统盘一般都是存在一个 boot分区，一个根目录分区，如果根目录分区是在最后一个分区的话扩容比较简单，但是如果不是最后一个分区的情况下，需要调整当前分区之后的所有分区的起始位置，非常复杂。综合考虑还是不建议用户将需要扩容的数据放到系统分区上面， 但是 gke 貌似是没有数据盘的， 回头确认下具体逻辑


