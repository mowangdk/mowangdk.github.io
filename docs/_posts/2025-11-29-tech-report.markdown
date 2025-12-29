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

### dirty metadata 的处理
- 同样，如果实时记录每一个block 是否为脏数据会对性能造成很大的影响， 正常情况下，只有在设备停止的时候才会写入磁盘

### per-block policy hints

相当于实现了一条 LRU 算法，记录每一个快被使用的次数，用于判断整体cache 回收策略， 当然推荐用户设置的每个块大小不宜太大，否则会影响脏页的会写性能跟 cache 和 origin 的回源信道


### policy messaging

不同的策略有不同的可调参数， 这里使用了 device-mapper 的message 信道来实现



# 工作

# 社区