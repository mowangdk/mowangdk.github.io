---
layout: post
title:  "weekly report"
date:   2024-03-08 22:30:08 +0800
categories: weeklyreport
---


# 读书


## nvme (https://nvmexpress.org/wp-content/uploads/NVM-Express-Base-Specification-2_0-2021.06.02-Ratified-4.pdf)

最近 nvme 用的越来越多, 我需要尽快恶补一下基础知识了... 目标并不是掌握所有的 nvme 特性, 那些东西随时可以通过翻阅 nvme 协议/gpt 进行查询, 主要还是了解 nvme 分型, 基础协议, 还有相关定义

1. nvme over pci vs nvme over fabric
   一个是跟本地外设进行交互的协议, 一个是跟网络上其他设备交互的协议. 在 nvme 1.3 之前这两个协议都已支持. 
2. nvme 对原有的接口进行了优化, 支持同时并行操作多达 65535 个 IO 队列, 每个队列最多可以有 64K 条未执行的队列. 另外, 还支持了很多企业级功能, 比如 end to end 的数据保护(与 SCSI Protection information 配合, 通常被称为 T10DIF) 
3. 全特性列表(1.3)
    - 在命令的提交或者完成路径上不需要读取 uncacheable/MMIO 寄存器
    - 在命令提交路径中最多只需要写入一个 MMIO 寄存器
    - 65535 个 IO 队列, 64K 条未执行的命令
    - 每条 IO 队列都有优先级, 具有良好定义的仲裁机制
    - 只需 64B 大小的命令就可以完成 4KB 的数据请求, 可以保证小 IO 能高效率的完成
    - 高效,精简的指令集
    - 支持 MSI/MSI-X 和中断聚合
    - 支持多 namespace
    - 对 IO 虚拟化架构有着良好的支持, 例如 SR-IOV
    - 鲁棒性的 错误报告和管理能力
    - 支持多路径 IO 和 namespace 共享
4. nvme 规范定义了一套精简的寄存器, 这些寄存器的功能包括
    - indication of controller capabilities(显示控制器的功能)
    - 控制器失败的状态(指定的状态通过 CQ 直接传递) 
        - CQ: Completion Queue
        - SQ: Submission Queue
    - 管理队列配置, 通过管理命令来配置 IO Queue
    - 门铃寄存器, 可用于配置 SQ & CQ 扩展性 

5. nvme controller 是跟单个 PCI function 相关联的, 可以在 Controller Capabilities 寄存器和 Identify Controller 数据结构中设置整个 controller 的能力. 

6. namespace 是可以格式化的逻辑设备的数量. 一个 nvme controller 可以支持多个 namespace, 通过 namespaceID 进行引用区分. namespace 可以使用 namespace management and  namespace attachment 命令进行管理. 每个 namespace 都有自己的配置, 所有 namespsace 通用的配置在 namespaceid 为 FFFFFFFFh 的 identity namespace 数据结构里面.

7. nvme 命令通过 SQ & CQ 来进行交互, 首先由用户进程将命令放到 SQ 中, Controller 执行 SQ 的的命令后, 将结果放到关联的 CQ 中. 多个 SQ 可以关联一个 CQ. SQ 和 CQ 都是在内存中的数据结构

8. 除此之外还有一个 Admin Submission & Completion Queue(normal SQ & CQ with identifier 0) 主要用于controller management & control. 只有 Admin 命令集才能使用这个 Queue. 一般来说 Admin Submission Queue 和 Admin Completion Queue 总是 1:1 的.

9.  当有 1 到 n 个 command 要去执行的时候, 主机软件通过更新特定的 SQ 尾部的 doorbell 寄存器,来向拥有特定 slot 数量的环状缓存 SQ 里面提交 command, controller 会按照顺序获取 SQ entries(每一个 entry 都是一个 commmand). 但是它可能以任意顺序执行这些命令

10. SQ 里面的 entry 有 64 bytes 大小, 在数据传输中使用的物理内存位置由 Physical Region page entrys 或者 Scatter Gather Lists 指定. 每个 command 一般会由 two PRP entries 或者 one SGL segment 组成 如果需要有两个以上的 PRP 指定数据传输 buffer. 则会创建一个 PRP List 用于保存, 如果使用多于一个 SGL 的话会以链表的形式进行附加 

11. namespace vs nvme controller vs pcie port 

