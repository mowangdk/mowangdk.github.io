---
layout: post
title:  "weekly report"
date:   2023-05-03 22:30:08 +0800
categories: weeklyreport
---

差不多有接近一个月没有写周报了, 除了五一假期之外, 主要是之前在写一篇关于 kubernetes 存储源码相关的文章, 这个耗费了两个周末的时间. 后续会把相关文档放出来. 

# 读书

最近两周在读图解网络硬件这本书, 其实这个对我的工作没有太多的帮助, 我也大概清楚,即使现在读完, 可能也会在未来某一天忘记. 不过出于买都买了的心理, 也就顺着读下去了, 怎么说呢? 确实书如其名, 该书详尽的介绍了网络硬件, 给我印象比较深的是三层交换机和路由器的区别. 

### 三层路由器
三层交换机是针对二层的一个加强, 跟路由器的主要区别就是他只支持 ip 这一个协议, 而路由器除了基本的 ip 协议之外, 还支持 IPX 和 AppleTalk 等协议. 简单来说三层就是特化版的路由器, 具有路由选择的功能, 基于硬件进行数据帧处理, 而不是 cpu. 并且三层交换机可以独立完成跨 VLAN 的通信. 二层交换机则需要路由器的配合才可以


其他的就很普通了

# 工作

### shell 能力

最近几年因为工作都是偏运维的工作, 所以或多或少都在接触shell, 但是每次写都要耗费很长时间来确认语法, 效率实在是不高. 但是见到有一些同事还是可以快速利用 shell 来快速获取一些信息的时候还是觉得很佩服, 比如如下命令就是过滤指定 pod 里面挂载的云盘. 下一个阶段就要学习一下 shell 了, 
```
for p in $(kubectl get po -A|grep xxx|awk '{print $2}');do echo "--- ${p} ---";  kubectl -n xxxxx exec -it ${p} -- lsblk 2>/dev/null;done
```

### dockershim

v1 版本的 dockershim 在已经创建出来之后依然会保持运行, 作为 container 的父进程 hold namespace 之类的信息, 并且是每一个 container 一个(v1 版本之前是每个 pod 一个, 具体原因需要再确认下)

# 社区

### csi-nvmf

最近在看 nvmf 这个开源项目, 据说是zijie 推进社区的, 但是目前来看进展比较缓慢, 需要持续跟踪

### kubernetes

最近提了两个 pr , 都被顺利的合并了, 虽然都不是什么大不了的功能. 不过也算是顺利的走向正轨了吧.

pr: 

https://github.com/kubernetes/kubernetes/pull/117595
https://github.com/kubernetes/kubernetes/pull/116277



