---
layout: post
title:  "weekly report"
date:   2023-10-15 22:30:08 +0800
categories: weeklyreport
---


# 读书

### 大话存储
虚拟化的部分也算是读完了吧, 但是实际上并没有读到我想要的虚拟化, 他讲的是单纯存储的虚拟化, 类似 lvm 这种,将单机的磁盘进行格式化的细分. 或者是外部存储将多个盘整合成存储池, 并且也没有讲得很细. 后面看了下主要就是存储集群了, 主要看下 nas 相关的部分


# 社区
最近在帮忙社区做一些整理工作, 主要是一些 e2e 的整理工作. 算是摸清楚了 csi 下面的那些 sidecar 是如何进行 e2e 测试的, 简单来说就是写了一个 mock driver, 这个 mock driver 实现了所有 csi 的接口, 这样需要测到不同的 sidecar 的时候, 就直接将 sidecar 和 mockserver 使用 kind 同步部署到 ci 环境里面, 在里面运行相关的示例进行测试, 值得注意的是, 所有的e2e 测试用例其实都是存储在kubernetes 代码中的, 相关环境准备是写在 csi-release-tools 这个 repo 里面

下面是相关流程方法的基本调用
```
-> RunE2ETests(t *testing.T)
     -> ginkgo.RunSpecs(t, "Kubernetes e2e suite", suiteConfig, reporterConfig)

-> AddDriverDefinition(filename string)
    -> "External Storage " +

-> GetDriverNameWithFeatureTags(driver TestDriver)
   -> "[Driver: %s]%s"

-> RegisterTests(suite TestSuite, driver TestDriver, pattern TestPattern)
    -> "[Testpattern: %s]%s %s%s"

-> (p *provisioningTestSuite) DefineTests(
```


# 工作

### k8s configmap

Configmap volumemounts defaultMode 只会对特定的文件执行 chmod. 对文件夹自身不会变更其属性, 默认为 0755(确实也没必要, 毕竟用户最终访问的是文件, 目录只要是能够访问就可以了) 并且 kubelet 有很多 metrics 要收集, 如果用户自定义了 root 的权限后续 kubelet 来收集 metrics 肯定是有问题的

相关 issue: https://github.com/kubernetes/kubernetes/issues/76158


### 同一个 pod 在节点上存在两组进程

用户扩容节点发现节点上的 csi-agent 起不来, 经排查, 用户节点上存在两组 csi-agent 进程. 但是节点上只有一个 pod.  经排查 第一组 csi-agent 进程是 containershim 维护的. 也就是确实是 contiainerd 启动的容器. 经 cotnainerd 同学确认, 是由于 container 本身有过重启, 并且重启之后找不到 blotdb (/xxx/xxx/containerd/io.containerd.metadata.v1.bolt/meta.db). 无法找回之前的 shim 进程. 进而重新启动了一组进程. 出现这种问题的原因一般是用户自己删除了 db 文件, 或者修改了 containerd 的根目录

### 整理了一下 mountpoints 的一些参数

最近遇到了一个 mountoption 的问题, 正好之前没有详细整理过, 趁此机会直接整理下,这些都是一些加速常见参数

Nodiratime: 

Do not update access times for directories on this filesystem.  This flag provides a subset of the functionality provided by MS_NOATIME; that is, MS_NOATIME implies MS_NODIRATIME.

noatime: 
Do not update access times for (all types of) files on this filesystem.

data=writeback: 

File System administrators in Linux are tempted to use the data=writeback option to achieve higher throughput from their file-system, but this comes with the side-effects of potential data corruption and data-secuity breach.

The 'data=writeback' mode does not preserve data ordering when writing to the disk, so commits to the journal may happen before the data is written to the file system. This method is faster because only the meta data is journaled, but is not good at protecting data integrity in the face of a system failure.

nobarrier: 

In operating systems, write barrier is a mechanism for enforcing a particular ordering in a sequence of writes to a storage system in a computer system. For example, a write barrier in a file system is a mechanism (program logic) that ensures that in-memory file system state is written out to persistent storage in the correct order

> 上面这两个配置还挺像的, 一个是写数据的时候是否要保证 journal 一定在实际写数据之后写入, 另一个是写多个数据的顺序是否一定要保证. 

lazytime: Reduce on-disk updates of inode timestamps (atime, mtime, ctime) by maintaining these changes only in memory.  The on-disk timestamps are updated only when:

the inode needs to be updated for some change unrelated to file timestamps;

the application employs fsync(2), syncfs(2), or sync(2);

an undeleted inode is evicted from memory;

more than 24 hours have passed since the inode was written to disk.


delalloc:

delalloc	(*)	Defer block allocation until just before ext4
			writes out the block(s) in question.  This
			allows ext4 to better allocation decisions
			more efficiently.

nodelalloc		Disable delayed allocation.  Blocks are allocated
			when the data is copied from userspace to the
			page cache, either via the write(2) system call
			or when an mmap'ed page which was previously
			unallocated is written for the first time.

--one-file-system
when removing a hierarchy recursively, skip any directory that is on a file system
different from that of the corresponding command line argument

### umount 参数

-f: Umount force_umount 只会强制卸载 nfs/fuse 等网络文件系统, ext4 是不会强制删除的 如果出现 fd 占用的情况下. force umount 是卸载不掉的. 



### 奇怪的 error

dmesg -T 报了如下错误

