---
layout: post
title:  "weekly report"
date:   2022-06-03 22:30:08 +0800
categories: weeklyreport
---

# 读书

 继续啃这本书 Understanding Linux Network Internals, 这周是第五章, 主要关于网络设备的安装. 包含以下六个方面

 - 网络代码的初始化
    - <start_kernal>
    - parse_early_param()
        - parse_args()
    - parse_args() # 这里并不是笔误, 的确在内核初始化的时候需要解析两次参数, 原因是有些参数必须要提前处理,并且保证初始化顺序, 所以就在逻辑上对所有参数进行区分(结构体里面有flag). 分别进行初始化(later in chapter6)
    - ...
    - init_IRQ()
    - init_timers()
    - softirq_init()
    - rest_init()
        - kernel_thread(init, ...) 
        - <start_init>
        - do_basic_setup()
            - usermodelhelper_init()
            - driver_init()
            - socket_init()
            - do_initcalls()
                - fn1 ...
                - fni ...
                - fnn ...
        - ...
        - free_init_mem()
        - ...
        - run_init_process() # 定义了第一个在系统上运行的进程(也是所有其他进程的父进程, PID = 1), 用户可以指定, 如果用户未指定, 则内核则运行一个标准的 init 进程

 - NIC 是如何安装的, 以及后续是如何使用的
   - 硬件安装是由 device driver 和 PCI bus layer 共同安装的. 
   - 软件安装, 比如网络协议相关的软件还有用户配置, 比如 ip 地址
   - 功能安装, 比如 trafic control, 还有出队和入队的功能安装 

 - NIC 如何发出中断, IRQ handlers 是如何工作的.
    - 每一个 NIC 都会被赋予一个 IRQ 用于和内核通讯(吸引内核的注意力到自己身上), 相反, 虚拟的设备则不需要IRQ. (因为不需要与外部通信, 比如loopback)
 - 用户如何向作为模块加载的设备驱动程序提供配置参数
 - 在设备安装和配置期间 内核和用户空间是怎么交互的. 并且热插拔功能的实现也是在这部分介绍的
 - 虚拟设备和物理设备对内核来说有和不同

 可以看到虽然这本书是网络方面的,但是它介绍的十分详细, 甚至连非网络部分也有着比较细致的介绍,(或者说,网络部分比较基础,想要学好网络部分, 就必须了解内核相关的知识.)

 -- 暂且先这些, 下周继续
 
 # 日常

### vpc

本周主要看了下如何实现一个 vpc . 基本上是通过 vxlan 来在一个大的电脑集群中分成不同的 arp 区域, 区域内 arp 都是可以顺利请求到的. 但是区域外的联通就需要额外的组件, 有两个单词需要记录下
- TX: Transmit
- RX: Receive