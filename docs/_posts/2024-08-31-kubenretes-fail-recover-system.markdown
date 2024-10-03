---
layout: post
title:  "Kubernetes fail recover system about volume"
date:   2024-08-21 22:30:08 +0800
categories: weeklyreport
---


## 背景

kubernetes 设计了一系列故障恢复的机制。本文主要是为了探究这些机制的原理，验证这些机制的触发条件，以及探索在基于社区的实现基础上我们的容器存储故障恢复流程中有什么可以进一步可以完善的方向

本文基于 Kubernetes main branch， commitid： eebc897e4fd8bf26a69d322c8dbcdf4da475934e

所谓的故障恢复， 就是当 apiserver / 节点 出现故障的时候， kubernetes 内部的组件如何通过内部机制保证用户的应用不受影响。可能采取的行为包括但不限于
- 强制删除， 在另一个节点启动
- 不断重试， 保证当依赖组件/服务正常的时候可以立即恢复

对于存储的故障，很大一部分场景集中在用户的应用(也就是 pod) 的删除、升级、迁移这几个动作中，这几个动作一般涉及 volume 的 mount， umount， attach， detach. 至于存储 create 和 delete 一般独立于用户的 pod 的声明周期之外, 不会对用户造成显著影响

对于 mount， umount， attach， detach 这几个动作涉及到 Kubernetes 社区本身的代码， 以及各个云厂商所编写的 csi 的代码。 今天我们主要讨论 Kubernetes 本身的一些机制，各个厂商由于都是由于都是按照 CSI 协议实现的标准代码，该协议定义了与 Kubernetes 的交互接口，相关接口的调用都是由 Kubernetes 触发。CSI 这里本身不涉及故障恢复的逻辑。整体的控制流还是基于 Kubernetes 本身的流程轮转。

> 本文中所有方案都是基于云存储的方案，本地存储及本地临时盘原则上不能存储高可用级别数据，无迁移方案

## 分类

这里根据存储类型进行分类

- 支持多节点挂载, 存储本身支持在多个节点上同时访问。 这种典型的存储有 nas，oss，cpfs。 这种类型的存储最佳故障恢复方式就是不管当前节点故障， 立即在新节点拉起 pod，因为访问的是同一个存储，所以不存在数据迁移等问题。 但是存在一定的应用依赖， 就是必须在上层隔离故障节点的流量/数据写入， 避免出现数据损坏。

- 只支持单节点挂载, 存储只支持在一个节点上访问。 典型的存储就是大部分的云盘存储。这种一旦出现节点故障， 就需要先将 pod 迁移到其他节点， 然后再重新挂载。


在故障恢复流程中，我们主要关注只支持单节点挂载的云盘。 因为多节点挂载的存储一般都可以通过直接在新节点上挂载的方式进行故障恢复。这种方式相对比较容易，后面再总结的时候会简单进行故障恢复流程说明，这里不详细展开， 而单节点挂载的云盘也是对 Kubernetes 故障恢复体系依赖程度最高的。 需要涉及 Kubernetes 的控制平面和数据平面组件的配合。 涉及到上面提到的 mount， umount， attach， detach 的全流程。 我们重点关注这里的逻辑。


## 单节点云盘挂载正常流程

![csi mount costs](/assets/img/volume_schedule.jpg)

整体来说 pod 启动，也就是云盘挂载需要分成两个部分。 
1. 管控侧调用 openapi 将块设备 attach 到节点上
2. 对应图中的第 8 步
3. 节点测对块设备进行初始化， 包括格式化特定的文件系统，然后将块设备 mount 到特定的路径上
4. 对应图中的第 16 步和第 17 步

同样 pod 删除， 对应的云盘卸载，需要先在节点上 umount 对应的路径， 以及调用 openapi detach 接口， 将云盘从对应节点上 detach

回到故障恢复，如果节点故障，导致某一个步骤没有顺利完成，就会造成卸载失败，有时会导致 pod 处于 terminating 状态, 有时当前pod 可以删除， 但是使用这块盘的新的 pod 会一直处于 ContainerCreating 状态

下面我们就来简单分析下 Kubenretes 内部的故障恢复流程， 如果对这方面没兴趣的同学可以直接跳到结论

## 控制平面

