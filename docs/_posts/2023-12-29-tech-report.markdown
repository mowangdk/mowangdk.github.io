---
layout: post
title:  "weekly report"
date:   2023-12-29 22:30:08 +0800
categories: weeklyreport
---

23 年最后一篇 tlog 了, 感慨一下


# 读书

本周没有什么实际进展, 

# 社区

### kubeadm remove files under umount failed

之前这个 pr 拖了能有两个月了, 最近是因为有一个其他用户提了个类似的 issue, 导致这个问题被重新拿出来看了. 社区的人还是纠结如果在 umount failed 的时候直接报错会影响默认行为. 但是在我看来这种反而应该是正常的. 总之这个事情需要得到 kubadm 其他成员的认可. 下周休假, 我尽量在这种把开源的事情搞定吧



### csi sidecar all in one

最近社区也在做这个事情(其实八月份就开始搞了), 尽管还没有 kep, 不过大佬已经做了初步的文档设计. 看了下, 因为要考虑各种各样的情况. 所以整个项目的milestone 有很多. 包括如何将 sidecar 里面新增的 pr 合并到 allinone 项目里面. 跟这个项目的 owner 说了我也想参加这个项目, 他便给我了许多文档, 先慢慢看吧,

- https://docs.google.com/document/d/1z7OU79YBnvlaDgcvmtYVnUAYFX1w9lyrgiPTV7RXjHM/edit 
- https://docs.google.com/document/d/1AKqJeAlBL8PkH8D9zABCZ82Bk1N46EygKPvVh5p4-qU/edit
- https://docs.google.com/document/d/1GAE1a0GZnsUmHnc_p4uxmQ7V_HZ6z5G4bWab5D6kkEE/edit


# 工作

上周因为写了一个社区的文章, 所以没有详细整理上周的问题. 这周统一整理下, 也是千奇百怪

### 云盘热扩容失败

非常神奇, 客户正常在集群里面通过变更 pvc 的 storage 大小对云盘进行扩容. 但是最终发现文件系统大小还是没变的. 但是排查所有组件的日志, 发现没有一个有报错. 唯一有疑点的是 openapi 扩容的时间点和本地扩容的时间点完全不一样. 这里统一记录一下

```
```

```
```


### golang 

发现了用户在升级到 v5.10 内核的 os 之后, 相关的应用性能出现了下跌. 但是应用本身没有发布新版本, 只是切换了一个内核, 就出现了问题. 一开始通过性能分析, fstrace, 热点图等工具发现, golang 的 io.copy 调用占了非常大的 cpu, 并且追踪 golang 代码发现. io.copy 的行为在内核 5.3 的时候进行了一次升级. 会减少内核空间和用户空间之间的数据 copy, 这本质是个优化方案(系统调用优化),如果内核程序代码不出 bug 的情况下, 无论怎么说都不会出现性能下降的情况. 经过内核同学排查当然内核没有相关问题. 于是就查其他方向. 由于程序主要是做 io.copy. 所以怀疑是否在跨多个内核版本升级的时候出现了io 配置的变更, 于是就开始找. 发现确实如此, 内核修改了 read_ahead 预读取的配置数量, 原因是因为在 io通信不好的情况下. 这个值过大会引起网络重传的进一步恶化.所以内核就把默认值给改了, 经过测试验证确实就是这个问题. 也是绝了.

```
Introduce a new ReadFrom method on *os.File, which makes *os.File
implement the io.ReaderFrom interface. If dst and src are both files,
this enables io.Copy(dst, src) to call dst.ReadFrom(src), which, in
turn, will call copy_file_range(2) if possible. If copy_file_range(2)
is not supported by the host kernel, or if either of dst or src
refers to a non-regular file, ReadFrom falls back to the regular
io.Copy code path.
```

```
https://github.com/golang/go/commit/7be3f09deb2dc1d57cfc18b18e12192be3544794

https://github.com/golang/go/commit/1c7650aa93bd53b7df0bbb34693fc5a16d9f67af

https://github.com/golang/go/issues/42400

https://access.redhat.com/solutions/5953561
```

### fsgroup

