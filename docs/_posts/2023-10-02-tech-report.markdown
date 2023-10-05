---
layout: post
title:  "weekly report"
date:   2023-10-02 22:30:08 +0800
categories: weeklyreport
---


# 读书

继续大话存储, 目前到了协议的融合, 协议之间通过调用, tunnel, map 三种方式互相融合形成了新的协议, 以满足不同场景. 不过大多数都是跟最普及的 ip/ethernet 协议融合. 这块可以说的倒是有很多, 但是目前接触的是在是太少了. 简单过一遍即可, 没必要详细研究, 主要是后面的章节 -- 虚拟化


### 虚拟化

计算机科学中的任何问题, 都可以通过加上一层逻辑来解决 -- david wheeler

可以说现代计算机的本身就是一层一层虚拟化, 一层一层抽象来共同组成的, 这一章主要探究的还是存储这部分的虚拟化.

带内虚拟化和带外虚拟化

这两个是新接触到的概念, 简单介绍下. 带内即是 inband, 所谓的控制指令和数据走的是同一条路线. 所谓的控制指令, 就是说用来控制实际数据流向和行为的数据. 这里举的例子就是 ip 网络, ip 网络的路由协议所产生的数据包和实际传输的数据是同一条链路. 就是带内的概念. 带外即是 outband 是控制指令和实际数据走的不一条路线. 控制指令通过单独的路线进行传输, 受到优待, 共路 和 随路 也是一样的意思

带内虚拟化指的是, 执行虚拟化的设备是横插在动作发起者和目标路径之前的设备, 动作发起者不感知虚拟化设备, 它能看到的只有目标设备, 只不过数据在流经虚拟化设备的时候就已经被虚拟化了

带外虚拟化, 是指在这个路径旁的另一条路径, 这条路走控制信号, 实际的数据传输还是通过发起者到目标设备. 也就是发起者必须感知旁路上的虚拟化设备, ,通过提示/授权之后, 才可以直接向目标请求数据

存储这部分虚拟化一开始介绍了各个公司的存储方案. 包括 IBM 的 SAN Volume Controller, 飞康的 NSS. 大概看了下,基本上都是一些池化的方案. 没啥特别的.
不过这边用的一些加速方案倒是可以参考下.

- SafeCache
    利用高性能介质,将主机下发的 io 直接写入高性能介质, 快速响应主机 io, 然后异步的将数据刷到主存储空间中
- HotZone
    是一种主动的数据访问优化方式, 将数据源划分成多个 zone, 对每个 zone 统计访问频繁程度. 最后将热点 zone 中的数据缓存到高速存储介质中以加速读访问
- Zero Memory Copy
    也就是数据不经过内存进行缓存/运算处理, 直接写入后端存储, 这样既提高了数据安全性, 还保证了性能. 飞康认为使用这种数据缓存是杯水车薪并且多此一举. 虚拟化设备后端本身已经挂接了拥有较大容量存储的存储控制器, 或者挂接各种新一代的闪存阵列产品作为全局缓存, 此时再加一点点缓存还不如不加



# 工作

### 简单整理了下 kubelet -> containerd -> containerd-shim -> runc 的调用流程.

```
kubelet.SyncPod()
-> kl.containerRuntime.SyncPod() (kubecontainer.Runtime)
-> kubeGenericRuntimeManager.SyncPod()
  -> m.createPodSandbox()
    ->  m.runtimeService.RunPodSandbox()
----containerd
        -> integration/cri-api/pkg/apis PodSandBoxManager.RunPodSandbox()
        -> (remote_runtime.go)runtimeService.RunPodSandBox()
          -> r.runtimeClient.RunPodSandbox(client)
          -> instrumentedService.RunPodSandbox()
          -> criService.RunPodSandbox()
            -> ociRuntime := c.getSandBoxRuntime() (save in containerd register memory, use name to select runtime)
            -> sandbox := netns.NewNetNS(), c.setupPodNetwork(ctx, &sandbox)
              	-> call cni setup interface
            -> c.client.TaskService().Create(ctx, request)
-----shim
            -> runtime.shim.v1.service.Create()
               -> process.Newinit()
               -> process.Create()
                  -> oci.runtime.Create()
-----runc
                    -> runc create/start xxxx command

  -> start() start ephemeral containers -> start init containers -> start containers
     -> m.startContainer(podSandboxID, podspec...) // 1. pull the image, 2. create the container, 3. start the container 4. run the post start lifecycle hooks
       -> m.internalLifecycle.PreCreateContainer()
       -> m.runtimeService.CreateContainer(podSandboxID)
          -> instrumentedService.CreateContainer()
          -> criService.RunPodSandbox()
            -> xxx
       -> m.internalLifecycle.PreStartContainer(pod, container, containerID)
       -> m.runtimeService.StartContainer(ctx, containerID)
          -> criService.StartContainer()
          -> same
       -> runner.Run() // post start hook

```

### mount remount-ro

这两天遇到了一个 remount-ro 的问题, 在 mount 快设备的时候, 可以指定 mount 对应的 options , 这个 options 的作用是, 当背后的块设备不可达的时候, 这个挂载点会被重新 remount 成只读的. 讲真, 之前从来没注意过这个功能. 需要试试