Kubernetes 主要分为控制平面和数据平面， 相关故障恢复的代码也同样存在于两边， 我们先从控制平面看起. 说到控制平面主要就是 kube-controller-manager 这个组件，他是负责维护 pod 和 可挂载 volume 生命周期的， 今天我们主要聊故障恢复 是负责 volume attach，detach 流程的重要管理工具, 也是跟节点或者可用区故障最相关的两个组件，下面我们简单看下几个基本概念和 kube-controller-manager 的基本配置

#### forceDetachTimeoutExpired 

当 kube-controller-manager 的 disableForceDetachOnTimeout 没有开启的时候（默认不开启, 运维人员可以通过手动修改kube-controller-manager 的参数来禁止这个功能），意味着 kube-controller-manager 默认就可以在 pod 强制删除之后超时的时候强制卸载.

> kube-controller-manager 相关参数： disable-force-detach-on-timeout

超时的定义在 maxWaitForUnmountDuration 在 kube-controller-manager 中默认定义为 6min, 超过这个时间, 管控侧就认为 node 或者是 kubelet 已经无返回了。开始强制 Detach 存储

> 社区计划于 1.32 开始将 disableForceDetachOnTimeout 设置为默认开启

#### NodeUnhealth

当 node 节点的状态为 NotReady 的时候， 这个 node 就会被标记为不健康, ```isHealth == false```

#### NodeOutOfService

当 node 被手动打上了这个 ```node.kubernetes.io/out-of-service``` taints 的时候， 代表是人工手动认为该节点已经无法提供服务了。


### Kubernetes kube-controller-manager 存储故障处理逻辑 

整理下 kube-controller-manager 的整体逻辑