https://github.com/kubernetes/kubernetes/blob/60dcf7fd8dbbb05fa02a08217001f98c5b1507c2/pkg/volume/volume_linux.go#L73

fsgroup 在默认情况下只会影响 readwriteonce & 设置了 fstype 的存储. 这样当用户使用像是 nas / oss 类型的存储的时候, 其实默认就不会做 fsgroup 相关的能力.

```golang
    // ReadWriteOnceWithFSTypeFSGroupPolicy indicates that each volume will be examined
 // to determine if the volume ownership and permissions
 // should be modified. If a fstype is defined and the volume's access mode
 // contains ReadWriteOnce, then the defined fsGroup will be applied.
 // This mode should be defined if it's expected that the
 // fsGroup may need to be modified depending on the pod's SecurityPolicy.
 // This is the default behavior if no other FSGroupPolicy is defined.
 ReadWriteOnceWithFSTypeFSGroupPolicy FSGroupPolicy = "ReadWriteOnceWithFSType"

 // FileFSGroupPolicy indicates that CSI driver supports volume ownership
 // and permission change via fsGroup, and Kubernetes will change the permissions
 // and ownership of every file in the volume to match the user requested fsGroup in
 // the pod's SecurityPolicy regardless of fstype or access mode.
 // Use this mode if Kubernetes should modify the permissions and ownership
 // of the volume.
 FileFSGroupPolicy FSGroupPolicy = "File"

 // NoneFSGroupPolicy indicates that volumes will be mounted without performing
 // any ownership or permission modifications, as the CSIDriver does not support
 // these operations.
 // This mode should be selected if the CSIDriver does not support fsGroup modifications,
 // for example when Kubernetes cannot change ownership and permissions on a volume due
 // to root-squash settings on a NFS volume.
 NoneFSGroupPolicy FSGroupPolicy = "None"
```

代码位置: 

```golang
func (c *csiMountMgr) supportsFSGroup(fsType string, fsGroup *int64, driverPolicy storage.FSGroupPolicy) bool {}
```
pkg/volume/csi/csi_mounter.go



### cadvisor

还是老生常谈的问题, 关于临时存储的监控问题, 由于 docker 和 containerd 的默认行为不同. cadvisor 在某个特殊分支里面通过 kubelet 暴露的```/stat/summary``` 接口返回的 json 的数据来暴露相关指标. 所以目前相关容器 rootfs/ephemeral storage 都是由 csi 来收集进而暴露. 后续貌似都会统一到这个接口进行调用. 相关 kubelet 调用链条是

```golang
call grpc /stat/summary
-> summaryProvider.Get()
-> sp.provider.ImageFsStats() // provider == kubelet
    // The following stats are provided by either CRI or cAdvisor.
    //
 // ImageFsStats returns the stats of the image filesystem.
 // Kubelet allows three options for container filesystems
 // Everything is on node fs (so no image filesystem)
 // Container storage is on a dedicated disk (imageFs is separate from root)
 // Container Filesystem is on root and Images are stored on ImageFs
 // First return parameter is the image filesystem and
 // second parameter is the container filesystem
 ImageFsStats(ctx context.Context) (imageFs *statsapi.FsStats, containerFs *statsapi.FsStats, callErr error)  
-> kl.criStatsProvider.ImageFsStats(ctx context.Context)
-> p.imageService.ImageFsInfo() 
--- containerd
-> func (c *criService) ImageFsInfo(ctx context.Context, r *runtime.ImageFsInfoRequest) 
--- criserver
-> (c *criService) ImageFsInfo(ctx context.Context, r *runtime.ImageFsInfoRequest) (*runtime.ImageFsInfoResponse, error)
-> snapshots := c.snapshotStore.List()
-> snapshot.Size()
```

这里需要单独看下, cri service 在启动的时候会启动 sync reconcile controller. 定期将 snapshot 里面的磁盘使用量存储在 store 里面, 再由上面的grpc server 消费

```
--- pkg/cri/server/service.go
-> snapshotsSyncer.start()
-> s.snapshotter.Usage(ctx, info.Name)
--- snapshots/overlay/overlay.go
   -> (o *snapshotter) Usage(ctx context.Context, key string)
-> s.store.Add(sn) // snapshotstore.Store
```
