---
layout: post
title:  "weekly report"
date:   2025-04-07 22:30:08 +0800
categories: weeklyreport
---


# 读书

# 社区

最近看了几个pr，这里简单梳理下，有些 pr 还没有合并，需要持续追踪


### CSINode auto removal

https://github.com/kubernetes/kubernetes/pull/131098

这个pr 说的是 CSINode 上有一个OwnerReference, 关联到 node 上, 当 CSINode 上的 nodeid 和 node 上的 id 不匹配的时候， 会自动删除这个 CSINode，这个逻辑本身没问题， 但是一旦 CSINode 被 GCed 之后，kubelet 就会重新注册好 CSInode, 所以实际上这个问题可能也不会引起太大的关注




# 工作

### mmap 和 directio

directio 是基于文件系统来说的， 不同的文件系统的特性不同。总之是绕过 pagecache（同样基于文件系统的概念） 的一种方式
但是 mmap 其实不会基于文件系统，尽管同pagecache 一样也是将数据写到内存里面，但是不依赖于文件系统的依赖， 不过当然也有一些缺点，比如需要自己管理声明周期等

### proc process stack

突然跟同事聊起了 proc 下面的stack是怎么获取的进程调用链路，因为这个stack 肯定是某一个cpu核在某个时刻运行的上下文，如果进程出现了卡顿，那么某个核多半会一直卡在某个stack这时候通过 proc 下面的虚拟文件去读取的话就会很容易排查问题，但是如果是多 thread 的情况下， 进程 pid 里面的stack指的是什么，拆了下多半是进程主线程的stack，剩余所有子线程都是在 /proc/pid/task 里面跟踪


### containerd tty 泄露

```container 配置
    stdin: true
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    tty: true
```
issue: https://github.com/containerd/containerd/issues/11160

存在泄露问题


### 查询进程被kill的关联信息

```
bpftrace -e 'tracepoint:syscalls:sys_enter_kill {if(args->sig==9){printf("process %s:%d send signal%d to pid %d\n", comm, pid, args->sig, args->pid); }}' 
```