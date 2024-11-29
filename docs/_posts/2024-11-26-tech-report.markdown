---
layout: post
title:  "weekly report"
date:   2024-11-26 22:30:08 +0800
categories: weeklyreport
---


# 读书

### v2 版本

go仓库打v2的tag开始要改import路径。。否则未来如果有从其他仓库import它会有问题


### gotoolchain
https://go.dev/doc/toolchain


普通情况举个例子， 在使用Go 1.21.3版本自带的go命令时，如果你的主模块在go.mod文件中指定的版本是go line 是 1.21.0，那么go命令就会以 1.21.3 命令运行. toolchain情况举个例子，当使用的自带的Go 1.21.3 版本自带的 go 命令的时候，如果在主模块的 go.mod 文件中指定的版本是 go/toolchain 声明为 1.21.9, go command就会找 1.21.9 这个版本去运行，首先会去PATH 路径去找，否则的话就会直接下载并缓存一份 Go1.21.9 的 toolchain. 当然，这个配置可以禁用。

其他模块的依赖模块可能需要设置一个低于首选工具链的最低Go版本要求，这是为了在直接操作该模块时能够兼容。在这种情况下，go.mod或go.work文件中的工具链行设定了一个优先级更高的工具链，当go命令决定使用哪个工具链时，它会优先于go行中设定的版本。

例如，设置GOTOOLCHAIN=go1.21.3+auto会让go命令默认使用Go 1.21.3版本，但仍然会根据go和toolchain指令使用更加新的工具链。因为默认的GOTOOLCHAIN设置可以通过go env -w来改变，所以如果你已经安装了Go 1.21.0或更高版本，那么可以灵活调整。

"go"行声明了使用模块或工作区所需的最小Go版本。出于兼容性考虑，如果在go.mod文件中省略了"go"行，那么模块被认为隐含地要求Go 1.16版本；如果在go.work文件中省略了"go"行，工作区则被认为隐含地要求Go 1.18版本。同样， gotoolchain 也有默认的版本， 当显示声明 gotoolchain 的版本大于default toolchain版本的时候， 默认使用显式声明的版本。默认 gotoolchain 版本跟 go行声明的版本一致

模块的 go 行必须声明一个版本，该版本大于或等于 require 语句中列出的每个模块声明的 go 版本。工作区的 go 行必须声明一个版本，该版本大于或等于 use 语句中列出的每个模块声明的 go 版本。

For example, if module M requires a dependency D with a go.mod that declares go 1.22.0, then M’s go.mod cannot say go 1.21.3.




# 开源


### pvc 优化

https://github.com/kubernetes/kubernetes/pull/126745/commits/39b6bd127823f2ce994550ae358c37fcd919e351


### pv finalizer

https://github.com/kubernetes/kubernetes/pull/125767

pv 报错： message: 'error getting deleter volume plugin for volume \"d-wz9h1fvsesbw49o5rarr\": no deletable volume plugin matched'

external-disk-provisioner 报错： shouldDelete is false, PersistentVolume is not Released PV="xxx"

pvc
- 删除 pvc 的时候检查 是否有 pod 引用, 所有实际删除动作都在pv的状态变化上执行

pv 上存在两个 finalizer
- external-provisioner.volume.kubernetes.io/finalizer
    - external-provisioner 管理， csi finalizer
- kubernetes.io/pv-controller
    - pv-controller 管理, intree finalizer
- kubernetes.io/pv-protection
    - pv_protection_controller 管理, 当pv 被 pvc bound 的时候加上, 非 bound 状态的时候去掉

external-provisioner
- 当开启了 HonorPVReclaimPolicy 默认为 pv 加上 finalizer (external-provisioner.volume.kubernetes.io/finalizer)
- finalizer 只给 reclaimPolicy 为 Delete 的 pv 加上， 当用户手动改成 Retain 的时候会自动去掉 
- 当这个finalizer被去掉的时候，  external-provisioner 不会去实际删除
- pv 只有在 released 状态的时候才会被 csi-provisioner 处理。  
- 先调用 openapi 删除 pv, 然后删除 pv, 给 pv 加 deletiontimestamp, 最后去掉 finalizer

