---
layout: post
title:  "weekly report"
date:   2025-05-12 22:30:08 +0800
categories: weeklyreport
---


# 读书

### syscall.ReadDirent(fd, buf)

该方法调用了底层的 getdents or getdents64. 读取 file dentries 相关信息，具体结构体如下.也是 dentry 的主要包含的内容

```c
           struct linux_dirent {
               unsigned long  d_ino;     /* Inode number */
               unsigned long  d_off;     /* Not an offset; see below */
               unsigned short d_reclen;  /* Length of this linux_dirent */
               char           d_name[];  /* Filename (null-terminated) */
                                 /* length is actually (d_reclen - 2 -
                                    offsetof(struct linux_dirent, d_name)) */
               /*
               char           pad;       // Zero padding byte
               char           d_type;    // File type (only since Linux
                                         // 2.6.4); offset is (d_reclen - 1)
               */
           }
```
可能其中用的最多的就是自 2.6.4 内核版本开始的 d_type, 不过因为有一些文件系统并不支持这个，所以在编写应用程序的时候必须正确处理 DT_UNKNOWN 返回值  

### FS_IOC_ENABLE_VERITY

fs-verity（fs/verity/）是文件系统可以挂钩的支持层，用于支持只读文件的透明完整性和真实性保护。目前，ext4、f2fs 和 btrfs 文件系统都支持它。与 fscrypt 一样，支持 fs-verity 不需要太多文件系统专用代码。

fs-verity与dm-verity类似，但适用于文件而非块设备。对于支持fs-verity 的文件系统上的普通文件，用户空间可以执行一个 ioctl，使文件系统为文件构建一个 Merkle 树，并将其持久化到与文件关联的文件系统特定位置。

在此之后，文件将被设置为只读，所有对文件的读取都会自动根据文件的 Merkle 树进行验证。对任何损坏数据的读取（包括 mmap 读取）都将失败。

FS_IOC_ENABLE_VERITY 会导致文件系统为文件构建一个 Merkle 树，并将其持久化到与文件关联的特定文件系统位置，然后将文件标记为 verity 文件。在大文件上执行此 ioctl 可能需要很长时间，而且会被致命信号中断。


### linux 文件删除相关

#### 深入解析文件删除：从本地 unlink 到分布式存储的元数据同步

文件的删除操作，在用户看来或许只是一个简单的 rm 命令，但在操作系统和存储系统的底层，它涉及到一系列复杂而精密的机制。unlink 系统调用是这一过程的核心，它的行为和影响随着存储架构的演进而变得愈发复杂，尤其是在现代分布式存储系统中。本文将从 Linux 内核的 unlink 基础出发，逐步探讨其在 FUSE 用户空间文件系统中的作用，并最终深入分析 Ceph 和 MinIO 这两大主流开源存储方案是如何应对文件删除及元数据同步这一核心挑战的。

##### Linux 内核中的 unlink：基础与机制
在传统的单机 Linux 系统中，unlink 系统调用的主要职责是“解除链接”，而非立即“删除数据”。其工作流程与几个关键概念紧密相关：
 * 目录条目 (dentry) 和索引节点 (inode)：
   * dentry： 文件系统中的每个文件名都对应一个目录条目，它将文件名映射到一个 inode 号码。具体内容见 上面 dirent(directory entry) 结构
   * inode： 索引节点包含了文件的所有元数据（如权限、大小、时间戳、所有者以及指向实际数据块的指针），但不包含文件名本身。
 * 链接计数 (i_nlink)：
   * 每个 inode都有一个链接计数，记录有多少个文件名（dentry）指向这个 inode。这就是硬链接的实现基础。
   * 当执行 unlink(pathname) 时，内核会找到 pathname 对应的 dentry，并将其从目录中移除。随后，对应 inode 的链接计数减 1。
 * 文件数据的实际回收：
   * 链接计数为零： 只有当 inode 的链接计数变为零时，内核才会考虑回收该 inode 及其占用的数据块。
   * 打开的文件描述符： 一个重要的例外是，如果此时仍有进程打开了这个文件（即持有该文件的文件描述符），即使链接计数已为零，文件数据和 inode 也不会立即被删除。内核会等待最后一个持有该文件描述符的进程关闭它之后，才真正释放资源。这确保了正在运行的程序不会因文件被“删除”而突然无法访问其数据。
简而言之，unlink 首先移除的是文件名到文件元数据的“指针”。数据的实际删除则是一个依赖于链接计数和文件打开状态的延迟过程。

