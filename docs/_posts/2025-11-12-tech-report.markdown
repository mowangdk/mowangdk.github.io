---
layout: post
title:  "weekly report"
date:   2025-11-01 22:30:08 +0800
categories: weeklyreport
---


# 读书

## erofs snapshotter

![snapshotrebase](/assets/img/snapshot_rebase.jpg)
### 背景知识

1.  **Containerd Snapshotter**: `containerd` 的核心组件之一，负责管理容器镜像的文件系统。它将镜像的每一层（layer）作为一个只读的快照（snapshot），然后为容器创建一个可写的快照层，并将它们堆叠起来形成容器的根文件系统。
2.  **镜像分层**: Docker/OCI 镜像是分层的。例如，一个 `ubuntu` 镜像可能包含多个层，每个层都是在前一个层的基础上进行修改。`L1`, `L2`, `L3` 代表镜像的三个层。
3.  **EROFS (Enhanced Read-Only File System)**: 一种由华为开发的、专为只读场景优化的文件系统。在云原生领域，像 Dragonfly 的 `nydus-snapshotter` 和 Ant Group 的 `estargz-snapshotter` 都利用了 EROFS 的思想，实现了镜像的**按需加载（lazy pulling）**。
4.  **Active vs. Committed**:
    *   **Committed (已提交)**: 表示一个已经下载、解压、并固化为文件系统层的只读快照。在图中用蓝色节点 `P` (Parent) 表示。`P1` 是 `P2` 的父层，`P2` 是 `P3` 的父层。
    *   **Active (活动中)**: 表示一个正在被操作的、临时的可写快照。在图中用红色节点 `L` (Layer) 表示。这是解压和准备镜像层时的工作空间。

---

### 左图：传统的串行拉取过程 (The Old Way)

左边的图展示了 `containerd` 默认的、未经优化的镜像拉取流程。这是一个**严格串行**的过程，必须等一个层完全处理完，才能开始处理下一个层。

我们按照编号顺序来解析：

1.  **1. Prepare (L₂, parent=P₁)**:
    *   **操作**: `containerd` 准备开始处理第二层 (`L₂`)。它创建了一个临时的**活动快照 `L₂`**。
    *   **关键**: `parent=P₁`。这个活动快照 `L₂` 是基于**已提交的父层 `P₁`** 创建的。这意味着在 `L₂` 上的所有文件修改都是相对于 `P₁` 的。

2.  **2. Unpack (1 GiB)**:
    *   **操作**: 将第二层的压缩包（1 GiB）解压到活动快照 `L₂` 中。这是一个耗时的 I/O 和 CPU 操作。

3.  **3. Commit (L₂, P₂)**:
    *   **操作**: 解压完成后，将活动快照 `L₂` **提交**为一个新的**只读快照 `P₂`**。`P₂` 现在成为了文件系统链条的一部分。活动快照 `L₂` 被销毁。

4.  **4. Prepare (L₃, parent=P₂)**:
    *   **操作**: 系统现在开始处理第三层 (`L₃`)。它创建了一个新的活动快照 `L₃`。
    *   **关键**: `parent=P₂`。这个新快照是基于**刚刚提交的 `P₂`** 创建的。**这就是串行的瓶颈所在**：必须等待 `P₂` 完全准备好，才能开始 `L₃` 的准备工作。

5.  **5. Unpack (2 GiB)**:
    *   **操作**: 将第三层的压缩包（2 GiB）解压到活动快照 `L₃` 中。

6.  **6. Commit (L₃, P₃)**:
    *   **操作**: 将活动快照 `L₃` 提交为新的只读快照 `P₃`。

**左图总结**:
*   **依赖链**: `L₃` 的准备依赖于 `P₂` 的提交，`P₂` 的提交依赖于 `L₂` 的解压，`L₂` 的准备依赖于 `P₁` 的提交。
*   **总耗时 (理论上)**: `Time(L₂) + Time(L₃)`，因为它们是完全串行的。
*   **问题**: 无法利用多核 CPU 和高 I/O 带宽来并行处理多个层，拉取大镜像时效率低下。

---

### 右图：优化的并行拉取过程 (Snapshot Rebase)

右边的图展示了引入 "Snapshot Rebase" 技术（这是 EROFS-based snapshotter 的核心优化）后的流程。

我们再次按编号顺序解析：

1.  **1. Parallel Unpacking (并行解压)**:
    *   **操作**: 这是最关键的改变。系统**同时**开始处理 `L₂` 和 `L₃`。
    *   **Prepare (L₂, parent=Empty)** 和 **Prepare (L₃, parent=Empty)**: 注意这里的 `parent=Empty`！系统为 `L₂` 和 `L₃` 创建的活动快照，都是**基于一个空的父层**。它们是**相互独立的**，不依赖于任何已提交的层。
    *   **Unpack**: `L₂` (1 GiB) 和 `L₃` (2 GiB) 的解压操作可以**并行进行**。如果系统资源充足，它们可以同时运行，充分利用 I/O 和 CPU。

