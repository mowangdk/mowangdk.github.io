---
layout: post
title:  "weekly report"
date:   2022-08-28 22:30:08 +0800
categories: weeklyreport
---

# 读书

### 论文

https://www.usenix.org/conference/osdi22/presentation/zhong

本周开始读这篇, 感觉这是之后可以用的上的东西.需要仔细研读一下. 相关的 GitHub repo 是 https://github.com/xrp-project/XRP. 虽然说工作中并无法摸鱼,不过还是可以花一些时间探究一下的 


# 日常

### io_uring

属于操作系统层面, 在 fs 层之上, 提供了批量处理 io 的方式. 所有操作对上层均为异步.
io_uring 是 Linux 提供的一个异步 I/O 接口。 在 2019 年加入 Linux 内核. io_uring实现了三个 syscall ```io_uring_setup```, ```io_uring_enter```, ```io_uring_register```. 它们分别用于设置 io_uring 上下文，提交并获取完成任务，以及注册内核用户共享的缓冲区。使用前两个 syscall 已经足够使用 io_uring 接口了。

- Submission Queue 一整块连续的内存空间存储的环形队列。用于存放将执行操作的数据。
- Completion Queue 一整块连续的内存空间存储的环形队列。用于存放完成操作返回的结果。
- Submission Queue Entry 提交队列中的一项。
- Completion Queue Entry 完成队列中的一项。
- Ring 比如 SQ Ring，就是“提交队列信息”的意思。包含队列数据、队列大小、丢失项等等信息。

io_uring_setup 设计的巧妙之处在于，内核通过一块和用户共享的内存区域进行消息的传递。在创建上下文后，任务提交、任务收割等操作都通过这块共享的内存区域进行，在 IO_SQPOLL 模式下(后面介绍)，可以完全绕过 Linux 的 syscall 机制完成需要内核介入的操作（比如读写文件），大大减少了 syscall 切换上下文、刷 TLB 的开销。




