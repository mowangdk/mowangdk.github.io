---
layout: post
title:  "weekly report"
date:   2025-08-08 22:30:08 +0800
categories: weeklyreport
---


# 读书

### FUSE

2. libfuse 的两种 API：高层 vs. 低层
libfuse 提供了两种不同抽象层次的 API，选择哪一种是文件系统设计的第一个关键决策。

2.1. 高层 API (High-Level)：基于路径
核心思想：函数接口直接接收完整的文件/目录路径作为参数。
- 优点：简单直观，易于上手，适合快速原型开发和简单的代理文件系统。
- 缺点：性能有潜在瓶颈，因为内核每次操作都可能需要文件系统从根路径开始查找。
2.2. 低层 API (Low-Level)：基于 Inode
核心思想：函数接口接收内核分配的 Inode 号码作为参数，更接近 Linux VFS 的工作方式。
- 优点：性能潜力巨大，可充分利用内核的 dentry 和 inode 缓存，避免重复路径查找。
- 缺点：概念更复杂，需要理解 Inode、dentry、lookup 等内核概念。
2.3. 如何选择？
新手或快速原型：选择高层 API。
生产级或高性能文件系统：选择低层 API。
3. FUSE 的基石：fusermount 辅助程序
3.1. fusermount 的角色：安全的特权桥梁
- fusermount (或 fusermount3) 是一个特权辅助程序（通常为 SetUID root）。它负责执行需要 root 权限的 mount() 系统调用，并将与内核通信的 /dev/fuse 文件描述符安全地传递给非特权的 FUSE 进程。

3.2. FUSE 进程的非特权特性
- 正是由于 fusermount 的存在，FUSE 进程本身可以、也强烈推荐作为非特权进程运行。这构成了 FUSE 安全模型的核心。

4. 生产环境中的 FUSE：健壮性与高可用
4.1. 挑战：FUSE 进程重启后的应用访问连续性
- 当 FUSE 进程崩溃时，挂载点会失效，任何访问都会返回 I/O 错误。libfuse 本身不提供自动重连机制。

4.2. 解决方案：systemd + auto_unmount
- 这是实现自动恢复的最佳实践组合：

auto_unmount：在挂载 FUSE 时使用 -o auto_unmount 选项。它能确保在 FUSE 进程终止时，内核自动清理（卸载）挂载点，避免“僵尸”挂载。
systemd 服务：创建一个 systemd 服务来管理 FUSE 进程，并设置 Restart=always。当进程崩溃后，systemd 会自动重启它，重新建立挂载。

4.3. 重启过程是否可以被感知？
是的。自动恢复方案的目标是缩短中断时间，而非完全消除中断。在数秒的恢复窗口期内，用户和应用会感知到中断，表现为 I/O 错误、Stale file handle 或短暂的应用卡顿。健壮的应用程序应包含重试逻辑来平滑度过这个窗口期。

5. FUSE 在云原生（Kubernetes）环境下的实践
5.1. 核心挑战：容器化环境中的挂载与权限
在 Kubernetes 中，容器拥有独立的命名空间和受限的权限，这给 FUSE 挂载带来了新的挑战。

5.2. 标准模式：单一特权容器模型
最直接、最可靠的模式是将 FUSE 应用及其所有依赖（包括 fusermount）打包在同一个容器内，并为该容器授予必要的特权（securityContext.privileged: true 或 capabilities: ["SYS_ADMIN"]）。

5.3. 架构探讨：拆分 fusermount 和 FUSE 应用
一个看似能增强安全性的想法是将特权挂载操作（fusermount）和非特权的文件系统逻辑（FUSE 应用）拆分到不同的容器中。

5.3.1. 核心障碍：文件描述符（FD）的跨容器传递
此方案的致命缺陷在于，文件描述符是进程私有的，无法直接从一个容器中的进程传递给另一个独立容器中的进程。理论上可通过 UNIX Domain Socket 的 SCM_RIGHTS 实现，但极其复杂且脆弱，是一种反模式。

5.4. 进阶模式：Sidecar 作为“挂载守护者”
这是在 Kubernetes 中实现职责分离的最佳模式。

5.4.1. 为何“临时挂载容器”不可行？—— 挂载命名空间的生命周期
一个常见的误区是让一个特权容器（如 Init 容器）执行挂载后就退出。这是行不通的，因为一个挂载点与执行它的进程所在的挂载命名空间绑定。当容器的所有进程退出后，其命名空间会被销毁，导致内核强制卸载其中的所有挂载点，从而使主应用手中的 FD 失效。

5.4.2. 推荐的 Sidecar 模式架构
Pod 包含两个容器：
mounter-sidecar：特权容器，负责执行挂载。
fuse-app-container：非特权容器，运行文件系统逻辑。
mounter-sidecar 的职责：
启动后执行挂载操作。
挂载完成后不能退出，必须通过 sleep infinity 或类似命令保持运行，以维持其挂载命名空间的存活。
fuse-app-container 的职责：
等待 mounter-sidecar 完成挂载（可通过共享卷上的标志文件等方式通信）。
开始运行 FUSE 逻辑。
这种模式完美地结合了职责分离的安全优势和Kubernetes Pod 生命周期的稳定性。

6. 总结与最佳实践
简单场景优先高层 API，生产环境考虑低层 API 以获得极致性能。
始终将 FUSE 进程作为非特权进程运行，充分利用其安全优势。
在非容器化环境中，使用 systemd 配合 auto_unmount 实现高可用。
在 Kubernetes 中，首选“单一特权容器”模型。
若需在 Kubernetes 中实现权限分离，采用“特权 Sidecar 守护者”模式，切勿尝试将挂载组件做成“用完即走”的临时容器。
通过遵循这些原则，开发者可以构建出既功能强大又安全、健壮的 FUSE 文件系统，并将其成功地部署在从传统服务器到现代云原生的各种环境中。


# 社区

### csi livenessprobe

https://github.com/kubernetes-csi/livenessprobe/blob/master/README.md

看了下社区标准的 livenessprobe, 它为了重启 maincontainer 将 livenessprobe 运行在 maincontainer 同时 port 也在 maincontainer 上暴露，而用于提供 server 的 livenessprobe 则运行在单独的 sidecar 容器里面， 这里我的关注点是sidecar 里面的 server 能否监听到 maincontainer 的 pods 上

The fundamental principle of Kubernetes networking is that all containers within a single Pod share the same network namespace.

Think of a Pod as a single, tiny virtual machine or a logical host. This means:

All containers inside the Pod share the same IP address.
All containers can communicate with each other using localhost (e.g., curl localhost:8080).
Crucially, all containers share the same port space.

用了一个demo yaml 测试了下， 确实可以在不同的 container 里面访问到对方的端口
```
➜  demo kubectl exec -it port-success-demo -c nginx-default -- /bin/sh
# curl 127.0.0.1:8080
Hello from nginx-custom on port 8080!#

➜  ~ kubectl exec -it port-success-demo -c nginx-custom -- /bin/sh
Hello from nginx-custom on port 8080!# curl 127.0.0.1:80
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
```

Namespace Type	Shared by Default in a Pod?	Purpose of the Design	How to Share Resources
Network	Yes	Allows containers to communicate easily via localhost and share the same IP address.	N/A (It's already shared)
Mount	No	Ensures container filesystem isolation, preventing dependency and file conflicts.	Using Volumes (e.g., emptyDir, hostPath, PersistentVolumeClaim)