2.  **2. Commit (L₂, P₂, parent=P₁)**:
    *   **操作**: `L₂` 的解压先完成了。现在到了提交环节。
    *   **关键 (Rebase)**: 在提交时，系统执行了一个 **"Rebase"（变基）** 操作。它告诉 `containerd`：“虽然我这个活动快照 `L₂` 是基于空目录解压的，但现在请你将它提交为一个**新的只读快照 `P₂`，并将它的父层设置为 `P₁`**。”
    *   `containerd` 的快照机制（通常是 `overlayfs`）会记录下这个新的父子关系。它并没有真正地去合并文件，只是在元数据层面将 `P₂` 的 "parent" 指针指向了 `P₁`。

3.  **3. Commit (L₃, P₃, parent=P₂)**:
    *   **操作**: `L₃` 的解压也完成了。现在轮到它提交。
    *   **关键 (Rebase)**: 同样地，系统执行 Rebase 操作，将基于空目录解压的 `L₃` 提交为一个新的只读快照 `P₃`，并**在此时才指定它的父层是 `P₂`**。

**右图总结**:
*   **解耦**: "Snapshot Rebase" 技术将**解压（Unpack）**过程与**父子关系建立（Commit）**过程进行了解耦。
*   **并行化**: 解压过程可以完全并行，因为它们在逻辑上是独立的（`parent=Empty`）。
*   **延迟绑定**: 镜像层之间的父子依赖关系，被**推迟**到了最后的 `Commit` 阶段才进行绑定。
*   **总耗时 (理论上)**: `Max(Time(L₂), Time(L₃))`。因为它们是并行执行的，总时间取决于最慢的那个。

### 结论

这张图的核心思想是：

*   **传统方式**: `Prepare -> Unpack -> Commit -> (等待) -> Prepare -> Unpack -> Commit` (串行)
*   **Rebase 优化**: `(Prepare+Unpack) || (Prepare+Unpack) -> Commit -> Commit` (并行解压，串行提交)

通过将耗时最长的 **Unpack（解压）** 过程并行化，"Snapshot Rebase" 技术可以显著缩短多层镜像的总体拉取和准备时间。这是 EROFS-based snapshotter (如 Nydus) 相比于传统 `overlayfs` snapshotter 在性能上的一个巨大优势。它通过欺骗（或者说，巧妙地利用）`containerd` 的快照机制，打破了原有的串行依赖，实现了“先并行干活，最后再整理关系”的高效模式。


## ESTALE(Stale File Handle)




# 工作

### agent sleep & resume