1. 当人工打了 ```node.kubernetes.io/out-of-service``` 会立即进行卸载， 不会检测 node 是否存在, 和节点侧的逻辑是否完成。会立即调用相关组件的 Detach 接口
    a. 当 Detach 成功的时候(对于 CSI 组件来说也就是删除 VolumeAttachments 成功, 不代表实际 Detach 成功）， kube-controller-manager 会打上相关的指标， 标记当前 volume 执行了强制逻辑， 非正常逻辑
    b. 当 Detach 不成功的时候, 相关 volume 会被加回去， 等待再次重试

2. 如果当前节点出现了 NotReady 的情况, 这时 pod 是无法被正常删掉的。 需要使用 ```kubectl delete -f <pod-name>``` 命令强制删除。 这样，attach detach controller 会在 6min(default) 之后强制将盘卸载, 当然，节点上依旧会有残留， 当节点恢复的时候， 同样的 volume 挂载上去可能会有问题。 并且不能保证存储组件的Detach一定可以成功.

> AttachDetachController 中节点侧的 Detach 逻辑（调用CSI）是否开始  是依赖 ```node.Status.VolumeInUse``` 字段， 这个字段在正常情况下完全由节点侧控制。 kube-controller-manager 只会对这个节点的状态进行同步



## 数据平面

关于数据平面， 一般情况下，节点的故障恢复逻辑一般是， 通过 kubectl drain 排水掉所有的 pod。并在当前节点打标。 如果 kubelet 正常的情况下， 它会在节点上执行排水流程， 驱逐pod。 但是存在一些情况 我们未能驱逐节点上的pod， 或者kubelet 和管控的网络断掉了等情况。 kubernetes 社区设计了 graceful node shutdown 和 non-graceful node shutdown 这里目前针对持久化 volume 还有一些问题， 我们稍后再说。

### kubelet graceful shutdown

kubelet graceful shutdown 是在 kubelet 1.21 版本中引入的（至今没有GA）， 使用 ```GracefulNodeShutdown``` featuregate 默认开启。这个功能简单就是说我们可以在 node shutdown 的时候对 pod 进行优雅关闭。当然这个功能本身 kubelet 是不具备的， 它是借用了 [systemd inhibiter locks](https://www.freedesktop.org/wiki/Software/systemd/inhibit/) 来实现的。 

除了上面的开关， graceful shutdown 依赖如下两个配置

- shutdownGracePeriod ： 指定了 kubelet 延迟 node 关机的总耗时。 也是当前节点 pod termination 的总耗时。 默认 0s, 也就是不开启. 设置值的时候必须大于 1s
- shutdownGracePeriodCriticalPods: 制定了节点关闭 critical pod 所需的时间。 必须小于 shutdownGracePeriod, 否则会被组件拦截，报错配置有误。 默认 0s, 也就是不开启, 同样必须大于 1s

当 systemd 检测到或者被通知系统将要关机的时候， kubelet 会将 NodeReady 的 Condition 标记到 node 上， 并且当 Condition 的 Reason 被设置为 "node is shutting down" 的时候， scheudler 也会识别这个 Condition， 开始禁止 pod 调度到当前 node 节点上。即使这个 pod 存在 toleration ```node.kubernetes.io/not-ready:NoSchedule```

> 存在一个极端情况是， 管理员手动取消了 node 的关机， 这种情况下， node 节点会正常处于 Ready 状态， 但是之前被 terminated 的 pod 就没办法恢复了， 只能等待 pod 再次被调度回这个节点（这个与我们待会讲的目前存在的问题有关）

在此之上还有一个增强， 支持用户自定义配置 pod 的删除顺序。 ```GracefulNodeShutdownBasedOnPodPriority``` 的 featuregate， 它本身也是要依赖上面两个配置的， 除此之外还有一个叫做 ShutdownGracePeriodByPodPriority 的配置。他和前面的 featuregate 是绑定的

其中 ShutdownGracePeriodByPodPriority 可以通过以下方式配置， 这里不详述了, 其中 prriority' 是 pod 的 priority classes， shutdownGracePeriodSeconds 是 shutdown 的时间

```
shutdownGracePeriodByPodPriority:
  - priority: 100000
    shutdownGracePeriodSeconds: 10
  - priority: 10000
    shutdownGracePeriodSeconds: 180
  - priority: 1000
    shutdownGracePeriodSeconds: 120
  - priority: 0
    shutdownGracePeriodSeconds: 60
```


#### graceful node shutdown 的问题

尽管我们已经有了这些配置， 但是 graceful shutdown 的实现还是存在一些问题。主要就是有状态应用的 graceful shutdown 的实现问题。这里我们详细追一下

上面 inhibiter locks 会在开始处理 pod 之前被获取， 释放锁的逻辑是等待Pod在Grace期內终止。， 这样操作系统就可以继续执行关机流程了。

释放逻辑： 
```
 m.dbusCon.ReleaseInhibitLock(m.inhibitLock) 
```

这里释放逻辑的判断条件是 ```IsTerminated```逻辑执行的结果，而 ```IsTerminated``` 的逻辑则依赖 ```terminatedAt``` 这个字段， 这个字段会在所有的 container 被 stop 之后设置。

```

func (s *podSyncStatus) IsTerminated() bool           { return !s.terminatedAt.IsZero() }
```

这里就是问题出现的关键点， 所有的 pod 被删除并不代表这个 pod 上的 volume 被删除了， 正常来说， volume 卸载的流程就是在 所有的 pod 被删除之后。

```
 if !dswp.podStateProvider.ShouldPodRuntimeBeRemoved(volumeToMount.Pod.UID) { 
```

那么， 当普通 pod 删除之后就会开始 critical pod 的删除流程， 但是这个时候 volume 的卸载流程也在同步进行， 所以这里就存在问题， 当存储插件先于 pod volume 卸载之前被删除的话。 pod 所关联的 volume 就会残留在这个节点上， 造成异常状况。

这里的异常情况指的是

- 原本可以走 graceful nodeshutdown 流程删除的 pod, 需要依赖 non-graceful nodeshutdown 流程才能删除
- 节点上未走正常流程删除的pod 会残留挂载路径在节点上。当同一个 volume 再次调度到当前节点上的时候， 会出现挂载不上的异常

针对这个问题， 目前社区也在修复过程中， 具体修复方式就是将 volumeManager 对象引入到 nodeshutdown manager 中， 这样当 pod 删除之后，我们可以同步等待 volumeManager 对象的删除流程完成， 再走后面的流程。

详情请见： https://github.com/kubernetes/kubernetes/pull/125070/files

### kubelet non-graceful shutdown

当然，上面的步骤并不是万无一失的， 当节点侧没有执行到清理流程的时候， pod 就无法被正确的回收， 尤其对于有状态应用， 因为原先的 pod 删除不掉会影响新 pod 的创建。 因为我们的节点处于完全失联/部分失能的状态， 所以我们无法通过上面的graceful shutdown 来处理。这是只能通过上面手动打标 ```node.kubernetes.io/out-of-service``` taint 来处理了。打了这个标记的节点， 新的 pod 无法被立即调度到这个节点。 当删除这个节点上的 pod 的时候，kcm 会立即调用 DetachVolume 进行卸载 