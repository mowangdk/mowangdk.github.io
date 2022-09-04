---
layout: post
title:  "weekly report"
date:   2022-09-04 22:30:08 +0800
categories: weeklyreport
---

# 读书

### 论文

https://www.usenix.org/conference/osdi22/presentation/zhong

论文这周没有什么进展, 摸鱼状态

# 日常

###  pod deleted failed device busy

这个问题算是老生长谈了. 因为总是出现,所以再提一嘴, 原因是因为有其他进程在使用 pod 关联的容器进程. 使用 fuser -m /dev/vdx, 或者使用 lsof 查看相关进程 pid kill 掉即可

### Z 进程

这周遇到了一个问题, 发现无法删除某一个节点上面的 pod. 相关报错为 ```The container could not be located when the pod was deleted.  The container used to be Running```. 详细看过主机进程状态, pod 相关进程处于 Z 状态, 无法回收. 获取 Z 进程 pid. 继续看 /proc/<pid> 下面的信息, 使用 ls -l task/*/fd 查看当前进程是否有打开的 fd. 导致进程无法被回收处于 Z 状态. 经确认确实存在一个内核级别的问题导致相关进程删除失败.


### CSI GlobalMount 问题

这周才发现, 自 kubernetes 1.24 之后, 社区更改了 GlobalMount 的路径 pattern. 这意味着之前我们在 csi 中所有的 custom logic 都要随之更改. 问了下社区的人, 他们说我们不应该使用这个路径, 这个路径他们会随时更改. 但是很多功能的确依赖这个路径. 很难完全绕开.这就必须要修改所有依赖这个路径的地方. 的确是个大工程, csi 的其他地方也要修改.