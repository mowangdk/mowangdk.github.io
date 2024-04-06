---
layout: post
title:  "node graceful shutdown 相关整理"
date:   2024-04-05 22:30:08 +0800
categories: weeklyreport
---

在上周的会议上突然谈论起来带有持久化应用的 node 抢占问题, 这里统一整理一下, 首先有三个已有的 issue, 先简单看下. 

### node graceful shutdown
第一个是 msau 在去年提的 https://github.com/kubernetes/kubernetes/issues/115149. issue 里面主要是描述了下两种现象, 
1. 当 csi 组件先于应用 volume 之前被删除的时候, 就会造成 VolumeInUse 字段没有被处理的情况, 这种情况下, 使用 systemNodeCritial 声明 csi 并没有什么用, 因为即使是 systemNodeCritial 也不会等待 csi 卸载完所有应用的 pod, 所以还是有问题
2. 还有就是即使我们在节点上配置了 shutdownGracePeriod,  当宿主机突然关闭的情况下, 我们同样也没办法卸载所有的 pod, 导致这些 pod 处于 terminating 的状态. 

关于 shutdownGracePeriod 这里贴一段原文说明: 
For example, if ShutdownGracePeriod=30s, and ShutdownGracePeriodCriticalPods=10s, kubelet will delay the node shutdown by 30 seconds. During this time, the first 20 seconds (30-10) would be reserved for gracefully terminating normal pods, and the last 10 seconds would be reserved for terminating critical pods.

这个配置是 kubelet 监听 PrepareForShutdown logind event 得到的. 详情可以参考 https://www.freedesktop.org/wiki/Software/systemd/inhibit/ 

第一个问题应该没什么可以说的, 第二个问题怎么看都像是个 bug. 还需要继续确认下, 果然, 它这边详细追踪了下第二种情况下的代码. 这里我也简单说明下


#### shutdown-grace-period

1. 首先可以确认的是, 抑制释放锁的逻辑是在非预期的情况下被释放掉了. 所以导致了宿主机意外关机, 代码是在

```golang
// kubernetes/pkg/kubelet/nodeshutdown/nodeshutdown_manager_linux.go L326
m.dbusCon.ReleaseInhibitLock(m.inhibitLock) 
```

2. 从这里继续往上追, 我们要等到所有的 pod 都顺利被 kill 掉之后才会调用上述的方法.
```golang
// 这里调用的是 pod_workers.go 里面的方法, 也是复用 evictPod 调用的方法
killPodNow(podWorkers PodWorkers, recorder record.EventRecorder) eviction.KillPodFunc
```

3. 在 ```killPodNow``` 里面调用了 updatePod, 在 updatePod 里面会检查 pod 是否为 terminated 状态, 如果是 terminated 状态, 才会更新 ch,进而标记 pod 已处理完成

```golang
close(ch)
```

4. IsTerminated 状态是依赖 pod 上的 terminatedAt, 这个属性在所有容器都停止运行时被设置
```
!s.terminatedAt.IsZero()
```

所以, 这里就是问题, 当pod 被打上 terminatedAt 的标签的时候, 并不代表所有的 pod 使用的云盘都被清理掉了. 导致了上述的问题, kubelet podmanager 也是通过 IsTerminated 状态来判断是否需要处理的. 这里会有并发的问题, msau 还贴心的帮忙画了一张图

![volume manager](https://user-images.githubusercontent.com/24448061/213246295-af0c53e0-e443-4da8-87ff-60dd794cc6be.png)

这里 msau 提供了两个可能得解决方案, 一个是在上述的 gracefulsetdown 里面加上清理的相关逻辑, 另一个就是在 csidriver 里面自己实现上述gracefulnodeshutdown 的逻辑 

mauricio 特别提了一下, csi prehook 之类的方案是不行的, 因为这里会在 csi 升级/卸载的时候也会有类似的问题. 所以不予考虑

sftim 同时也提出了, kubelet 是否可以为 gracefulshutdown 提供额外的上下文, 来帮助 csi 等 daemonset 来做出判断, 他用了一个 /etc/nologin 的标志来说明, 节点处于关机状态时, 登录系统可以检测这个标记位来禁止新用户登录. 同样, 他也提出了, 除了存储之外, 是否还有其他的 external device 也会遇到同样的问题, 比如网络或者其他外设

mauricio 同时也提出了将 pod 分组的逻辑, 我们可以将工作负载配置为一个组，让节点关机管理器在终止它们时等待相应的关机宽限期。而像CSI驱动程序这样的关键工作负载将被安排在不同的组里稍后终止。这样，我们就给了CSI驱动程序完成卷拆卸CSI调用的机会


基于上面 的做法, mauricio 提了一个 pr: https://github.com/kubernetes/kubernetes/pull/120091, 简单看了下, 基本上是利用现有的 priority group 来实现的, 我个人觉得是没啥问题. 看看后续是怎么个状态吧

这个问题也在上周的周会上讨论了一下, 结论就是先将 mauricio 的 pr 给 sig-node 的同学一起 review 下, 后面是否需要架构性的变更则需要再提新的 pr 来解决



