---
layout: post
title:  "Kubernetes csi volume usages"
date:   2022-05-28 22:30:08 +0800
categories: blog
---


### 背景

目前 Kubernetes 社区主推使用 CSI 存储组件进行存储卷的挂载和使用. 然而这里面的流程却不止只有 CSI 组件参与, 这里面的流程十分复杂, 并且由于 Kubernetes 面向终态的设计理念, 这里面有很多信息都是异步传递的. 这理解整个存储卷的挂载卸载逻辑造成了不小的障碍,今天用一Kubernetes


### 存储卷创建

在 Kubernetes 集群中创建存储卷一般有两种方式
- 关键已经存在的存储卷
- 通过 Kubernetes 组件调用相关 openapi 来自动创建存储卷

第一种十分简单, 只要将对应的存储资源标识符记下来, 在创建 pv 资源的时候填入即可(相应的处理 Driver 需要提前确认). pvc 资源正常创建即可.

使用第二种方式的前提条件则是 pvc 声明的存储 Driver 支持动态创建这种模式(实现了相关的 api 调用), 否则是无法使用的, 使用流程如下, 你在集群中声明了 pvc, 并且在 pvc 中声明已经存在的 storageclass, 这个 storageclass 是创建存储使用的模板, 它声明了你要创建存储的类型, 创建时使用的参数, 使用的存储 Driver. 这种情况下, 一旦 pvc 在集群中创建, Kubernetes 管控端的组件 pv-controller 就会检查相关的参数, 根据参数找到集群中已注册的存储 Drvier. 调用 external-provisioner 项目中的 ```Provision``` 代码进行存储卷的创建(csi 逻辑), Provision 方法则会调用 CSI 中 相应的 ```CreateVolume()``` 实施真正的创建. 可以看出整个流程涉及到了三个组件, pv-controller, external-provisioner, csi-plugin. 

### 存储卷的绑定

总所周知, 只有 PVC/PV 处于 Bound 状态的时候才能够被 pod 使用. 而这个状态完全是由 pv-controller 这个组件控制的. 但是即便如此, 这个更新流程也不能够算得上十分简单. 因为这个绑定状态是 PVC/PV 这两个资源共同声明才生效的, pv-controller 有两个 goroutine, 一个 goroutine 专门负责 PVC 的状态变更, 一个 goroutine 则检查 PV 的状态变更, 只有两边的状态同时达到 Bound.  pv-controller 才会更新 etcd 的数据声明这两个资源为绑定状态, 这其中要数次更新 pvc/pv 的资源, 由于 goroutine 的执行循序是不确定的, 这就导致可能出现版本冲突的情况. 造成重试

> Kubernetes 为了避免资源覆盖, 是根据版本来进行更新的. 每次更新请求都会带着 client 缓存的资源版本上报, 一旦这个版本与 server 端不一致则更新失败(说明有其他的客户端修改了资源)


### 存储卷的使用

![csi mount costs](/assets/img/kubelet-volumes.png)

上图列出了整个存储卷的使用流程, 我现在按照顺序说明下
- 当 pod 被调度到某个节点的时候, 并且这个 pod 引用了某个已经 Bound 的存储卷, 这时就会检查 volume 对应的 csidriver 的 ATTACHREQUIRED 字段, 如果为 true, 则由 attach-detach-controller(kcm) 在集群中创建一个 VolumeAttachment 对象, 这个对象用于记录一个存储卷的 Attach 状态, 如果为 false 则直接跳过 attach 阶段
- CSI 组件中的 external-attacher 监听集群中的 VolumeAttachment 对象, 一旦有新的对象创建, 开始调用 csi-provisioner 组件处理
- csi-provisioner 组件暴露 ```ControllerPublishVolume``` 方法供 external-attacher sidecar 组件调用, 这个方法是我们自己编写的
- ```ControllerPublishVolume``` 方法将块设备(一般需要 attach 动作的只有块设备) 挂载到所在目标节点上. 这个过程就是将底层的块设备转化成 /dev/vd* 设备符. 一般 nfs 类型的存储都无需挂载这一步
- ```ControllerPublishVolume``` 顺利返回后, external-attacher 组件调用 kubeapi 方法将 VolumeAttachment 的 volumeAttachment 设置为 true
- 自从这个 VolumeAttachment 被创建的之后, attachdetachcontroller 就一直监听这个对象, 看它什么时候被挂载上, 一旦这个资源声明 volumeAttached 为 true. attachdetachcontroller 就开始继续处理挂载逻辑
- 所谓的处理逻辑就是为这个 pod 所在的 Node 节点打上标签, 打上标签之后, 这个 Node 节点的 Status 就会有如下字段, 这一步之后, 管控端, 也就是 kcm 就完成了它的基本逻辑.

```yaml
  volumesAttached:  
  - devicePath: ""    
    name: kubernetes.io/csi/<driver-name>^<volume-id>
```

- 这个 Node 上的 kubelet 会一直轮询 (default 10s) 所有调度到这个 Node 节点的 Pod 的 volume 状态(不止 volume, 还有其他资源的状态), 因为只有 pod 的 volume 状态是已挂载的情况下, kubelet 才能继续执行 pullimage & start runtime container 的流程. kubelet 轮训的就是 volumeAttached 这个字段, 一旦为 true, kubelet 就开始更新 Node 节点的状态, 挂载的情况下就是为 Node.Status 添加如下字段(当然, 也同样处理其他字段)

```yaml
  volumesInUse:  
  - kubernetes.io/csi/diskplugin.csi.alibabacloud.com^d-2ze8plgq2y0c4myjkf8l
```

> 以上两个 Node 节点上的资源只适用于 csidriver AttachRequired 字段为 true 的存储, 其他类型存储都不会出现在 node 的 status 字段里面

- 自从 pod 被调度到这个 Node 上之后(1.24 之前, 1.24 以及之后这个改成了 volume 被 attach 之后才开始检查,算是一个小优化了), kubelet 就会启动一个 goroutine 来轮询 pod volume 的状态, 也即是检查上面的 volumeInUse 字段. 一旦这个字段出现了 pod 声明的 volume, kubelet 就会开始调用节点上的 csi-plugin 组件开始 mount 的操作. 主要是```NodeStageVolume```方法和 ```NodePublishVolume```方法, 最后的挂载路径是

```shell
/dev/vdb on /var/lib/kubelet/plugins/kubernetes.io/csi/pv/<pv-name>/globalmount type ext4 (rw,relatime)
/dev/vdb on /var/lib/kubelet/pods/<pod-uid>/volumes/kubernetes.io~csi/<pv-name>/mount type ext4 (rw,relatime)
```

- 上一步检查这个步骤, 同样只适用于 csidriver AttachRequired 字段为 true 的存储, 如果是非 true 的话则会直接调用 ```NodePublishVolume``` 字段直接将存储挂载到 pod 内部, 不必走上述流程挂载完成后, 比如如下 nfs 存储

```shell
xxxx:/xxxx on /var/lib/kubelet/pods/<pod-uid>/volumes/kubernetes.io~csi/<pv-name>/mount type nfs 
```

### 总结

以上就是存储的整个一个流程, 本文主要理清了 volume 的状态在不同组件中的流转, 可以看到 块设备类型的存储的流程要比网络类型存储的流程复杂的多. 灵活得多