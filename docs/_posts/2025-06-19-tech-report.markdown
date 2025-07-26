---
layout: post
title:  "weekly report"
date:   2025-06-06 22:30:08 +0800
categories: weeklyreport
---


# 读书





# 社区

### volume mutable nodeaffinity
https://github.com/kubernetes/enhancements/pull/5382

The main point is we delegate the validation of migrate to SP， As a CO, what we need to do is gurantee the proper behave when SP return errors or CO races

As for race condition， we add a new field ```in_progress``` to ControllerModifyVolume, 
Before altering the topology to an unfeasible state, consider adding clearer guidance on handling scheduled pods.


### CSI AIO


   

# 工作

### THP(Transparent Huge Pages)

什么是透明大页 (THP)？
透明大页（THP）是 Linux 内核的一项功能，它通过将物理内存中连续的常规页面（通常为 4KB）透明地替换为更大的内存块（大页，通常为 2MB），来优化内存管理。这种机制减少了页表条目数量，从而降低了 TLB（Translation Lookaside Buffer）压力，并可能减少内存碎片，整体上降低了内存管理开销 2。

THP 对 Go 程序的潜在影响
Go 语言程序可以透明地受益于 THP，这意味着通常不需要修改 Go 代码。

性能提升：对于具有大堆内存（1 GiB 或更多）的 Go 应用程序，启用 THP 可以改善吞吐量（最多 10%）和延迟，同时额外内存开销较小（1-2% 或更少）。这是因为 THP 减少了内存访问的开销 2。
内存开销：对于堆内存较小的 Go 应用程序，THP 可能无法带来性能收益，反而可能导致显著的额外内存使用（高达 50%）。这是因为即使应用程序只需要较小的内存区域，THP 也会分配一个完整的 2MB 大页，可能造成内存浪费 2。
潜在延迟：尽管 THP 带来性能优势，但它也可能在许多应用程序中引入延迟 2。
Go 运行时与 THP 的交互
Go 运行时的内存管理策略会与 THP 交互。

Go 运行时过去和现在的问题：Go 运行时处理大页的相关策略曾存在问题，导致小堆内存应用出现高达 40% 的内存过量使用，而大堆内存应用则可能因部分区域错误地被标记为 MADV_NOHUGEPAGE 而损失高达 1% 的吞吐量 1。
Go 运行时改进目标：Go 社区的目标是消除小堆内存的内存过量使用，并提高大堆内存的利用率 1。这包括根据内存分配器块的未来密度来决定哪些堆区域应由大页支持，并调整后台清除器（scavenger）的行为 1。
Go 程序的 THP 配置建议
在生产环境中为 Go 程序启用 THP 时，建议进行以下额外设置以优化其行为：

- 设置 /sys/kernel/mm/transparent_hugepage/enabled 为 madvise：
   madvise 模式意味着内核仅在应用程序通过 madvise(MADV_HUGEPAGE) 明确请求时，才考虑对该内存区域使用大页 3。这提供了更精细的控制，避免了不必要的内存浪费。
- 设置 /sys/kernel/mm/transparent_hugepage/defrag 为 defer 或 defer+madvise：
   此设置控制 Linux 内核将常规页面合并为大页的积极程度 
   defer 模式指示内核以延迟和后台方式合并大页，有助于避免在内存受限系统上引入停顿 
   defer+madvise 模式在此基础上，对明确请求大页的其他应用程序更为友好 
- 设置 /sys/kernel/mm/transparent_hugepage/khugepaged/max_ptes_none 为 0：
   这个设置限制了 khugepaged 守护进程在分配大页时可以分配的额外页面数量。默认设置可能过于激进，会抵消 Go 运行时向操作系统归还内存的工作, 对于 Go 1.21 及更高版本和 Linux 6.2 及更高版本，Go 运行时不再直接修改大页状态。如果升级到 Go 1.21.1 或更高版本后内存使用增加，应用此设置可能解决问题

替代方案：可以通过 Go 程序的 Prctl 函数调用 PR_SET_THP_DISABLE 来在进程级别禁用大页，或者在 Go 1.21.6 和 Go 1.22 中使用 GODEBUG=disablethp=1 环境变量来禁用堆内存的大页（此 GODEBUG 设置未来可能被移除）

结论与建议
THP 并非适用于所有 Go 程序的通用优化。它对于内存密集型且具有大堆内存的应用程序可能带来显著收益，但对于小堆内存应用程序则可能导致额外内存消耗。因此，始终建议进行实验和基准测试，以了解 THP 对您的特定 Go 应用程序的实际影响 1。同时，密切监控内存使用情况和性能指标至关重要。

简单的example:

```golang
package main

import (
	"fmt"
	"runtime"
	"time"
)

const (
	// 定义一个较大的内存块大小，例如 1GB
	// 这样可以确保 Go 程序的堆内存足够大，从而可能从 THP 中受益
	MEM_BLOCK_SIZE = 1 * 1024 * 1024 * 1024 // 1 GB
)

func main() {
	fmt.Println("Go 程序启动，尝试分配并使用大量内存...")

	// 分配一个大的字节切片
	// 这将在 Go 运行时中分配一个大的堆区域
	largeBuffer := make([]byte, MEM_BLOCK_SIZE)

	// 简单地填充这个切片，确保其被访问和“使用”，
	// 避免编译器或运行时优化掉未使用的内存分配。
	// 这有助于确保这部分内存被认为是“活跃”的。
	for i := 0; i < MEM_BLOCK_SIZE; i++ {
		largeBuffer[i] = byte(i % 256)
	}

	fmt.Printf("已分配并初始化 %d 字节（约 %d GB）的内存。\n", MEM_BLOCK_SIZE, MEM_BLOCK_SIZE/1024/1024/1024)

	// 打印当前内存统计信息
	var m runtime.MemStats
	runtime.ReadMemStats(&m)
	fmt.Printf("当前 Go 运行时内存统计：\n")
	fmt.Printf("  HeapSys: %d B (从操作系统获取的堆内存)\n", m.HeapSys)
	fmt.Printf("  HeapAlloc: %d B (当前分配的堆对象)\n", m.HeapAlloc)
	fmt.Printf("  HeapInuse: %d B (正在使用的堆内存)\n", m.HeapInuse)
	fmt.Printf("  NumGC: %d (GC 循环次数)\n", m.NumGC)

	fmt.Println("\n程序将保持运行一分钟，以便观察系统内存使用情况...")

	// 保持程序运行一段时间，以便操作系统和 Go 运行时有时间处理内存
	// 并在 /proc/<pid>/smaps 中观察 AnonHugePages
	time.Sleep(1 * time.Minute)

	fmt.Println("\n程序即将结束。")
	// 退出时，内存会被 Go 运行时和操作系统回收
}
```