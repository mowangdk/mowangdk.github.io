---
layout: post
title:  "weekly report"
date:   2025-03-10 22:30:08 +0800
categories: weeklyreport
---


# 读书

这周主要在读一些大模型的基础概念，简单来说就是大模型是如何搞出来的，当然也没想当什么专家， 只是想先了解下，多少交流起来不那么费劲


### ioring by openai

尝试在 openai 上通过简单的问答来了解 ioring 这个新型的存储， 并且直接让他给我组装了一个文章

IO_uring 与 epoll 的对比及优势劣势分析

在现代 Linux 系统中，epoll 和 IO_uring 都是用于处理异步 I/O 操作的重要技术。虽然两者都旨在提高 I/O 操作的性能，减少系统调用的开销，但它们的设计理念、应用场景以及优缺点存在显著差异。本文将从兼容性、复杂性、性能优化、内存管理等多个角度，深入分析 IO_uring 相较于 epoll 的优劣。

1. 概述
	•	epoll 是 Linux 内核提供的一种高效事件通知机制，适用于需要处理大量文件描述符的高并发网络 I/O 操作。它通过 epoll_create()、epoll_ctl() 和 epoll_wait() 等 API 允许应用程序高效地管理文件描述符及其状态。
	•	IO_uring 是 Linux 内核在 5.1 版本引入的一种全新的异步 I/O 接口，旨在减少传统 I/O 操作中的系统调用和上下文切换，提升 I/O 操作的性能。它通过环形队列（提交队列和完成队列）来实现低延迟的 I/O 操作，广泛适用于高并发的 I/O 场景。

2. IO_uring 相对 epoll 的劣势

2.1 兼容性问题
	•	epoll 是一个成熟的技术，几乎所有的 Linux 系统都支持，且被大量生产环境中的应用广泛使用。对于开发者来说，使用 epoll 时可以确保跨系统的兼容性。
	•	IO_uring 是相对较新的技术，要求内核版本为 5.1 或更高版本。因此，只有支持该技术的内核才能使用 IO_uring，这在老旧的系统或不支持该内核的环境中可能会遇到兼容性问题。

2.2 学习曲线和复杂性
	•	epoll 提供了简单明了的 API，开发者可以快速上手并实现高并发 I/O 操作。其事件驱动模型非常适合处理大量文件描述符的 I/O 请求。
	•	IO_uring 的 API 较为复杂，涉及提交队列、完成队列等概念，需要开发者对内存映射、环形缓冲区等底层细节有较深入的理解。对于新手开发者，理解和使用 IO_uring 可能需要更多的时间和精力。

2.3 性能优化与场景适用性
	•	epoll 在高并发的场景下表现出色，尤其是处理大量网络连接时。它的性能瓶颈主要出现在需要处理大量文件描述符的场景，尤其是当 I/O 请求非常多时，epoll 的轮询机制可能会影响性能。
	•	IO_uring 在需要处理大量异步 I/O 请求时能够提供显著的性能提升，特别是在文件操作、网络 I/O 等场景中能够减少内存拷贝、减少上下文切换，提高吞吐量。然而，在较小规模或简单的 I/O 场景下，IO_uring 的复杂性可能会带来不必要的开销，导致性能不如 epoll。

2.4 内存管理与资源占用
	•	epoll 在内存管理上相对简单，不需要进行复杂的内存映射，它的内存占用较低，适合处理大量文件描述符的高并发场景。
	•	IO_uring 通过使用提交队列和完成队列来管理 I/O 请求，因此其内存管理较为复杂。尽管 IO_uring 在高并发场景下能减少内存拷贝和上下文切换，但在某些场景中，尤其是在资源较为紧张的系统上，IO_uring 可能会导致较高的内存消耗。

2.5 调试和排错
	•	epoll 是一个成熟的技术，具有大量的调试工具和文档，开发者可以轻松地进行调试和问题定位。
	•	IO_uring 由于较为新颖，其调试工具和排错支持相对较少。在出现问题时，开发者可能需要花费更多时间进行调试和定位，尤其是涉及内存映射和低层次性能调优时，可能会遇到更复杂的问题。

2.6 系统调用开销
	•	epoll 在高并发场景下性能优越，其设计优化了系统调用的开销，只需要通过 epoll_wait() 等系统调用进行事件轮询，避免了大量不必要的系统调用。
	•	IO_uring 虽然减少了传统异步 I/O 模型（如 select、poll）中的系统调用开销，但依然需要通过 io_uring_enter() 来提交和管理 I/O 请求。在某些低频率 I/O 操作的场景中，IO_uring 的系统调用开销可能不如 epoll。

2.7 事件驱动 vs 完成通知
	•	epoll 是基于事件驱动的模型，它通过事件通知机制等待 I/O 事件的发生，适用于大量并发文件描述符的事件管理。
	•	IO_uring 则采用完成通知的机制，内核通过完成队列主动通知用户空间 I/O 请求的完成情况。这种设计能够进一步减少上下文切换和系统调用的开销，但在低频率的 I/O 操作中，IO_uring 的优势可能并不明显。

3. 总结：何时选择 epoll 或 IO_uring

选择 epoll 的场景：
	•	小规模、高并发网络 I/O：对于大多数需要高并发且简单的 I/O 处理的场景（例如 Web 服务器），epoll 是一个足够高效且易于实现的解决方案。
	•	兼容性要求高的系统：如果需要确保在各种 Linux 系统上兼容运行，epoll 是一个成熟且稳定的选择。
	•	内存占用和系统开销要求低的场景：对于资源较少的系统，epoll 的内存占用和系统调用开销较低，适合在低资源环境下运行。

