---
layout: post
title:  "weekly report"
date:   2023-04-08 22:30:08 +0800
categories: weeklyreport
---

# 读书

两周摸鱼, 没读什么书

# 工作


### coredump file

应用使用了一个云盘, 进程正常将日志写入磁盘中. 但是突然在某一个时刻, 磁盘的 io 清零了, 同时 pod 所在的 rootfs 目录开始突然变大.

这个现象很奇特, 表面上看像是写入外挂盘中的数据突然写进了容器的rootfs. 但是经检查所有的盘都没有卸载过的痕迹, 并且应用本身还在正常运行(k8s 中所有挂卸载的行为都伴随这 pod 的启停)

最后发现结果是应用进程直接崩了, 由于配置的进程 core 文件的路径有问题, 导致这个 core 文件直接写到了 rootfs 里面. 进程崩了自然也就没有日志

### kubelet dp

之前以为 kubelet 没有相关的 hook 机制. 没想到想错了. 其实这个 hook 早就有了, terway 就既使用了 cni 又使用了 dp

https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/

### kubelet 强制删除 pod

在 kubelet volumemanager reconciler 中有这么一段逻辑 ```cleanupMounts``` 这里的主要逻辑为, 如果在 kubelet 的 dsw 中找不到当前pod 使用的 volume 信息(代表当前 pod 处于 terminating/deleted 状态), 则清理 pod 使用的volume 相关挂载点(调用 operationExecutor.UnmountVolume 方法)
- reconciler.cleanupMounts
  - operationExecutor.UnmountVolume
  - operationGenerator.GenerateUnmountVolumeFunc(以 fs 为例)
  - csi_mounter.go
    - csiMountMgr.TearDownAt
    - csi.NodeUnpublishVolume

有趣的是, cleanupMounts 这个方法只执行一次, 这就导致了, 如果 pod 处于 terminating 状态, 并且 kubelet 发生了重启. kubelet 重启后会尝试链接 csi 进行 umount, 如果这时候 csi 没有起来(初始化中) 这就会导致 kubelet 会直接回收掉 pod, 而不去尝试重试

姑且记录一下吧





