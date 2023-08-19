---
layout: post
title:  "weekly report"
date:   2023-08-19 22:30:08 +0800
categories: weeklyreport
---

# 读书

### 大话存储

这周没有什么进展

# 社区

subtree update 的 commit 无法跟需求 pr 一起合并, 必须单独拎出来提一个 pr 才行, 不过目前看起来基本上差不多了

```
❯ git subtree pull --prefix=release-tools git@github.com:kubernetes-csi/csi-release-tools.git master --squash
```

# 工作

### readwriteonce vs readwriteoncepod

之前一直把这两个概念搞混了, readwriteonce 其实是针对节点维度的, readwriteoncepod 是针对 pod 维度的. oncepod 是在 1.22 之后才有的新功能


### loopdevice

这周又详细看了下 loopdevice , 因为在容器环境里面, 当我们摘除掉 loopdevice 关联的 file 的时候 loopdevice 会变成 delete 状态无法删除. 这个设计本身就是为了防止 设备符关联的文件被替换. 本身这个行为没问题, 但是目前我们的目的就是要修改这个, 所以导致了一定的麻烦, 需要先删除loopdevice 设备, 然后重新关联

> 注: /sys/block/loop0/loop/backing_file  这里包含了 loopdevice 设备关联的本地文件


### quota

因为 quota 这个值是实时算出来的, 这个在单机文件系统上其实问题不大, 但是在分布式的文件系统上,由于不可能实时计算(代价时间太高), 只能周期性的统计, 这就导致了如果我们在周期之前去 copy 一个非常大的文件的话, 那么 copy 的时候确实可能出现超出 quota 的情况


### docker root

出于安全起见, docker 的 root 跟  linux 的 root 其实并不等价, docker root 是阉割版的, 不包含特权的.这样可以保证容器的基本隔离性和安全性.  相关文档: https://docs.docker.com/engine/security/#linux-kernel-capabilities