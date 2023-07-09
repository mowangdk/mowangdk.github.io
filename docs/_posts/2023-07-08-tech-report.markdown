---
layout: post
title:  "weekly report"
date:   2023-07-08 22:30:08 +0800
categories: weeklyreport
---

# 读书

继续读了大话存储这本书, FC 相关的内容太多了索性直接跳过了. 看了下整体直接从下一章读反而更有效率

### NFS & FC(disk)
虽然说两者可能不是一个级别, 甚至不是一个地方的协议, 不过为了方便理解姑且用这种表述方式吧, 使用 NFS 协议的存储使用 tcp/ip 协议进行网络传输, 实际的文件系统是存储在后端存储集群的. 而类似云盘的这种, 文件系统虽然也是存储在后端的不过生成却是在前端的. 并且使用的是专属的 FC 网络(协议消耗少)

# 社区


issue: https://github.com/kubernetes-csi/external-snapshotter/issues/843

issue 本身已经被 reviewer lgtm 了, 但是 owner 最近 PTO 了, 导致迟迟没有合并

# 工作

### permission denied

现象: 
这周遇到了一个排查了比较久的问题, 原因是我们在更新 baseimage 的是否发现, 明明是相同的代码, 但是却出现了不同的行为, 原先的镜像可以正常创建, 但是更新后的镜像就没有办法了. 会在创建目录的时候报 permissiondenied 的错误

排查思路:

首先肯定是这两个镜像之间有差异, 目前已知的差异是更新后的镜像去掉了 shell, 当然还包括其他差异, 从这点出发, 首先怀疑是路径创建因为 shell 而失败.于是我们在 dockerfile 里面使用 workdir 来创建目录, 但是依旧有问题, 怀疑并非 shell 导致, 可能纯粹是因为其他差异 报错是权限不够, 但是 dockerfile 和 yaml 都没有变, 代码也没变,究竟是哪里出现了问题. 目录是镜像二进制创建的, 二进制用的是镜像/yaml 中定义的 user, 使用 docker inspect 观察两个不同的镜像, 发现更新后的镜像使用了 uid: 65536 , 于是猜测是这个问题导致的. 在 yaml 中复写 0 之后 恢复

### kubernetes emptydir limit 

详细看了下 emptydir limit 是如何实现的. 发现是复用了文件系统的能力, 只有当 xfs/ext4 的时候才会生效. 代码位置在 ```pkg/volume/util/fsquota/common/quota_common_linux_impl.go``` 


### kworker zombie

最近发现我们的一个进程莫名的变成了zombie内核进程, 没有办法通过 kill -9 杀掉, 只能等待进程运行完或者是重启节点.本来想用 strace 追一下的, 但是发现内核进程没有办法 attach, 会报错 permission denied. 查了一下想使用 ```echo 0 > /proc/sys/kernel/yama/ptrace_scope``` 发现也没有用, 只能先摆在这里了


### kubernetes du
发现 kubernetes 在统计 volume metrics 的时候会遍历指定目录下所有的文件大小计算(du) secrets, hostpath, configmap, emptydir 都会按照这种方式指定, 我们整体评估下来是有一定风险的, 这个我们下周再跟监控同学确认下


### 集群里面大量 pod ContainerCreating

看了下 external-attacher 发现有客户端限流的场景, 如果客户端有超时删除重建的场景, 这个pod 会越来越多. 所以我们一般建议超市时间长一些, 这样不会等到 external-attacher 刚消费到这个 pvc 又被删除这种情况