---
layout: post
title:  "weekly report"
date:   2025-03-01 22:30:08 +0800
categories: weeklyreport
---


# 读书

### 3FS

ds 昨天开源了 [3FS](https://github.com/deepseek-ai/3FS), 因为我本身不是做分布式存储的，之前也没有类似的经验。所以也只是简单先看下文档，然后熟悉下流程，具体的事情交由专业的人去干吧，先了解下整体架构

整体架构跟 CEPH 还是比较类似的，不过也可以理解，毕竟是经典架构

- cluster manager
- metadata service
- storage service
- client

metadata & storage service 向 cluster manager 发送心跳，cluster manager 处理变更，本身有主从架构进行高可用，进行配置分发等操作。集群配置存储在 etcd or zookeeper 中。其中 metadata service 实现了文件语义，像是 open 或者 createfile 等语义都可以进行处理。metadata 本身是无状态的，并且有多副本保证可用性。 相关元数据信息都存储在 KV store 里面（FoundationDB）

storage service 负责管理本地 SSD，对外提供一个 chunk store interface, 这个服务实现了 CRAQ(Chain Replication With Apportioned Queries) 来保证强一致性, 一个 3FS 的文件被分为大小相等的块这些块被复制到多个固态硬盘上

支持两种客户端， 一个是 fuse，一个是 nativeclient


然后当然现在大部分ai存储系统都是基于对象存储的，不过 oss 自然有其缺陷, 3FS 认为主要有三点是他们选择直接做一套新的东西而不是复用现有的oss存储方案。	

- Atomic directory manipulation
object store 虽然可以模拟文件系统树，但是它并不支持原子移动文件 or 目录等操作。对于大批量小文件的操作有着非常大的性能问题

- Symbolic and hard links 
他们想要用符号链接和硬链接来做快照，这个还是在object 里面解决不了.

- Familiar interface
文件接口广为人知，在任何地方都能使用。无需学习新的存储 API

怎么说，以上三个点基本上都是老生常谈了，没啥特别的。

然后就到了感兴趣的地方， 异步0拷贝API, 基本上他们是在 fuse 里面实现了一个native client，这个client 提供了一个interface来做异步零拷贝IO， 元数据的操作依旧通过 fuse, 简单的使用方式就是， 用户通过 open 标准方法打开一个文件的 fd， 并且通过一个native api 把它注册到 3FS 上， 然后使用 native client 直接对fd 进行 IO 操作

异步零拷贝灵感来源于 linux io_uring. 在 3FS 里面主要分为两个部分

- lov: 
用于零拷贝读/写操作的大型内存区域，由用户进程和本地客户端共享。InfiniBand 内存注册由客户端管理。在本地 API 中，所有读取数据都将读入 Iov，所有写入数据都应在调用 API 前写入 Iov。

- lor:
一个小型共享环形缓冲区，用于用户进程与本地客户端之间的通信。lor 的用法类似于 Linux 的 io_uring，即用户进程排队等待读/写请求，本地客户端排队等待完成这些请求。请求分批执行，其大小由 io_depth 参数控制。多个批次可并行处理，无论是来自不同环还是同一环。不过，多线程应用程序仍建议使用多个环，因为共享一个环需要同步，这会影响性能。

在本地客户端中，会生成多个线程，以便从 lors 获取 I/O 请求。这些请求被分批分派给存储服务，从而减少了小型读取请求造成的 RPC 开销。


#### File metadata store

创建新文件时，元数据服务会采用循环策略，根据条带大小从指定的链表中选择连续的复制链。然后，生成一个随机种子，对选中的链进行洗牌。这种分配策略可确保数据在链和固态硬盘上的均衡分布。

当应用程序打开文件时，客户端会联系元服务以获取文件的数据布局信息。然后，客户端可以独立计算用于数据操作的块 ID 和链，从而最大限度地减少元服务在关键路径中的参与。

文件系统的元数据主要包含两种主要的数据结构， inodes & directory entries

Inodes:

inode 数据结构存储在 FoundationDB 里面，Key 是以 INOD prefix 开头并且后面加上 inode id（identified by a globally unique 64-bit identifier that increments monotonically）
value 包含以下元素
- ownership, permissions, access/modification/change times
- filelength, chunk-size, selected-range in chain table, shuffle seed
- parent directory inode's id, default layout configurations for subdirectories/files(chain table， chunk size, strip size)
- 对于软链来说， 还要在value里面存储target path


Directory entry:

Key 是由 DENT prefix 开头并且后面加上，所在的父目录的 inode id, 和entry name 共同组成的, Value 则主要存储 inode id 和 inode type. 一个目录里面所有的条目形成一个连续的键范围，这样可以更高效的列出目录
对于元数据的操作则利用了 FoundationDB 的事务。 fsstat, lookup, listdir 使用只读事务， create, link, unlink, rename 使用读写事务。对于读写事务，这里采用了类似乐观锁的机制，当发生冲突的是时候， 
基于 FoundationDB 的事务会自动重试 



Dynamic file attribute

大多数本地文件系统都会跟踪被删除的文件， 直到所有打开它的 fd 都关掉, 实现这个功能需要追踪所有打开的 fd . 但是这个追踪会对 metadata service 带来很大的性能损耗，并且这个其实在训练过程中不会出现, 所以 3FS 在实现上
不会最终以只读方式打开的fd。 当然对于以可写模式打开的文件， 3FS 还是会追踪的。 文件的大小被记录在inode 数据结构中， 当用户写文件的时候， 这个大小会被更新， 这个是由 client 端进行的周期性更新。由于有可能是多个client端共同更新。
所以这个大小只能保证最终一致，当client 端 close or fsync 动作的时候。 元服务会从storage service中查询最后一个数据块的 ID 和长度，从而获得精确的文件长度。由于数据是在多个链上进行条带化处理的，所以这个操作会产生很大的开销


#### Chunk storage service

chunk 主要被设计用来进行容灾， 3FS 使用了 chain replication with apportioned queries ，也就是每个file chunk写入都写入链表头部，他会被自动复制到链表中的其他元素，读取可以从任意一个链表元素中读取， 一般这些元素会分布在多个存储节点(SSD)上。

当一个写请求被 storage service 接收的时候， 它执行了如下步骤

- 检查写入请求中的链版本是否与最新的已知版本一致；如果不一致，则拒绝请求。写入请求可能是由客户端或链中的前任发送的。
- 服务会发出 RDMA 读操作，以提取写入数据。如果客户机/前置机出现故障，RDMA 读操作可能会超时，写操作也会中止。
- 一旦写入的数据被获取到本地内存缓冲区，就会从锁管理器获取要更新的数据块的锁。对同一数据块的并发写入将被阻塞。所有写入都会在头部目标序列化。
- 服务会将已提交版本的数据块读入内存，更新，并将更新后的数据块存储为待处理版本。存储目标可能会存储一个数据块的两个版本：已提交版本和待处理版本。每个版本都有一个单调递增的版本号。已提交版本和待处理版本的版本号分别为 v 和 u，并满足 u = v + 1。
- 如果服务是尾部服务，则已提交的版本会被待处理版本原子替换，并向前置服务发送确认消息。否则，写入请求将转发给后续服务。更新已提交版本时，当前链版本会作为一个字段存储在大块元数据中。
- 当确认信息到达存储服务时，服务会用待处理版本替换已提交版本，并继续将信息传播到其前置版本。然后，本地块锁被释放。


剩下关于 storage service 的内容主要就是关于，故障处理， 故障恢复之类的东西， 暂时不做深入了解



