---
layout: post
title:  "weekly report"
date:   2022-12-10 22:30:08 +0800
categories: weeklyreport
---


# 读书

### 闪客的操作系统源码分析

基本上到了整个系列的结束了, 最后作者使用 shell 命令的执行来将前面说过的东西串起来

shell 命令从键盘输入开始, 
- 系统通过中断接收到用户敲击的字符, 
- 首先会把他放在键盘的队列中
- 处理程序从队列中取出数据, 经过 termios 规范进行校验(这里这个方法的名称还是很有趣的, 可以借鉴到之后的自己写的代码里面去, xx_cooked(), 证明原始数据已经被处理完成)
- 处理完之后放到储存队列 secondary 里面. 这里存放的都是 cooked 的数据了. 
- shell 进程中处理, 这里分为两步, 一部是将数据写到显存中, 好让用户在屏幕上看到, 另一个部分是写在 buffer 里面, 等待换行/回车符开始执行命令

### 大话存储

#### ssd 磁盘

说实话尽管做存储也有一段时间了, 但是说来惭愧之前一直都没有好好了解一下所谓的 ssd(固态硬盘) 为什么会比普通的云盘介质快那么多.算是补上了一些只是漏洞吧

- ssd 里面一般同时存在易失性存储和非易失性存储, NAND Flash(也就是我们常用的U盘) + DRAM
- 因为有DRAM 的存在, 所以 ssd 中一般是存在电池用来保存/刷新DRAM 中的数据到持久化存储上的
- 由于 Flash 是用闪存粒子(电子元件)来做存储介质的(非磁性材料) 所以跟内存同样是直接使用地址来做寻址的. 比机械磁盘的磁头移动要快上不少
- NAND Flash 最大的问题就是在于其数据修改, 目前所有的删除都是元数据级别的删除, 并不会有实际动作,可以忽略, Flash 无法做到单个 Block 数据的变更, 所以更新的逻辑是, 把现有的 Page 数据全部拷贝到 SSD DRAM 中,再将磁盘指定 page 全部擦除, 修改完数据之后再将 DRAM 中的数据全部写回到 Flash. 这就导致我们往非 Free Page 处写入数据非常慢
- 由于闪存粒子是有声明周期的, 如果这个电子元件使用时间过长就会导致漏电/损坏. 如果损坏元件比较少, 这时 ssd 本身会使用纠错码进行修复. 如果过多的话, 这块存储就直接不可用了. 会导致用户可见的存储容量不断减少

> 当然 ssd 经过这么多年, 工程师们也在想尽各种方法来对这个情况进行优化, 现在商用的 ssd 存储已经可以做到比较经济实用的情况了

# 工作

### 这周听了一下龙蜥网络相关的优化分享, 学习到了两个新名词. 

   - RDMA (Remote Direct Memory Access)
      > RDMA 支持零复制的网络传输, 通过使用网络适配器直接在应用程序内存之间传输数据. 不需要再应用程序内存和操作系统缓冲区之间复制数据, 这种传输不需要CPU, CPU缓存或者上下文切换参与.
      - Kernel tcp/ip
          - 应用进程在用户态, 调用 socket 接口与内核交互
          - 协议和驱动相关的逻辑都运行在内核态, 通过内核与网卡交互
          - 层次清晰, 稳定友好的接口抽象, 应用程序使用简单
      - RDMA
          - 将传输层, 应用层,链路层等传统协议栈都下沉到网卡硬件中
          - 应用进程和网卡交互是通过内存进行的, 没有所谓数据包的概念
          - 数据链路性能极好, 应用改造成本大
   - SMC-R (Shared Memory Communication over RDMA) https://datatracker.ietf.org/doc/rfc7609/
      > 是 IBM 提出来的一个通过共享内存进行通信的方式(RFC7609), 可以在应用不感知 RDMA 的情况下支持 RDMA.
      - 简单说明下就是检测本地端和对端是否支持 RDMA, 如果支持, 则使用 RDMA 进行通信, 如果不支持, 则回退到 TCP/IP协议进行通信
        - Uses TCP connection (3-way handshake) to establish SMC-R connection
        - Each TCP end point exchanges TCP options that indicate whether it supports the SMC-R protocol
        - SMC-R “rendezvous(会面)” (RDMA attributes) information is then exchanged within the TCP data stream (similar to SSL handshake)
        - Socket application data is exchanged via RDMA (write operations)
        - TCP connection remains active (controls SMC-R connection)
        - This model preserves many critical existing operational and network management features of TCP/IP 
      这个可能是未来的方向, 现在需要大致了解一下


### 这周有一个问题就是用户在 copy 了一个集群 pvc/pv yaml 到另一个集群 apply 之后出现了 claim lost 的问题

简单定位了一下是 pvcontroller 判断出了问题, 导致报的错误跟实际情况并不相符. 不过这个用户也有问题, copy 的时候直接将 annotation copy 过来了, 导致 pvcontroler 判断存在问题.
k8s 社区好像将 annotation 视作 status 一样的,实际对资源行为生效的一种配置, 这点可以加到之后的文档设计里面去