在第二张图中, 与共享名称空间相关的控制器可同时对该名称空间进行操作。单个 controller 对 namespace 写是原子性的. 多个 controller 对同一个 namespace 的操作不是并发安全的, 需要在文件系统/存储协议/应用级别锁机制来实现。如果向访问共享名称空间的不同控制器发出的命令之间有任何排序要求，则主机软件或相关应用程序必须执行这些排序要求。

![](/assets/img/nvme1.jpg)
![](/assets/img/nvme2.jpg)
![](/assets/img/nvme3.jpg)
![](/assets/img/nvme4.jpg)

12. 几个名词
- arbitration burst: 一次可以从 SQ 中取出的最大 command 数量
- arbitration mechanism: 用于从众多 SQ 中选择哪个 SQ 是下一个消费对象, 一般有, round robin, weighted round robin with urgent priority class. 和 vendor specific
- cache: 被 nvme subsystem 使用的存储空间. 在 host 上无法访问. 这个空间可能存储着已经落盘的数据和尚未落盘的数据(not commited)
- candidate command: 指的是被 controller 消费的 command, 被认为已经准备好去执行了
- command completion: 指的是 controller 已经执行完的命令, 并且已经更新了 CQ 中 entry 的状态, 并且将 entry 推送到了 CQ.
- controller: A PCI Express function that implements NVM Express
- directive(指示/命令): host 和 nvme subsystem 交互的方法, 可以通过 Directive Send and Directive Receive command 进行交互. 
- emulated controller, 软件的 nvme controller, 一个模拟的 controller 可能/可能没有对应的 物理的 nvme controller. 也就是 physical PCIe function   
- firmware slot: 用于存储固件镜像的位置. 一个 controller 可以存储 1-7 个固件镜像
- LBA: Logical block address 逻辑块的地址, LBA range, 连续快的集合, 通过 LBA and 逻辑块的数量指定
- metadata: 关于 LBA 中 data 的上下文信息. 如果这个逻辑块是 nvme controller 存入的, 那么 nvme subsystem 会存入一些元信息
- nvme subsystem: 包含一个到多个的 controllers, 一个到多个的 namespaces, 一个到多个的 PCI Express port, 一个非易失性存储介质, 还有controller 操作这个存储介质的 interface
- private namespace: 一个 namespace 在同一个时刻只会被关联到一个 controller, host 可根据识别名称空间数据结构中名称空间多路径 I/O 和名称空间共享能力 (NMIC) 字段的值，确定名称空间是专用名称空间还是共享名称空间。




# 社区

### csi sidecar AIO

昨天晚上开了一个会, aws 的人也上来了. 还是聊了不少东西的, 我这边准备把 Feature Deployment workflow definition 列一下, 下下周让大家 review 下


# 工作

### csi pod 删除失败

遇到了一个问题, csi pod 删除失败, 报错 ```Error removing mounted layer, unlink xxxx device or resource busy```, 表面上看很容易就是 container 某一层被占用了, 直接 lsof 看发现 layer 有非常多的进程占用, 基本上是全部了,  节点上 dmesg 输出了非常多的 segfault. 
最神奇的现象是 umount 这个路径报错 not mounted, 返回 EINVAL.  但是 findmnt 这个路径却有输出.

最后找了下内核同学帮忙定位. 
```
grep /var/lib/docker/overlay2/xxxx/merged /proc/*/mounts | sort | uniq -c | awk '{print $2}' | grep -Eo '/proc/[0-9]+' | xargs -I {} ls -alh {}/ns/mnt | awk '{print $11}' | sort | uniq -c
```

findmnt 有信息应该是 /proc/mounts 列出的挂载点列表里面包含，umount返回 EINVAL 的路径很多，目前看没办法简单断定是哪一步，这种状态不一致重启是能解决的，目前我这边也没有很好的手段，尝试过按 /proc/mounts 的参数挂回去发现挂不回去，然后不知道docker内部调用 umount 的时候是不是带 lazy 参数，尝试构建 umount -l 执行的时候挂载点的文件还被读写的场景，看看能不能复现（目前测试下来看不是这种情况导致的）

最后发现是 ksys_umount 这个方法直接退出了, 通过 crash 工具抓出来的 mount 结构和 bdftrace 抓的 mount 结构不一致, 发现存在两个挂载点. 并且使用的是同一个设备. 