kcm(syncVolume)
- 当检查到了 pv 所关联的 pvc 为空， 更新 pv 为 Released 状态(intree and outoftree) 
- 立即处理 volume
- 对于没开Honer featuregate 的情况。 
    - ***如果 volume 被打了 DeletionTimestamp， 直接返回，不做任何处理（说明已经有goroutine or external plugin 在处理了）***
    - 如果没有DeletionTimestamp, 检查 volume 如果还在引用/尚未被引用过， 这种情况直接返回
- 开始删除 volume, 根据 pv 的声明开始找plugin, intree 返回对应plugin， outoftree 直接返回(预期)
- intree 调用 plugin.deleter.Delete() 方法直接删除
- 如果开启了 featuregate,则在 Delete 之后 删除 kubernetes.io/pv-controller finalizer

回到上面的问题
kcm 和 external-provisioenr 同时对 pv 进行处理， external-provisioner 给pv 打上 deletiontimestamp, 但是由于有finalizer的存在，第一次无法顺利删除， 只能等待重试，kcm 继续走，发现没有 "pv.kubernetes.io/provisioned-by": "diskplugin.csi.alibabacloud.com", 
于是直接打上failed， external-provisioner在重试的时候不会对 failed 状态的pv 进行处理，于是失败

### namespace in pod

/var/run/secrets/kubernetes.io/serviceaccount/namespace

除了通过 env 注入之外， 也可以读取serviceaccount相关元信息来获取.



# 工作

I/O（输入/输出）和Die（集成电路中的小块半导体材料）。I/O是计算机系统与外部世界进行通信的过程，而Die是集成电路上制造特定功能电路的半导体元件。Die的大小和数量直接影响集成电路的成本和性能。通常，更大的Die意味着更高的集成度和成本，而分割Die可以降低成本但可能影响性能。


### it's already contains jbd

sh-4.4# blkid /dev/vdd
/dev/vdd: LABEL="xxxx" UUID="xxx" LOGUUID="xxx" TYPE="jbd"

'jbd' 是journey block journal (日志块日志)的缩写，它是Linux内核中用于ext3和ext4文件系统的日志管理部分。当出现"unknown filesystem type 'jbd'"错误时，通常意味着系统尝试挂载一个使用ext3或ext4格式的分区，但无法识别或加载对应的文件系统模块。这可能是由于内核版本不匹配、文件系统损坏、或在某些加密或存储解决方案中遇到的兼容性问题。解决方法可能包括更新内核、使用fsck检查和修复文件系统，或在挂载命令中明确指定文件系统类型。


### nfs 依赖 Python。。

关于pyyaml，nfs-utils依赖它, 以下文件都是 Python 写的
```
/usr/sbin/mountstats
/usr/sbin/nfsconvert
/usr/sbin/nfsdclddb
/usr/sbin/nfsdclnts
/usr/sbin/nfsiostat
/usr/sbin/rpcctl
```

### 安全越来越重要

https://docs.docker.com/security/for-admins/hardened-desktop/enhanced-container-isolation/config/

docker又搞了一套 enhanced-container-isolation 的机制， 还是主要专注于 docker 的安全隔离， 感觉 runc 的安全性确实是 docker 一直在钻研的， 站在 docker 的视角上也确实该加强，kata 这边确实最近因为安全问题受到了很多的关注. ok, 回归正题, 上面的文档主要介绍了一些 enhanced-container-isolation 的高级配置， 比如是否允许 container 挂载socket， 也就是是否与外部通信，影响 docker 的隔离性， 当然可以通过某些配置放开， 比如说镜像地址， 某种程度上还支持 derived 镜像，也就是一些本地生成的临时镜像。因为这些镜像可能并没有一个 docker 地址。

在限制了 imagelist 基础上， 还可以限制通过socket 发送的 commandline， 这个某种意义上就跟 givsor 很像了，可以统一限制可以通过 socket 发送到主机或者是其他namespace 的命令, 有白名单和黑名单之分，配置起来相对也比较容易


