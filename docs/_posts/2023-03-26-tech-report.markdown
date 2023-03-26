---
layout: post
title:  "weekly report"
date:   2023-03-26 22:30:08 +0800
categories: weeklyreport
---

# 读书

### Linux 系统架构和应用技巧

本周因为周末要卖书, 所以打算把这一本书看完. 整个书算是专门为运维同学写的吧. linux 相关的只是有很多, 都是用 RHEL 演示的, 毕竟作者在 Red Hat 工作, 总之给我印象最深的是第一章, 然后剩下的就 emmm. 大家见仁见智吧

总体来说收获还是有一些的

- io调度(也就是磁盘磁头的运行策略, 使用 lsblk -t 获取): 当用户写盘的时候, 默认都是写在 pagecache 中的. 至于 pagecache 中的数据什么时候写盘, 这个就要看当前块存储指定的调度策略是什么了. 这与我之前的认知有一些出入. 也算是完善了一些知识图谱, 这个 scheduler 主要分以下几种, 
  - noop (no optimization) 只有一个子队列, 按照 bio 调度的顺序进行处理
  - cfq (Complete fair queuing) 根据 bio 对象的进程 id, 根据散列函数将请求分散到 64 个子队列中. 然后处理各个子队列中的请求
  - deadline 根据 bio 的相对位置做调度, 使得磁头移动最短
  - anticipatory, 在 deadline 调度器的基础上, 追加预测下一个 bio 对象的功能, 例如, 当进程 A 发出的读请求紧接着又收到了来之进程 B 的请求时, 该策略会预测进程 A 马上会发出另一个读请求, 这种情况下调度进程 B 的请求之前会延迟一段时间, 以等待来之进程 A 的下一个请求

- 内存模型部分主要是他介绍了虚拟内存和物理内存对应关系(尤其是内核空间), 在 x86 下,进程使用的虚拟内存被限制在 4GB  每个进程自己的虚拟内存的 3GB->4GB部分其实对应的是同一个物理内存的同一部分.也即是宿主机上的低端内存(1MB -> 896MB), 但是在 x86_64 中, 逻辑地址空间不限制为 4GB, 同样内核空间内存也不限制于 1MB -> 896MB.
- "cat /proc/meminfo"
    - Active(刚被使用的) = Active(anon) + Active(file)
	- Inactive(长时间未被使用的) = Inactive(anon) + Inactive(file)
    - 上面这两个分类说实话有些模糊
	- shmem(shared memory) 是 tmpfs 所使用的内存
	  - buffers + cached = Active(file) + Inactive(file) + shmem(shared memory)
	- anonpages 就是程序内使用的内存. 
	- 用户内存 = Active(file) + Inactive(file) + Active(anon) + inactive(anon) + unevictable = buffers + cached + anonpages

### 大话存储

这周没看, 下周补上

# 工作

### k8s dynamic client

这周遇到了一个问题, 在 k8s 集群中, 相关的 CRD 已经创建了, 但是在实际使用 dynamic client apply cr 的时候, 出现了资源找不到的报错. 定位流程如下
- 查看资源 groups version name 是否拼错(成功)
- 用组件日志将拼接出来的 yaml, 直接 apply 是否报错(成功)
- 相关 client 的权限是否 ready(ready)
- 查看集群的审计日志, 看看 yaml 是否跟本地一致(一致)

> 于是就产生了一个奇怪的现象, 同样的 yaml, 手动 apply 是成功的, 但是使用代码创建就失败. 不用说, 肯定是 package 的使用方式出现了问题

- 于是接下来, 将两次请求的审计日志分别 down 下来(一次成功, 一次失败) 使用 diff 查看, 结果发现 使用 dynamic 创建的资源在 yaml 外部缺少了 namespace 信息. 导致资源对应不上(资源本身是 namespace 级别) , 加上了 withNamespace 之后就成功了