最近最火的莫过于 ai agent 莫属了， 估计各大云厂商都在琢磨怎么样才能把这套东西玩起来， 尤其是在容器上面玩起来， 包括社区也借助 crid 实现了进程级别的 [checkpkpoint & restore](https://kubernetes.io/docs/reference/node/kubelet-checkpoint-api/) 当然这个原本做的目的是为了debug。当然文档里面也写了， 这个其实还是有很多安全问题的， 比如这个checkpoint 会保留container 里面所有进程的数据，其中包括内存中的，可能会涉及到一些aksk之类的东西。没有详细看过实现， 不清楚是否会保存pod引用的pvc/pv 的数据啥的。不知道后面是否会有相关任务。把最近一周聊过的东西丢给了AI， 让他帮忙整理了一篇说明，随意看看

---

云原生运行时快照：从环境复现到 AI Agent“浅休眠”的关键技术
在云原生时代，应用的弹性伸缩和快速启动能力至关重要。而随着 AI Agent 等长时运行、有状态应用的兴起，一种新的需求变得日益迫切：如何让复杂的计算任务在暂停后，能以极低的成本和极快的速度恢复其完整的上下文？

运行时快照技术，正从一个最初专注于复现用户特定环境以排查疑难 Bug 的“小众”领域，演变为解决上述问题的核心。无论是为 Serverless 应用提供极致的启动速度，还是让 AI Agent 实现高效的“浅休眠”（Shallow Sleep），其底层逻辑都是一致的：捕获并快速恢复一个包含了“热”状态的运行时环境。

本文将阐述一个核心工作流：通过文件级净化与审计、CRIU 进程固化与 ** 快照存储分发**，构建一套能够支撑从传统应用到未来智能体的、安全高效的应用运行时快照方案。

第一步：精心雕琢与审计运行时模板
创建快照的第一步，不是直接对运行中的环境进行“一刀切”，而是要精心“雕琢”一个理想的快照源。这正是该方案区别于传统“黑盒”式快照的核心优势所在——它将一个不透明的运行时环境，转变为一个完全透明、可审计、可控的“白盒”式软件制品。

净化与瘦身:
清除敏感信息: 运行中的进程内存和临时文件中，可能残留着访问密钥（Token）、用户数据或其他机密信息。在制作快照前，必须通过自动化的脚本或规范的流程，确保这些信息已被清除。
剥离非必要资产: 现代应用常常包含巨大的外部依赖，例如 AI 应用中的模型文件。这些文件体积庞大且更新独立，不应被打包进快照。正确的做法是将其剥离，待实例恢复后再通过共享存储等方式挂载。
移除临时文件: 清理调试日志（debug log）、临时缓存等运行时产生但无需持久化的文件。
文件级审计:
在完成初步净化后，必须对模板中的所有文件进行严格的审计。这是确保快照技术从一个单纯追求速度的“技巧”，演变为符合现代 DevSecOps 标准的、安全可靠工程化“规范”的关键。
安全性审计: 可以通过静态扫描工具对所有文件进行漏洞扫描（Vulnerability Scanning），确保制品中不包含带有已知安全漏洞的依赖库。
合规性检查: 能够检查文件内容，确保其中不包含密码、密钥、个人身份信息（PII）等敏感数据，从而满足 GDPR、数据安全法等合规要求。
只有经过了净化和审计的双重处理，我们才得到了一个真正安全、合规且轻量化的快照源。

第二步：进程状态固化——CRIU 的快照魔法
在获得一个“干净”的应用文件系统后，下一步是固化应用进程的“热”状态。这正是 CRIU (Checkpoint and Restore In Userspace) 发挥作用的时刻。

在此工作流中，当应用完成初始化或 AI Agent 完成一轮复杂的思考后，我们调用 CRIU 对其执行 dump 操作。CRIU 生成的状态文件与前一步净化审计后的应用文件系统相结合，共同构成了一个完整的“运行时模板”。

第三步：高性能分发与恢复——pangu2.0 log-structured 的力量
拥有了“运行时模板”后，如何将其高效、快速地分发并在需要时瞬时启动，是决定方案成败的关键。这正是 pangu2.0 log-structured（RoW）快照技术展现其威力的地方。

模板播种 (Seeding):
管理员或自动化系统将制作好的“运行时模板”一次性地写入到一个由 pangu2.0 log-structured 存储系统管理的只读基础卷（Base Volume）中，并预先分发到目标节点。
瞬时实例化 (Instantiation):
当需要启动一个新应用实例时，系统无需耗时地复制整个模板，而是向底层存储发出一个 ** 克隆指令，在毫秒级时间内生成一个全新的可写快照卷**。
新创建的容器或微虚拟机直接挂载这个可写快照卷，然后调用 CRIU 执行 restore 操作，应用瞬间恢复到被“冻结”时的状态，立即可用。
更重要的是，与传统的写时复制（CoW）快照技术不同，pangu2.0 log-structured 机制的核心优势在于，即使在一个任务中进行多次连续快照，也不会像快照链增长那样导致性能衰减。这一点对于需要频繁保存状态的长时运行任务（如 AI Agent 在复杂推理中保存中间上下文）至关重要，它能确保在整个生命周期内，Agent 进行读写操作的性能始终保持稳定。

第四步：OCI 制品分发
虽然原生快照在区域内性能卓越，但它与特定的存储系统绑定，缺乏跨云、跨环境的通用性。为了解决这个问题，我们希望可能可以引入一个可选的、解耦的转换机制，而不是将其作为强制的性能瓶颈。
● 异步转换: 一个后台任务可以被触发，用于将之前生成的原生快照“物化”（Materialize）为一个标准的 OCI 制品。该任务会挂载快照，将其中的文件内容打包成一个 OCI 镜像层，并推送到镜像仓库。
● 应用场景: 转换后的 OCI 制品主要用于：
● 跨区域或跨云迁移。
● 版本化归档与长期存储。
● 与现有 CI/CD 工具链和安全扫描生态的集成。


### cgroup in cgroup

尝试在容器里面创建 cgroup， 发现相关cgroup 会放到宿主机上, 看起来 cgroupv1 是共用的， cgroupv2 有独立的 cgroup namespace 之后就是相对于宿主机的, 不过默认应该是只挂载当前容器的 cgroup 路径的（也就是看不到其他 pod 的 cgroup）

[~]$ ls /sys/fs/cgroup/kubepods.slice/kubepods-besteffort.slice/kubepods-besteffort-xxxx.slice/cri-containerd-xxxx.scope
cgroup.controllers       cpu.group_balancer        cpuset.mems.effective     hugetlb.2MB.events.local  memory.current                        memory.min                     memory.use_priority_swap
cgroup.events            cpu.ht_ratio              cpu.soft_cpus             hugetlb.2MB.max           memory.direct_compact_latency         memory.numa_stat               memory.wmark_high
cgroup.freeze            cpu.identity              cpu.specs_ratio           hugetlb.2MB.rsvd.current  memory.direct_reclaim_global_latency  memory.oom.group               memory.wmark_low
cgroup.max.depth         cpu.idle                  cpu.stat                  hugetlb.2MB.rsvd.max      memory.direct_reclaim_memcg_latency   memory.pagecache_limit.enable  memory.wmark_min_adj
cgroup.max.descendants   cpu.ioblock_latency       cpu.wait_latency          ioasids.current           memory.direct_swapin_latency          memory.pagecache_limit.size    memory.wmark_ratio
cgroup.procs             cpu.max                   cpu.weight                ioasids.events            memory.direct_swapout_global_latency  memory.pagecache_limit.sync    memory.wmark_scale_factor
cgroup.stat              cpu.max.burst             cpu.weight.nice           ioasids.max               memory.direct_swapout_memcg_latency   memory.pressure                my-sub-group
cgroup.subtree_control   cpu.max.init_buffer       hugetlb.1GB.current       io.bfq.weight             memory.events                         memory.priority                pids.current
cgroup.threads           cpu.pressure              hugetlb.1GB.events        io.extstat                memory.events.local                   memory.reap_background         pids.events
cgroup.type              cpu.priority              hugetlb.1GB.events.local  io.latency                memory.exstat                         memory.stat                    pids.max
cpu.block_latency        cpu.sched_cfs_statistics  hugetlb.1GB.max           io.max                    memory.high                           memory.swap.current            rdma.current
cpu.bvt_warp_ns          cpuset.cpus               hugetlb.1GB.rsvd.current  io.pressure               memory.idle_page_stats                memory.swap.events             rdma.max
cpu.cgroup_wait_latency  cpuset.cpus.effective     hugetlb.1GB.rsvd.max      io.stat                   memory.idle_page_stats.local          memory.swap.high
cpu.enable_sli           cpuset.cpus.partition     hugetlb.2MB.current       io.weight                 memory.low                            memory.swap.max
cpu.exstat               cpuset.mems               hugetlb.2MB.events        memory.async_fork         memory.max                            memory.use_priority_oom

[~]$ ls /sys/fs/cgroup/kubepods.slice/kubepods-besteffort.slice/kubepods-besteffort-podxxxxx.slice/cri-containerd-xxxx.scope
cgroup.controllers       cpu.group_balancer        cpuset.mems.effective     hugetlb.2MB.events.local  memory.current                        memory.min                     memory.use_priority_swap
cgroup.events            cpu.ht_ratio              cpu.soft_cpus             hugetlb.2MB.max           memory.direct_compact_latency         memory.numa_stat               memory.wmark_high
cgroup.freeze            cpu.identity              cpu.specs_ratio           hugetlb.2MB.rsvd.current  memory.direct_reclaim_global_latency  memory.oom.group               memory.wmark_low
cgroup.max.depth         cpu.idle                  cpu.stat                  hugetlb.2MB.rsvd.max      memory.direct_reclaim_memcg_latency   memory.pagecache_limit.enable  memory.wmark_min_adj
cgroup.max.descendants   cpu.ioblock_latency       cpu.wait_latency          ioasids.current           memory.direct_swapin_latency          memory.pagecache_limit.size    memory.wmark_ratio
cgroup.procs             cpu.max                   cpu.weight                ioasids.events            memory.direct_swapout_global_latency  memory.pagecache_limit.sync    memory.wmark_scale_factor
cgroup.stat              cpu.max.burst             cpu.weight.nice           ioasids.max               memory.direct_swapout_memcg_latency   memory.pressure                my-sub-group
cgroup.subtree_control   cpu.max.init_buffer       hugetlb.1GB.current       io.bfq.weight             memory.events                         memory.priority                pids.current
cgroup.threads           cpu.pressure              hugetlb.1GB.events        io.extstat                memory.events.local                   memory.reap_background         pids.events
cgroup.type              cpu.priority              hugetlb.1GB.events.local  io.latency                memory.exstat                         memory.stat                    pids.max
cpu.block_latency        cpu.sched_cfs_statistics  hugetlb.1GB.max           io.max                    memory.high                           memory.swap.current            rdma.current
cpu.bvt_warp_ns          cpuset.cpus               hugetlb.1GB.rsvd.current  io.pressure               memory.idle_page_stats                memory.swap.events             rdma.max
cpu.cgroup_wait_latency  cpuset.cpus.effective     hugetlb.1GB.rsvd.max      io.stat                   memory.idle_page_stats.local          memory.swap.high
cpu.enable_sli           cpuset.cpus.partition     hugetlb.2MB.current       io.weight                 memory.low                            memory.swap.max
cpu.exstat               cpuset.mems               hugetlb.2MB.events        memory.async_fork         memory.max                            memory.use_priority_oom



