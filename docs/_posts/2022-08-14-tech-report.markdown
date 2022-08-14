---
layout: post
title:  "weekly report"
date:   2022-08-14 22:30:08 +0800
categories: weeklyreport
---

# 读书

### 论文

https://www.usenix.org/system/files/conference/fast17/fast17-vangoor.pdf

这周利用上下班时间主要读了这篇论文, 这篇论文没有提出什么新的概念, 而是主要整体评估了 fuse 的性能. 分析了 fuse 在不同场景下的性能表现. 个人感觉这种论文更像是工具书, 后续如果有相关的需求(性能优化,文件系统选型)才来查看. 简单总结一下就是如下特点

- fuse 在内核中实现了 fuse driver, 该 fuse driver 将用户从 VFS 下发的任务经过 filter 和包装通过 /dev/fuse 管道设备发送给用户空间的 fuse daemon
- 由于上面的架构,数据需要在用户空间和内核空间中来回转发, 导致 fuse 系统的性能天生就是劣于内核文件系统
- 但是 fuse 系统做了很多优化(writeback cache, zerocopy via splicing, and multi-threading.) 可以让 fuse 文件系统在某些特殊场景上得到和内核文件系统持平甚至更优的性能
- 大家使用 fuse 主要是因为 fuse daemon 是运行在用户空间的, 我们很容易就可以对文件系统进行功能的验证和 bug 修复, 所以 fuse 文件系统经常被用于文件系统的开发和验证.一旦文件系统稳定之后, 将其转换成内核文件系统
- 论文里面使用了 ext4 作为自定义 fuse 文件系统的底层, 并且与普通的 ext4 文件系统对比,这样就可以屏蔽文件系统本身的差异, 仅观察fuse这种模式对读写效率的影响  


# 日常

### mountOptions

之前一直没有注意到, mountOptions 是如何被注入到 csi 中的, 之前有个人问起来,所以详细的看了下, 如果是动态创建的情况下, mountOptions 是在 storageclass 中定义的, 在 external-provisioner 中, 一旦 pv 被正常创建, mountOptions 会被设置到 pv.Spec.MountOptions 的字段中, kubelet 在调用 csi 之前, 从 pv 中将 mountOptions 提取出来,并作为单独的参数传入到 NodeStageVolume 中, 由 csi 执行具体的挂载. 如果 mountOptions 里面设定了 rw, 则 csi 则会在挂载之前使用 fsck 检查块设备并修复问题
