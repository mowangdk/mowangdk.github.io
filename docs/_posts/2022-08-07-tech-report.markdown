---
layout: post
title:  "weekly report"
date:   2022-08-07 22:30:08 +0800
categories: weeklyreport
---

# 读书

### linux 设备驱动程序

这周没有什么特别的进展, 属于摸鱼的一周

### 操作系统

这周将操作系统基本功能的初始化了解了一下包括一下内容
- 主存的初始化(就是维护了一个 mem_map 的 mapping 结构)
- 内存缓存区初始化 (一个双向链表加一个 hash map 索引表)
- trap_init (初始化各种默认中断, 初始化内存中的 idt 数据结构)
- tty 显存初始化(特定内存位置映射到显存, 在这块内存中的数据会自动显示在屏幕上)
- schedule 进程初始化(初始化 gdt 中的 TSS & LDT 数据结构,并把当前进程(0 号)进程编入结构中)
- 键盘中断初始化

# 日常


### 如何判断一个进程是否因为 oom 退出

进程 oom 是系统判断的, 无法从进程 strace 日志中看到(只能看到收到相关的中断, 但是并不会有所谓 oom 的中断), 需要使用 dmesg 日志看, 会查询到 oom 相关的日志

### mmio, dma, pmio, 

内存映射 I/O 允许 CPU 通过读写特定的内存地址来控制硬件。通常，这将用于低带宽操作，例如更改控制位。

DMA允许硬件直接读写内存而不涉及CPU。通常，这将用于高带宽操作，例如磁盘 I/O 或摄像机视频输入。

#### history

Back in the old days, on x86 (PC) hardware, there was only I/O space and memory space. These were two different address spaces, accessed with different bus protocol and different CPU instructions, but able to talk over the same plug-in card slot.
Most devices used I/O space for both the control interface and the bulk data-transfer interface. The simple way to access data was to execute lots of CPU instructions to transfer data one word at a time from an I/O address to a memory address (sometimes known as "bit-banging.")
In order to move data from devices to host memory autonomously, there was no support in the ISA bus protocol for devices to initiate transfers. A compromise solution was invented: the DMA controller. This was a piece of hardware that sat up by the CPU and initiated transfers to move data from a device's I/O address to memory, or vice versa. Because the I/O address is the same, the DMA controller is doing the exact same operations as a CPU would, but a little more efficiently and allowing some freedom to keep running in the background (though possibly not for long as it can't talk to memory).
Fast-forward to the days of PCI, and the bus protocols got a lot smarter: any device can initiate a transfer. So it's possible for, say, a RAID controller card to move any data it likes to or from the host at any time it likes. This is called "bus master" mode, but for no particular reason people continue to refer to this mode as "DMA" even though the old DMA controller is long gone. Unlike old DMA transfers, there is frequently no corresponding I/O address at all, and the bus master mode is frequently the only interface present on the device, with no CPU "bit-banging" mode at all.



### 关于 http2.0 协议
3. grpc & dubbo3.0 都是基于http2.0 协议. grpc 目前已经基本成为事实上的标准 rpc 协议
4. 由于 http2.0 流的存在并且优化了 request header 的处理方式, 导致抓包和使用 ebpf (kprobe)分析变得十分困难- upprobe 是在userspace 的 probe, 可以完整的拿出 http2.0 的 header 进行分析, 是某种情况下的替代

