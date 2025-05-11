---
layout: post
title:  "weekly report"
date:   2025-05-02 22:30:08 +0800
categories: weeklyreport
---


# 读书

# 社区

### AIO

本周做了一些 KEP 的重构工作，在昨天（今天凌晨）的会议上着重讨论了一下 dependency management 的事情，有一个点是之前没有注意到的， 关于csi-lib-utils & csi-release-tools 的，之前我以为go.work 是为 sidecars 准备的，但是今天才意识到 这个其实是为 csi-lib-utils & csi-release-tools 准备的, 最终我们还是决定继续去掉 sidecar 里面的 gomod 来保证一致性，毕竟我们终态确实是要演进到这里的

### pv affinity mutable

同样在今天的会议上提了一下，xing他们貌似也有同样的需求， 但是总体来说大家貌似兴趣不高，但是好在也没有反对， 后面我们就提一个kep来做这个事情吧


# 工作

### containerd

containerd overlayfs snapshot 模型重点总结

1. Snapshot 的角色
	•	snapshot 是 containerd 中用于构建文件系统层次结构的核心机制。
	•	有两类：
	•	View：只读层，通常用于镜像层或只读挂载。
	•	Active：可写层，用于容器运行时的写入。

⸻

2. OverlayFS 的文件系统合成方式
	•	overlayfs 通过将多个层叠加生成最终视图：

merged = upperdir + lowerdir[N] + lowerdir[N-1] + ...


	•	写操作只落在 upperdir。
	•	读取时优先从 upperdir 查找，再往下查找 lowerdir。

⸻

3. 磁盘目录结构

存放路径：

/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/

其中：
	•	/snapshots/<id>/ 是每个 snapshot 的物理目录。
	•	fs/：只读层内容。
	•	upper/：Active 层的写目录。
	•	work/：overlayfs 的工作目录（Active 层需要）。
	•	metadata.db：containerd 使用的 BoltDB 数据库存储 snapshot 的元信息（如状态、父子关系等）。

⸻

4. 常见流程

操作	结果
镜像拉取	每一层生成一个只读 View snapshot
创建容器（Prepare）	创建 Active snapshot，生成 upper/、work/
挂载容器文件系统	overlayfs 合并 lowerdir 与 upperdir，生成 merged
Commit snapshot	把 Active 层提交为新的 View 层（变成只读）
删除 snapshot	释放磁盘空间，清除关联的 upper/work/fs 目录


⸻

5. snapshot 生命周期核心 API（containerd 中）
	•	Prepare()：创建 Active 层
	•	View()：创建只读视图
	•	Commit()：从 Active 提交为只读 View
	•	Mounts()：获取 overlay mount 配置
	•	Remove()：清理 snapshot

⸻

6. 额外提示
	•	snapshot 是分层增量存储，便于重用和快速创建。
	•	overlayfs 的 snapshot 结构与 Git commit 概念类似，有 parent-child 层级。
	•	每个 snapshot 的关系和元信息都记录在 metadata.db 中，由 containerd 自动维护。

⸻

#### opts 

*systemd containerd management*

```
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target

[Service]
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/local/bin/containerd
Delegate=yes
KillMode=process

[Install]
WantedBy=multi-user.target
```

Delegate: 允许 containerd 及其运行时管理所创建容器的 cgroup。如果不设置该选项，systemd 会尝试将进程移入它自己的 cgroup，从而导致容器及其运行时无法正确计算容器的资源使用情况。


KillMode: KillMode 用于处理 containerd 进程关闭的情况， 默认情况下，systemd 会在其命名的 cgroup 中查找并杀死它所知道的该服务的所有进程。但是这不是我们想要的。作为操作人员，我们希望能够升级 containerd，并允许现有容器不间断地继续运行。将 KillMode 设置为 process ，可以确保 systemd 只杀死 containerd 守护进程，而不会杀死任何子进程，例如 shims 和 containers。


*configuration*
主要分为两类， 一个叫做 root， 存储持久化数据， 一个叫做 state 主要用于存储临时数据

```root
/var/lib/containerd/
├── io.containerd.content.v1.content
│   ├── blobs
│   └── ingest
├── io.containerd.metadata.v1.bolt
│   └── meta.db
├── io.containerd.runtime.v2.task
│   ├── default
│   └── example
├── io.containerd.snapshotter.v1.btrfs
└── io.containerd.snapshotter.v1.overlayfs
    ├── metadata.db
    └── snapshots
```