```
 EXT4-fs (vdo): VFS: Can't find ext4 filesystem
 EXT4-fs (vdo): barriers disabled
 EXT4-fs (vdo): mounted filesystem with writeback data mode. 
 EXT4-fs (vdo): barriers disabled
 EXT4-fs (vdo): mounted filesystem with writeback data mode. 
 EXT4-fs (vdo): barriers disabled
 EXT4-fs (vdo): mounted filesystem with writeback data mode. 
 virtio_blk virtio18: [vdo] * 512-byte logical blocks 
 EXT4-fs (vdo): VFS: Can't find ext4 filesystem
 EXT4-fs (vdo): barriers disabled
 EXT4-fs (vdo): mounted filesystem with writeback data mode. 
 EXT4-fs (vdo): barriers disabled
 EXT4-fs (vdo): mounted filesystem with writeback data mode. 
 virtio_blk virtio18: [vdo] * 512-byte logical blocks 
 EXT4-fs (vdo): VFS: Can't find ext4 filesystem
 EXT4-fs (vdo): barriers disabled
 EXT4-fs (vdo): mounted filesystem with writeback data mode. 
 EXT4-fs (vdo): barriers disabled
 EXT4-fs (vdo): mounted filesystem with writeback data mode. 
 virtio_blk virtio18: [vdo] * 512-byte logical blocks 
 EXT4-fs (vdo): VFS: Can't find ext4 filesystem
 EXT4-fs (vdo): barriers disabled
 EXT4-fs (vdo): mounted filesystem with writeback data mode. 
 EXT4-fs (vdo): barriers disabled
 EXT4-fs (vdo): mounted filesystem with writeback data mode. 
 virtio_blk virtio18: [vdo] * 512-byte logical blocks 
 EXT4-fs (vdo): VFS: Can't find ext4 filesystem
 EXT4-fs (vdo): barriers disabled
 EXT4-fs (vdo): mounted filesystem with writeback data mode.
```

十分奇怪的日志, 可以看出一开始是 VFS 的报错,这个是预期的. 然后就是挂载成功, 相关参数的配置, 然后又开始报错,并且都是同一个设备. 一直在反复的重试, 下面是正常的日志

```
pci 0000:00:07.0: [1af4:1001] type 00 class 0x010000
pci 0000:00:07.0: reg 0x10: [mem 0x00000000-0x00000fff 64bit pref]
pci 0000:00:07.0: reg 0x18: [mem 0x00000000-0x00000fff 64bit pref]
pci 0000:00:07.0: BAR 0: assigned [mem 0xc0000000-0xc0000fff 64bit pref]
pci 0000:00:07.0: BAR 2: assigned [mem 0xc0001000-0xc0001fff 64bit pref]
virtio-pci 0000:00:07.0: virtio_pci: leaving for legacy driver
virtio_blk virtio4: [vdb] 104857600 512-byte logical blocks (53.7 GB/50.0 GiB)
EXT4-fs (vdb): VFS: Can't find ext4 filesystem
EXT4-fs (vdb): mounted filesystem with ordered data mode. Opts: (null)
IPv6: ADDRCONF(NETDEV_UP): veth90aa64e9: link is not ready
cni0: port 11(veth90aa64e9) entered blocking state
cni0: port 11(veth90aa64e9) entered disabled state
device veth90aa64e9 entered promiscuous mode
IPv6: ADDRCONF(NETDEV_UP): eth0: link is not ready
IPv6: ADDRCONF(NETDEV_CHANGE): eth0: link becomes ready
IPv6: ADDRCONF(NETDEV_CHANGE): veth90aa64e9: link becomes ready
cni0: port 11(veth90aa64e9) entered blocking state
cni0: port 11(veth90aa64e9) entered forwarding state
cni0: port 11(veth90aa64e9) entered disabled state
device veth90aa64e9 left promiscuous mode
cni0: port 11(veth90aa64e9) entered disabled state
virtio-pci 0000:00:07.0: EDR: ACPI event 0x3 received
pci 0000:00:07.0: [1af4:1001] type 00 class 0x010000
pci 0000:00:07.0: reg 0x10: [mem 0x00000000-0x00000fff 64bit pref]
pci 0000:00:07.0: reg 0x18: [mem 0x00000000-0x00000fff 64bit pref]
pci 0000:00:07.0: BAR 0: assigned [mem 0xc0000000-0xc0000fff 64bit pref]
pci 0000:00:07.0: BAR 2: assigned [mem 0xc0001000-0xc0001fff 64bit pref]
virtio-pci 0000:00:07.0: virtio_pci: leaving for legacy driver
virtio_blk virtio4: [vdb] 104857600 512-byte logical blocks (53.7 GB/50.0 GiB)
EXT4-fs (vdb): mounted filesystem with ordered data mode. Opts: (null)
IPv6: ADDRCONF(NETDEV_UP): veth6078d6d7: link is not ready
cni0: port 11(veth6078d6d7) entered blocking state
cni0: port 11(veth6078d6d7) entered disabled state
device veth6078d6d7 entered promiscuous mode
IPv6: ADDRCONF(NETDEV_UP): eth0: link is not ready
IPv6: ADDRCONF(NETDEV_CHANGE): eth0: link becomes ready
IPv6: ADDRCONF(NETDEV_CHANGE): veth6078d6d7: link becomes ready
cni0: port 11(veth6078d6d7) entered blocking state
cni0: port 11(veth6078d6d7) entered forwarding state
```

上面的重试应该是 csi 导致的, 这个也查明了, 问题就是为什么 csi 一直在反复重试, 通过 csi 日志可知, 这个盘的数据是有问题的, 没办法识别到正确的文件系统, 也没有办法去格式化, 所以一直反复横跳
问题本身没有什么, 主要是之前一直没有注意我们在初始化容器存储环境时候正常的 dmesg 日志, 特此记录下