选择 IO_uring 的场景：
	•	大规模异步 I/O 操作：当应用程序需要处理大量异步 I/O 请求，且对性能要求较高时，IO_uring 提供了更优的吞吐量和低延迟。
	•	需要降低内存拷贝和上下文切换的场景：对于文件操作、网络传输等大规模 I/O 场景，IO_uring 的设计能够有效减少内存拷贝和上下文切换，提高性能。
	•	高性能计算和分布式存储系统：在分布式系统中，尤其是高性能计算或大规模存储系统中，IO_uring 可以显著提高数据传输的效率。

4. 结语

IO_uring 和 epoll 各自有其优势和适用场景。尽管 IO_uring 提供了更高效的异步 I/O 操作，适合高并发、大规模 I/O 请求的场景，但它的复杂性、系统兼容性和学习曲线使得它在一些简单场景中并不总是优于 epoll。对于传统的高并发 I/O 场景，epoll 仍然是一个非常高效且成熟的解决方案。开发者应根据具体的应用需求、系统环境和性能要求，选择最合适的 I/O 机制。


确实不错，感觉以后都不需要自己写文档了, 并且以后提问确实更重要了, 对世界是否好奇，是否能问出好问题，并且是否具备能力验证，将是非常重要的议题


# 工作

### fio 压测

这两天基于同事的脚本部署了一个基于 弹性临时盘 的 3fs 集群， 准备和普通云盘对比下性能, 之前其实没有单独搞过 fio, 正好趁着这次新建一下， 以前一直好奇 fio 如果测试读取的话，是否需要提前放置数据， 结果发现并不需要， fio 在测试之前会去 Laying out IO file。 （但是响应的也很慢）

#### 随机读

command: ```fio -numjobs=1 -iodepth=1 -ioengine=libaio -direct=1 -rw=randread -bs=4k --group_reporting -size=2G -time_based -runtime=60 -name 3fs -directory=xxxx```

| | 3fs | cloud_essd autopl |
| --- | --- | --- |
| IOPS | 1160 | 3175 |
| BW | 4641KiB/s | 12.4MiB/s |


command: ```fio -numjobs=12 -iodepth=1 -ioengine=libaio -direct=1 -rw=randread -bs=4k --group_reporting -size=2G -time_based -runtime=60 -name 3fs -directory=xxxx```

| | 3fs | cloud_essd autopl |
| --- | --- | --- |
| IOPS | 12.2k | 19.6k |
| BW | 47.8MiB/s | 76.5MiB/s |


command: ```fio -numjobs=20 -iodepth=1 -ioengine=libaio -direct=1 -rw=randread -bs=4k --group_reporting -size=2G -time_based -runtime=60 -name 3fs -directory=xxxx```

| | 3fs | cloud_essd autopl |
| --- | --- | --- |
| IOPS | 11.9k | 40.7k |
| BW | 33.1MiB/s | 159MiB/s |

额外测试了一下 numjobs=30 更低了，3fs iops大概 5k, BW 大概 16.9MB/s， 云盘压到头大概是 IOPS=41.0k, BW=160MiB/s (168MB/s), 整体是 3fs 的 3 倍左右, 最高 BW 可以达到 320MB/s

同样， 扩容到了200Gi， 尝试下随机写

command: ```fio -numjobs=14 -iodepth=1 -ioengine=libaio -direct=1 -rw=randread -bs=4k --group_reporting -size=2G -time_based -runtime=60 -name 3fs -directory=xxxx``

IOPS=16.1k, BW=51.1MiB/s

command: ```fio -numjobs=20 -iodepth=1 -ioengine=libaio -direct=1 -rw=randread -bs=4k --group_reporting -size=2G -time_based -runtime=60 -name 3fs -directory=xxxx``

IOPS=14.6k, BW=44.4MiB/s




#### 顺序读

command: ```fio -numjobs=12 -iodepth=1 -ioengine=libaio -direct=1 -rw=read -bs=4k --group_reporting -size=2G -time_based -runtime=60 -name 3fs -directory=xxxx```

| | 3fs | cloud_essd autopl |
| --- | --- | --- |
| IOPS | 14.8k | 40.7k |
| BW | 45.0MiB/s | 159MiB/s |

command: ```fio -numjobs=16 -iodepth=1 -ioengine=libaio -direct=1 -rw=read -bs=4k --group_reporting -size=2G -time_based -runtime=60 -name 3fs -directory=xxxx```

| | 3fs |
| --- | --- |
| IOPS | 16.6k |
| BW | 47.2MiB/s |

尝试将 3fs 进行扩容，从 64Gi 扩容至 200Gi, 使用 16 jobs 进行压测，发现提升不多

| | 3fs |
| --- | --- |
| IOPS | 17.3k |
| BW | 49.2MiB/s |

降低 jobs 到 14， 发现有所提升

| | 3fs |
| --- | --- |
| IOPS | 16.3k |
| BW | 50.9MiB/s |


#### 顺序写

试了下 200Gi 的顺序写， 惨不忍睹
command: ```fio -numjobs=16 -iodepth=1 -ioengine=libaio -direct=1 -rw=write -bs=4k --group_reporting -size=2G -time_based -runtime=60 -name 3fs -directory=/mnt/3fs```
| | 3fs | cloud_essd autopl |
| --- | --- | --- |
| IOPS | 90 | 40.7k |
| BW | 362KiB/s | 159MiB/s |