##### unlink 在远程分布式 FUSE 系统中的行为与挑战
在远程分布式 FUSE 系统中，unlink 的行为因其用户空间守护进程与后端存储的交互而变得更为复杂：
 * 网络通信： unlink 操作会触发网络请求到远程存储系统。
 * 一致性模型： unlink 的可见性（一个客户端删除文件后，其他客户端何时能感知）取决于远程系统的实现，可能从强一致性到最终一致性不等。
 * 原子性： 在分布式环境中确保 unlink 的原子性（操作要么完全成功，要么完全失败）更具挑战。
 * 处理“打开但已取消链接”的文件：
   * 默认情况下，FUSE 在 unlink 一个打开的文件时，会将其重命名为一个临时的隐藏文件（如 .fuse_hiddenXXX）。当最后一个进程关闭该文件时，FUSE 守护进程才会被通知进行最终删除。
   * 若使用 hard_remove 挂载选项，unlink 会尝试立即删除，这可能导致已打开该文件的进程后续操作失败，与标准 POSIX 行为有所差异。
FUSE 守护进程的核心任务是将来自内核的文件系统操作（如 unlink）转化为对后端存储系统的具体指令，并妥善处理分布式环境带来的各种挑战。
##### 实战剖析：Ceph 与 MinIO 的元数据同步策略

Ceph 是一个强大的开源统一存储平台，同时提供对象存储、块存储和文件系统 (CephFS) 功能。其元数据管理尤为关键：
 * 架构概览：
   * MDS (Metadata Server)： CephFS 依赖一个或多个 MDS 来管理文件系统的命名空间（目录结构、文件名、权限等）。MDS 通过动态子树分区来分配负载，实现高可用和可扩展性。
   * OSD (Object Storage Daemon)： 负责存储所有数据和文件系统元数据本身（以对象的形式）。
 * unlink 处理流程：
   * 客户端（如 CephFS FUSE 客户端）的 unlink 请求发送给活动的 MDS。
   * MDS 负责修改命名空间，从目录中移除条目，并更新 inode 的链接计数。
   * MDS 使用日志 (journal) 记录元数据更改，这些日志被持久化到 OSD 集群，确保元数据操作的原子性和持久性。
   * 如果链接计数为零且无打开句柄，MDS 会协调 OSD 删除相关的数据对象。
 * 元数据同步与一致性：
   * 强一致性： CephFS 为元数据操作提供强一致性。
   * Capabilities (权能)： 一种精巧的机制，MDS 通过授予客户端 capabilities 来管理其对 inode 的访问权限和缓存一致性。当元数据被修改（如 unlink）时，MDS 可以撤销或调整相关客户端的 capabilities，确保缓存同步。
   * 日志持久化： 元数据更改只有在被 MDS 日志记录并安全写入 OSD 后才算完成。
Ceph 通过其 MDS 集群、日志机制和 capabilities 系统，为 CephFS 提供了 POSIX兼容的强一致性元数据操作，即使在复杂的分布式环境中也是如此。

MinIO 是一款高性能、与 Amazon S3 兼容的开源对象存储服务。其设计哲学与 Ceph 有所不同，更侧重于扁平化的对象命名空间和去中心化。
 * 架构概览：
   * 去中心化： MinIO 没有中心化的元数据服务器。元数据（对象名、版本、校验和等）与数据本身一同分布在集群的各个节点和驱动器上。
   * xl.meta 文件： 每个对象通常都有一个对应的 xl.meta 文件，存储在同一纠删码集内的磁盘上，包含了该对象的关键元数据。
   * 纠删码： MinIO 使用纠删码来保证数据和元数据的高可用性和持久性。
 * DeleteObject (等效于 unlink) 处理流程：
   * 客户端发送 S3 API DeleteObject 请求。
   * 请求可达任一 MinIO 节点，该节点计算出对象的纠删码集。
   * 未版本化桶： 直接删除对象的数据分片及其 xl.meta 元数据文件。
   * 版本化桶：
     * 不指定版本ID的删除：创建“删除标记 (delete marker)”作为对象的最新版本，这本身也是一种元数据状态。
     * 指定版本ID的删除：永久删除该特定版本的对象数据及其 xl.meta。
 * 元数据同步与一致性：
   * 强一致性 (Strict Consistency)： MinIO 为其核心操作提供强一致性。成功的写操作（包括删除）对后续读操作立即可见。
   * 原子更新： 对 xl.meta 文件的更改以原子方式进行。
   * 分布式确认： 写操作（包括删除）需要在构成对象的纠删码集中的足够数量的驱动器上成功完成，以满足一致性和持久性要求。
   * 复制 (Replication)： 支持桶复制，可将对象、元数据、删除标记和版本删除操作同步到其他集群，提供最终一致性保障。
MinIO 通过将元数据与数据紧密耦合、利用纠删码以及原子更新机制，实现了在分布式对象存储场景下的高效元数据管理和强一致性。


# 社区

### node readiness

社区最近终于对readiness 有设计了，不过应该还是借了gpu 的光，如果只有存储的话估计大家也不会提这个。不过甭管怎么说吧，总之是一个进步

https://docs.google.com/document/d/11i2_rewvcbQkFFq1BIwHa7lgIefgZ8Mak-QMZjP7fFs/edit?tab=t.0

