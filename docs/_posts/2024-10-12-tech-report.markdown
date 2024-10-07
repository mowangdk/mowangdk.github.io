---
layout: post
title:  "weekly report"
date:   2024-10-12 22:30:08 +0800
categories: weeklyreport
---


# 读书

### auditd

auditd 是 linux 审计系统的用户空间组件， 他负责把审计写在系统盘上。并且可以通过 ausearch or aureport 工具完成查看， 通过 auditctl 来配置或者是加载规则到内核， 此外还有一个 augenrules 程序， 读取位于 /etc/audit/rules.d/ 的规则， 并把它编译成 audit.rules 文件。为什么突然搞这个，是因为突然发现节点上 csi 进行卸载云盘动作的时候过于缓慢，导致整体出现异常。如果能监听某个目录的操作的话，并且记录每个操作的耗时，应该能更容易定位问题。


在 /etc/audit/rules.d/audit.rules 配置

```
-a always,exit -F arch=b64 -S mount -k block_device_mount
```
这样，原则上就可以抓到 mount 时候的日志, 使用如下命令抓取

```
sudo ausearch -k block_device_mount
```
最后发现确实抓到了一些数据， 不过并没有时长相关的数据，比较麻烦


### vnc

vnc 由客户端，服务端和一个简单的协议构成，服务端将屏幕的帧缓存直接发送给客户端， 客户端将io输入发送给服务端。从而完成的交互， 该技术因为不依赖操作系统，纯粹的底层交互，所以经常被用于故障，以及系统装机等情况（这时候sshd还不存在， 甚至操作系统都不存在） 但是另一方面， 由于是把帧数据进行发送， 导致在实际传输的时候使用的流量会比较大。进而衍生出了很多的压缩编码。



### 可重复构建

简单来说就是同一份source源码在任何地方构建得出的制品都是完全一样的，主要是为了安全。详细可以参考

https://reproducible-builds.org/docs/commandments/

这里列举了一些基本要求，基本上如果你满足了这些需求， 就达到了可重复构建的标准.


# 开源

# 工作

### exec format error

一个神奇的问题， 其实之前也遇到过类似的， 问题是在容器里面运行二进制的时候出现了 exec format error 的错误，已经检查过节点架构和镜像架构是一致的， 但是还是报错. 容器因为二进制启动失败从而crash。以上的现象初步说明容器镜像本身可能没问题， 否则不可能拉起来然后开始执行二进制， 出问题的是二进制， 二进制可能由于什么原因损坏了，导致报这个错误

#### bdf

bdf 是由虚拟化侧负责生成的， 在热插入的时候会将 bdf 与 块存储的绑定关系返回给管控， 当重启的时候管控会将bdf 重新下发给虚拟化。 每台 ecs 的起始 bdf 不一定相同， 依靠虚拟化的策略 


#### 节点上没有daemonset 的 pod

daemon 正常部署， 但是特定节点上没有具体的 pod  查看 daemon 的 event 发现有如下问题：

```
Warning  FailedCreate  9m45s (x224 over 15h)  daemonset-controller  Error creating: admission webhook "mutating-pods.alibabacloud.com" denied the request: plugin SecurityEnhancement error: plugin SecurityEnhancement admit for kube-system/ failed: waitToken timeout
``` 

升级 openkruise 后解决

### pod 启动失败， no such file or directory

```
failed to write 496794: openat2 /sys/fs/cgroup/systemd/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-podxxxxxx.slice/cri-containerd-xxxxx.scope/cgroup.procs: no such file or directory: unknown
```

container在start的过程中，遇到了systemd组件做了一次 daemon-reload的动作，然后导致刚刚由runc创建的cgroup目录被systemd给清理了。是一个低概率的问题, 接下来的一个问题是 systemd-reload 为什么会触发文件删除？


### no relationship found between node '%s' and this object

k8s 管控会将所有 kubelet（单个 node） 需要访问的资源在内存中绘制成一个 graph， 当相关资源被创建出来的时候， 会自动在 graph 里面添加对应的 edge， 当kubelet 实际访问这个对象的时候，会查询这个对象是否在 graph 里面， 如果没有则会报标题所示错误， 但是问题是这个错误里面同样包含尚未在集群里面被创建的对象。这时同样会报 no relationship 的错误，但是其实跟权限无关。同样，因为这个 error 没有返回， 所以也不知道具体是什么问题导致的报错, 如要判断根因的话只能去管控侧去翻日志了。


```golang

	ok, err := r.hasPathFrom(nodeName, startingType, attrs.GetNamespace(), attrs.GetName())
	if err != nil {
		klog.V(2).InfoS("NODE DENY", "err", err)
		return authorizer.DecisionNoOpinion, fmt.Sprintf("no relationship found between node '%s' and this object", nodeName), nil
	}
	if !ok {
		klog.V(2).Infof("NODE DENY: '%s' %#v", nodeName, attrs)
		return authorizer.DecisionNoOpinion, fmt.Sprintf("no relationship found between node '%s' and this object", nodeName), nil
	}

```