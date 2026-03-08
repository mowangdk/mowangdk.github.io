---
layout: post
title:  "weekly report"
date:   2025-11-01 22:30:08 +0800
categories: weeklyreport
---


# 读书
### dm-cache 

https://www.kernel.org/doc/Documentation/device-mapper/cache.txt

最近在准备 fosdem 2026 的 topic，顺便把相关的知识复习下， 

首先 dm-cache 主要接受三个设备
- An origin device - the big, slow one.
- A cache device - the small, fast one
-  A small metadata device - records which blocks are in the cache,
   which are dirty, and extra hints for use by the policy object.
   This information could be put on the cache device, but having it
   separate allows the volume manager to configure it differently,
   e.g. as a mirror for extra robustness.  This metadata device may only
   be used by a single cache device.

整体的缓存模式总体支持三种， 但是正常情况下只会支持两种，第三种更像是紧急状态下使用的一种 mode， 是 cache 的 feature-gate 参数。
值得注意的是，并没有显式write back 的参数， 只有 writeback & writethrough 这两个参数
- writeback, the default, is selected then a write to a block that is
cached will go only to the cache and the block will be marked dirty in
the metadata.
- writethrough, a write to a cached block will not
complete until it has hit both the origin and cache devices.  Clean
blocks should remain clean.

- passthrough useful when the cache contents are not known
to be coherent with the origin device, then all reads are served from
the origin device (all reads miss the cache) and all writes are
forwarded to the origin device

剩下的就是 policy args 了， 用来自定义 block size， 还有下面的 migrate throttling 之类的。

从 origin device 到 cache deivce 的 data migrating 使用了 origin device 的带宽，这个单款是可以限制的。
后面可能会根据正常的io 对 migrating 的io 进行限制


### 元数据处理
- 如果没有 强制 FLUSH（冲刷）或 FUA（强制单元访问）类型的 bio（块IO请求）时 元数据默认会落在cache 里面， 等待 每秒钟一次的 fsync.
- 如果掉电，最多会损失1s内的数据， 虽然有可能丢数据，但是元数据本身的一致性是不会损坏的

> 1. FLUSH (冲刷 / 清洗)
FLUSH 操作就像是执行一次“大扫除”。
机制： 当内核发送一个带有 REQ_PREFLUSH 标志的 bio 时，它是在告诉存储设备：“请把你当前缓存中所有等待写入的数据，立刻刷入物理存储介质（磁盘扇区或闪存颗粒）中。”
执行逻辑：
主机发起请求。
硬盘开始把缓存里堆积的所有数据写到盘片/闪存上。
全部写完后，硬盘返回“成功”给主机。
代价： 开销较大。因为它不仅处理你当前这一条数据，还要处理之前所有堆积在缓存里的数据，会造成短暂的 IO 阻塞。

> 2. FUA (Force Unit Access，强制单元访问)
FUA 操作更像是“专人专送”。
机制： 当内核发送一个带有 REQ_FUA 标志的 bio 时，它是在要求：“这条特定的数据必须直接写到物理介质上，不能在你的缓存里停留。在它真正落盘之前，不要告诉我操作完成了。”
执行逻辑：
主机发起带有 FUA 标志的写请求。
硬盘接收到数据，绕过（或者穿透）缓存，直接写入物理介质。
仅这条数据落盘后，硬盘就返回“成功”。
特点： 它只保证当前这一条 IO 的持久化。相比 FLUSH，如果存储设备支持高效的 FUA，它的性能抖动通常更小。

### dirty metadata 的处理
- 同样，如果实时记录每一个block 是否为脏数据会对性能造成很大的影响， 正常情况下，只有在设备停止的时候才会写入磁盘

### per-block policy hints

相当于实现了一条 LRU 算法，记录每一个快被使用的次数，用于判断整体cache 回收策略， 当然推荐用户设置的每个块大小不宜太大，否则会影响脏页的会写性能跟 cache 和 origin 的回源信道


### policy messaging

不同的策略有不同的可调参数， 这里使用了 device-mapper 的message 信道来实现

example: dmsetup message my_cache 0 sequential_threshold 1024

基本上使用了 SMQ 代替了 MQ, 详细的细节可以参考 https://www.kernel.org/doc/Documentation/device-mapper/cache-policies.txt

这是文档中最具技术含量的部分，解释了为什么 SMQ (Stochastic Multiqueue) 取代了旧的 MQ。

