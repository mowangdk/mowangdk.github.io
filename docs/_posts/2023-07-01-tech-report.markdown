---
layout: post
title:  "weekly report"
date:   2023-07-01 22:30:08 +0800
categories: weeklyreport
---

接近一个月没有写什么东西了, 这一个月俗事太多, 没有好好读书, 不过工作中倒是发生了一些有趣的事情可以写一写

# 读书

摸鱼, 七月恢复

# 社区


issue: https://github.com/kubernetes-csi/external-snapshotter/issues/843

issue 本身已经被 reviewer lgtm 了, 但是 owner 最近 PTO 了, 导致迟迟没有合并

# 工作

### 卸载路径失效问题

最近经常在 fuse & nfs 挂载点上遇到这个问题. 在触发 csi-plugin ```NodeUnPublishVolume``` 方法的时候, 发现挂载点已经不存在了, 但是等到流程流转到 kubelet 上的时候发现这个挂载点还是存在的, 至今这个问题依旧没有被探明, 目前有两点怀疑
- 容器内 mount 命令无法检查到真实挂载情况(但是之前一直没问题)
- 挂载点被第三方进程处理, 因为挂载点没有任何的元信息(也不可能有, mount options 里面都是指定的参数, 没有自定义的元信息), 我们目前也同样无法判明这种情况

鉴于以上问题无法判别, 我们采取了一个社区推荐的方式, 就是当检测 targetpath 并非挂载点 & 目录下无数据的时候, 直接删除掉这个目录, 目前看来这种做法已经收到了一些成效, 后面就再也没出现上述情况了, 目前看来可能性 2 大一些


### 云盘监控数据丢失问题

一个集群需要帮忙定位云盘数据监控问题, 相关指标暴露到 metrics 的接口上, 但是发现及时节点上存在云盘挂载依旧没有 metrics, 经确认是程序缺少rbac权限问题

### oss

在 oss 上通过扫描上传的带有dir 的文件其实并不是真正存储在与 oss 中的对象, 只是虚拟出来的带有 dir 的文件. 当我们实际将这个 dir 挂载到宿主机上的时候,会报timeout 的错误, 因为 fuse 根本找不到这个路径, 但是通过 fuse 创建的确实是在 oss 中存在对象的路径, 这个就应该算控制台上实现上的问题了


### fuse 

Fuse 调用 chmod 的时候出现 io error , 查看 fuse chmod 实现, 发现chmod 的时候会检查 parents dir 的权限, 当前进程 user 的权限, 以及 操作对象的 oss metadata 来确定当前 user 是否可以操作, 如果不能操作的话就直接返回报错