```state
/run/containerd
├── containerd.sock
├── debug.sock
├── io.containerd.runtime.v2.task
│   └── default
│       └── redis
│           ├── config.json
│           ├── init.pid
│           ├── log.json
│           └── rootfs
│               ├── bin
│               ├── data
│               ├── dev
│               ├── etc
│               ├── home
│               ├── lib
│               ├── media
│               ├── mnt
│               ├── proc
│               ├── root
│               ├── run
│               ├── sbin
│               ├── srv
│               ├── sys
│               ├── tmp
│               ├── usr
│               └── var
└── runc
    └── default
        └── redis
            └── state.json
```

*plugin*

containerd 的内核非常小。真正的功能来自插件。从snapshotters、runtimes and content，所有这些都是在运行时注册的插件。由于这些插件千差万别，我们需要一种为插件提供类型安全配置的方法。我们能做到这一点的唯一方法是通过配置文件，而不是 CLI 标志。在配置文件中，您可以通过 [plugins.<name>] 部分为您使用的插件集指定插件级选项。你必须阅读插件的具体文档，找到你的插件所接受的选项。

```
$ ctr plugins ls
TYPE                            ID                    PLATFORMS      STATUS
io.containerd.content.v1        content               -              ok
io.containerd.snapshotter.v1    btrfs                 linux/amd64    ok
io.containerd.snapshotter.v1    aufs                  linux/amd64    error
io.containerd.snapshotter.v1    native                linux/amd64    ok
io.containerd.snapshotter.v1    overlayfs             linux/amd64    ok
io.containerd.snapshotter.v1    zfs                   linux/amd64    error
io.containerd.metadata.v1       bolt                  -              ok
io.containerd.differ.v1         walking               linux/amd64    ok
io.containerd.gc.v1             scheduler             -              ok
io.containerd.service.v1        containers-service    -              ok
io.containerd.service.v1        content-service       -              ok
io.containerd.service.v1        diff-service          -              ok
io.containerd.service.v1        images-service        -              ok
io.containerd.service.v1        leases-service        -              ok
io.containerd.service.v1        namespaces-service    -              ok
io.containerd.service.v1        snapshots-service     -              ok
io.containerd.runtime.v1        linux                 linux/amd64    ok
io.containerd.runtime.v2        task                  linux/amd64    ok
io.containerd.monitor.v1        cgroups               linux/amd64    ok
io.containerd.service.v1        tasks-service         -              ok
io.containerd.internal.v1       restart               -              ok
io.containerd.grpc.v1           containers            -              ok
io.containerd.grpc.v1           content               -              ok
io.containerd.grpc.v1           diff                  -              ok
io.containerd.grpc.v1           events                -              ok
io.containerd.grpc.v1           healthcheck           -              ok
io.containerd.grpc.v1           images                -              ok
io.containerd.grpc.v1           leases                -              ok
io.containerd.grpc.v1           namespaces            -              ok
io.containerd.grpc.v1           snapshots             -              ok
io.containerd.grpc.v1           tasks                 -              ok
io.containerd.grpc.v1           version               -              ok
io.containerd.grpc.v1           cri                   linux/amd64    ok
```


### per-cpu memory

https://mp.weixin.qq.com/s/b-Tuj9N0bzdoXGqeRAVxJg?poc_token=HDTACGijwAIwOcU8QvPbrAeLDiXpV5Ozkc9VGTmF

读了下这个, 还是挺有意思的， 没想到已经开始往这个方向开始演化了。也就是说为了避免memory 被频繁的换出换进，正在设计一种专门为 per-process-per-cpu-memory 开始的优化， 简单来说每个进程在每个cpu上跑的时候都去特定位置去读取，避免相关缓存被频繁换进，目前的方向是为每一个cpu对应的每一个进程都划分一块缓存，这样，无论进程被重新调度到那个cpu，这些缓存都可以立即被使用，但是有一个问题， 当一个进程只有四个线程，但是确跑在512 core cpu 的机器上， 这种会造成极大的浪费， 因为每一个cpu都会分配一块缓存，但是进程的并发度太小，根本无法有效利用