### containerd erofs snapshotter

最近正好有一个pr, 这里简单看一下 erofs snapshotter 主要是干什么的

#### rootfs snapshotter
详细研究 erofs 之前，我们先看下普通 overlay snapshotter 的实现, 以下就是一个最常见的容器挂载点

```
overlay on /xxx/io.containerd.runtime.v2.task/k8s.io/de591e4e10c60809468993bad0d522f089f6c258d933d2b0018e4282228742bf/rootfs type overlay (rw,relatime,lowerdir=/xxxx/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/494/fs:/xxx/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/493/fs:/xxxx/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/492/fs:/xxxx/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/491/fs:/xxxx/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/490/fs:/xxxx/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/489/fs,upperdir=/xxxx/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/495/fs,workdir=/xxxx/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/495/work,index=off)
```

下面我们简单过一下 image 从被拉取到组成 rootfs 的过程

1. containerd fetch image

```remote/docker/fetcher.go
func (f *fetcher) Fetch(ctx context.Context, desc ocispec.Descriptor, ref string) (io.ReadCloser, error) {
    // New http read seeker open function

    open := func(offset int64) (io.ReadCloser, error) {
        // firstly try fetch via external urls
        // Try manifests endpoints for manifests types
        // Finally use blobs endpoints
    }
    return newHTTPReadSeeker(open)

}
```
2. 实际镜像读取逻辑被封装到了 remote/docker/httpReadSeeker.go 里面
3. Unpack 函数封装了 image 处理程序， 相关 unpack 逻辑可以单独在一个 goroutine 里面执行， 位置在 unpack.unpacker.go
```golang
Unpack(h images.Handler) images.Handler
```
4. 这里存在一个关键函数 Handler 接口， 这个接口用于处理image 相关的处理， 包括上面的fetch， unpack 全部都是包装到 Handler 接口里面处理的

```
	handler = images.Handlers(append(baseHandlers,
		fetchHandler(store, fetcher, progressTracker),
		checkNeedsFix,
		childrenHandler, // List children to track hierarchy
		appendDistSrcLabelHandler,
	)...)
    handler = unpacker.Unpack(handler)
```
在 imageService 创建 image 之前，unpack 会推迟 blobs 的下载和解压缩
5. imageService 创建 image 是在 cri 启动 sandbox 的时候创建
6. snapshotter 并且调用 Prepare 方法将不同的 layer 组成 mount 命令, snapshotter 则在创建 container 的时候被初始化
```golang
	snapshotterOpt := []snapshots.Opt{snapshots.WithLabels(snapshots.FilterInheritedLabels(config.Annotations))}
	extraSOpts, err := sandboxSnapshotterOpts(config)
	if err != nil {
		return cin, err
	}
	snapshotterOpt = append(snapshotterOpt, extraSOpts...)

	opts := []containerd.NewContainerOpts{
		containerd.WithSnapshotter(c.imageService.RuntimeSnapshotter(ctx, ociRuntime)),
		customopts.WithNewSnapshot(id, containerdImage, snapshotterOpt...),
		containerd.WithSpec(spec, specOpts...),
		containerd.WithContainerLabels(sandboxLabels),
		containerd.WithContainerExtension(crilabels.SandboxMetadataExtension, &metadata),
		containerd.WithRuntime(ociRuntime.Type, podSandbox.Runtime.Options),
	}

	container, err := c.client.NewContainer(ctx, id, opts...)
```
6. 最后在 container_create.go 创建 container 的时候，会调用 snapshotter 的 Mount 方法

这块可能还需要根据日志来过一遍，如果只看代码的话可能会被多个入口搞混，这个下次再说，今天先继续 erofs， 

#### erofs 的挂载方式

下次请教下再看看


### lvcache

https://man7.org/linux/man-pages/man7/lvmcache.7.html




# 工作

### csi DeleteVolume 接口

组件是根据 pv 的状态判断是否触发 DeleteVolume 接口的，这个 pv 的状态在 informer 里面存在相关缓存，因为同时存在 pvc/pv 的缓存， 所以大概率会出现不一致（重复调用） 的场景，相关创建请求会要求幂等， 代码里面存在修改 informer cache 的逻辑，如果 pv yaml不改，这个修改cache 的逻辑将永远得不到更新

https://github.com/kubernetes-sigs/sig-storage-lib-external-provisioner/pull/179

存在这个pr问题

### ephemeral volume

想当然以为 pod ephemeral volume 会跟statefulset volumetemplate 一样被转成普通的volume声明， 结果发现， volume其实并不会出现变化，想想也是，毕竟是pod 的 spec，预期就是不变的

### kubelet 清理 文件卡死

内核hang死，应该是 xfs 的问题，已修复 https://web.git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/fs/xfs?id=9a5280b312e2e7898b6397b2ca3cfd03f67d7be1