1. 为什么 MQ 不够好？
内存开销大： MQ 每个缓存块需要 88 字节来存指针和计数器。
反应迟钝（Adaptability）： MQ 记录绝对点击量。如果一个块在过去很热（点击 100 万次），现在变冷了，新块需要很久才能超过它的计数并“上位”。
结构不平衡： 大部分块都在低频队列，高频队列几乎是空的，导致缓存空间利用率不佳。
2. SMQ (随机多队列) 的改进（当前默认策略）
极致的内存压缩：
从 88 字节降到了 25 字节。
黑科技： 弃用指针，改用 28 位索引（Index）；不再存储显式的命中计数。
热点区域（Hotspot Queue）：
SMQ 不是以“块”为单位识别热点，而是先以较大的“区域”识别热点。这节省了管理开销。
随机交换（Stochastic Process）：
不再数数： 当一个块命中时，它会与上一级队列中“最久未使用的（LRU）”块进行交换。
效果： 这是一个概率学上的优化。频繁命中的块会自然而然地“浮”到最高层队列。这让它能极快地适应 IO 模式的变化（比如你突然关闭了模型 A，开始加载模型 B）。
三、 Cleaner 策略
功能： 它是一个“只出不进”的特殊策略。
用途： 当你想拆除缓存盘（Decommission）时，切换到 cleaner。它会把缓存盘里所有脏数据（修改过的数据）写回到原始盘，写完后整个缓存就是“干净”的，可以安全移除。


# 工作

## mount bind 挂载点在 mount 标准命令输出的时候看不到子目录

### 一、 为什么 Bind Mount 看不到子挂载点？

这是 Linux **非递归挂载（Non-recursive mount）** 的默认特性。

#### 1. 核心原因：挂载点是隔离的
在 Linux 中，挂载点（Mount Point）是挂在 VFS 树上的一个个“节点”。
当你执行一个标准的 Bind Mount 时：
`mount --bind /source /target`
内核仅仅是将 `/source` 这个**单一挂载点**的内容映射到了 `/target`。

如果 `/source/subdir` 下面还挂载了另一个磁盘（比如 `/dev/sdb`），那么 `/source/subdir` 实际上是原文件系统中的一个**挂载点（Mount Point）**。在执行普通的 bind mount 时，内核不会自动“递归”地去查找并复制这些子挂载点。

#### 2. 结果：
你在 `/target/subdir` 看到的只是 `/source` 原文件系统里那个**空的、作为挂载桩的目录**，而看不到挂载到那个目录上的真实数据。

#### 3. 解决方法：使用 `rbind`
如果你需要看到子目录下的挂载信息，必须使用 **Recursive Bind（递归绑定）**：
`mount --rbind /source /target`
这样内核才会扫描 `/source` 下所有的子挂载点，并在 `/target` 下对应的位置全部建立绑定关系。

---

### 二、 `-o trybind=no` 是什么意思？

这个接口通常不是 Linux 标准 `mount` 命令的通用选项，它常见于 **并行文件系统（如 IBM Spectrum Scale / GPFS）** 或某些 **CSI（容器存储接口）驱动** 的挂载参数中。

在这些特定的存储系统中，它的含义通常如下：

#### 1. 背景：Bind Mount 的局限性
在分布式或复杂文件系统中，直接使用 Linux 的 `mount --bind` 有时会导致：
*   **元数据丢失**：某些文件系统特有的属性（如 Fileset quota）可能无法通过 bind 传递。
*   **状态不一致**：如果源挂载点发生了变化，bind 的目标点有时不能同步感知（除非设置了正确的传播属性）。

#### 2. `trybind` 的逻辑
当驱动程序（如 CSI）尝试将一个目录（比如一个 Fileset）挂载给 Pod 时，它有两种选择：
*   **`trybind=yes` (默认行为)**：驱动程序优先尝试使用 Linux 的 `mount --bind` 命令。这种方式速度快，开销小。
*   **`trybind=no` (强制重新挂载)**：驱动程序**放弃使用 bind 机制**，转而执行一次**全新的、独立的挂载操作（New Mount）**。

#### 3. 为什么要设置 `trybind=no`？
设置 `no` 通常是为了解决以下问题：
*   **可见性问题**：正如你第一个问题所说的，如果源路径下有复杂的子挂载点或特殊的 Fileset 结构，标准的 bind mount 可能会导致子目录不可见。强制 `trybind=no` 可以让系统重新进行一次完整的挂载初始化，确保整个路径树（包括子挂载点）都能正确显现。
*   **权限与隔离**：在某些安全场景下，不希望 Pod 共享宿主机的挂载命名空间状态，重新挂载可以提供更好的隔离性。
*   **特定功能支持**：某些高级功能（如跨集群挂载的特定参数）必须通过独立挂载才能激活，bind mount 无法携带这些参数。

### 总结对照

| 现象/选项 | 行为描述 | 解决的问题 |
| :--- | :--- | :--- |
| **普通 Bind** | 只绑定单一层级，不包含子挂载点。 | 简单的目录映射。 |
| **Recursive Bind (`rbind`)** | 绑定整个树，包含所有子挂载点。 | 解决子目录数据不可见（如果是挂载点）。 |
| **`trybind=no`** | 禁用 bind 机制，强制执行全新的 mount 操作。 | 解决复杂存储架构下的子目录可见性或属性丢失问题。 |

如果你是在配置 **IBM Spectrum Scale (GPFS) 的 CSI 驱动** 或类似的并行文件系统，遇到子目录看不到数据的情况，通常将 `trybind` 设为 `no` 或者改用递归绑定模式是最有效的对策。

# 社区