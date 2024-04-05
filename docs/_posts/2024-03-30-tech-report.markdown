---
layout: post
title:  "weekly report"
date:   2024-03-30 22:30:08 +0800
categories: weeklyreport
---


# 读书


## nvme (https://nvmexpress.org/wp-content/uploads/NVM-Express-Base-Specification-2_0-2021.06.02-Ratified-4.pdf)

继续看了几章, 感觉都是介绍相关 register 能力的. 就没有详细看,毕竟我也不是专门开发的. 再往后看看吧


# 社区

### csi sidecar AIO

这周照常开会, 谈论了一下 kubecon 和 feature management 的问题, kubecon 基本上没啥问题, 我这边准备开始写提案了. feature management 也确定了大致方向, 但是还是有很多东西需要进一步确认, 准备这两天写一个文档大家一起对齐一下.

# 工作

### overlayBD vs overlayFS

OverlayFS（Overlay File System）是一种联合文件系统（union filesystem），它允许将多个目录叠加在一起，形成一个单一的统一视图。在OverlayFS中，有两个主要概念：下层（lower layer）和上层（upper layer）。下层通常是只读的，而上层则是可写的。当对文件进行修改时，这些更改会发生在上层，而不会影响下层。这个特性使得OverlayFS非常适合于容器技术，特别是在Docker和其他容器平台中，用于制作轻量级、可共享的容器镜像。

OverlayBD 是 dadi 创建的一个专属架构, 基本上就是 overlayFS 在 块存储上的一种实现吧, 将多个块设备透明的合并到一个块设备的机制. 同样的, 只支持最上层的块设备进行读写.

### os kernel vs os rootfs

OS Kernel（操作系统内核）：

内核是操作系统的核心部分，它管理着计算机的硬件资源，并为上层应用程序提供基本的服务。内核充当应用程序与硬件之间的中介，负责处理进程管理、内存管理、设备驱动程序、系统调用和安全等多个底层任务。负责处理与硬件相关的复杂任务，为用户空间程序提供一个安全、稳定的运行环境。

OS RootFS（操作系统根文件系统）：

根文件系统（rootfs）是包含操作系统所有文件的文件系统。在Unix和类Unix系统中，它被挂载在根目录 / 下。根文件系统包含了所有的目录和文件，包括用户数据、应用程序、系统配置文件、系统库（libraries）、二进制可执行文件等。是操作系统的文件存储结构，它是构成操作系统的文件和目录的一个集合，提供了必要的用户级别应用和库。
在容器技术中，这两个概念也经常出现。容器通常不包含完整的操作系统内核，因为它们运行在宿主机的内核上。但是，容器确实包含了一个 rootfs，它提供了必须的用户空间组件，让容器内的应用程序可以运行。


### mount helper

在 Unix 和类 Unix 操作系统中，特别是在 Linux 上，一个 "mount helper" 是一个辅助程序，它用于简化和支持文件系统挂载操作。当您尝试挂载某种类型的文件系统时，mount helper 作为 mount 命令的扩展，提供特定文件系统类型的额外支持和选项。

例如，挂载一个文件系统时，您可能会使用以下命令：

mount -t type device directory
在这个命令中：

-t type：指定要挂载的文件系统类型。
device：指定文件系统的设备或文件。
directory：指定挂载点的目录。
如果指定的文件系统类型有一个对应的 mount helper，那么 mount 命令会调用这个 helper 程序来处理挂载过程。这个机制允许 mount 命令对多种文件系统类型具有灵活性，并且可以通过安装额外的软件包来支持新的文件系统类型，而无需修改 mount 命令本身。

举个例子，在挂载一个 NFS（Network File System）文件系统时，mount 命令可能会调用 mount.nfs 这个 helper：

mount -t nfs server:/path /local/mountpoint
在这种情况下，mount.nfs helper 会处理 NFS 相关的挂载选项和网络通信，而 mount 命令负责设置和管理挂载点。

mount helper 程序通常位于 /sbin 或 /usr/sbin 目录中，并且它们的命名通常以 mount. 开头，后跟文件系统的类型，例如 mount.nfs、mount.cifs（用于挂载 Windows 共享）等。

###  LC_ALL: cannot change locale

切换了宿主机镜像之后出现的问题, 一般来说就是因为容器内默认的 locale 和宿主机上的 locale 不一致. 导致的问题, 当然可以在容器内设置宿主机上存在的 locale, 不过考虑到可移植性的问题, 也不是一个非常好的解决方法

### rg command

全称 Rapid Golfing Regular Expressions, 是一个命令行工具, 用于在文本文件中搜寻指定的字符串, 是 grep 的替代品. 在某些情况下更快更易用.
