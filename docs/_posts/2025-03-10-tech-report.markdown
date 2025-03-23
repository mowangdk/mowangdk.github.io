---
layout: post
title:  "weekly report"
date:   2025-03-10 22:30:08 +0800
categories: weeklyreport
---


# 读书

### RDMA over Converaged Ethernet(RoCE)

基于融合以太网的远程直接内存访问（RoCE）是一种通过无损以太网实现超低延迟高效数据传输的机制。随着数据中心向可靠的以太网架构演进，ConnectX®以太网适配卡系列通过RoCE技术，利用成熟高效的RDMA传输协议，为在10GigE和40GigE链路速率下部署主流数据中心应用（如金融、数据库、存储和内容分发网络）提供了支持超低延迟的平台。其硬件卸载功能进一步发挥了这一高效RDMA传输的优势，满足了对性能要求严苛的高交易量应用场景需求。

当在以太网链路层部署RDMA应用时，需注意以下要点：

网络环境要求：需确保以太网基础设施为无损网络（如通过PFC和ECN技术避免丢包）；
RoCE版本选择：RoCEv2需基于UDP/IP协议栈，可与现有IP网络兼容；
硬件兼容性：须使用支持RoCE的网卡（如ConnectX系列）和交换机；
性能调优：需配置最小化传输路径（如旁路TCP/IP栈）、调整中断聚合策略以减少延迟；
安全隔离：建议通过VLAN或专用物理平面隔离RoCE流量与其他网络流量。


### RDMA

这种看了下 rdma 相关的资料，还是有一些收获的， 比如得知了 网卡本身也是有 dma 的， 只不过是网卡接收到了数据放到内存中是 网卡 driver 来获取（通过中断）， driver 取出之后放到网络协议栈中解析，解析完成之后最后是由用户的应用程序来获取。而 rdma 则是直接把数据放到用户应用程序中，避免了这几次数据拷贝和上下文解析.

当然这自然引入了很多的安全风险和潜在问题， 比如硬件都是直接用的物理内存地址， 但是用户组件用的是虚拟内存地址， 我们怎么保证网卡可以把数据写到用户的虚拟内存地址里面？并且如果当用户内存进行整理， 比如swap，这种情况下， 网卡是否会将数据写到其他应用程序的内存里面？ 并且开启了这个功能是否意味着rdma 有能力写入任意的用户进程内存， 并且用户进程有能力读写物理内存

为了解决上面的问题， RDMA 引入了 PD(Protection Domain) 和 MR(Memory Registration) 在 RDMA 中，PD 是一个容纳了各种资源的“容器”，类似一个租户 ID，将这些资源纳入自己的保护范围内，避免他们被未经授权的访问。一个进程中可以创建多个 PD，各个 PD 所容纳的资源彼此隔离，无法一起使用。MR 则是 RDMA 中对内存保护的一种措施，只有将要操作的内存注册到 MR 中，这段内存才能被 RDMA 使用

所以用户态内存只能访问到 MR 保护的内存，并且代码里面可以pin 主相关的虚拟内存，保证不会出现swap

发送一个 RDMA 请求的大致流程为：

- 软件构造 WQE （Work Queue Element），提交至 Work Queue 中
- 软件写 Doorbell 通知硬件
- 硬件拉取 WQE，处理 WQE
- 硬件处理完成，产生 CQE，写入 CQ
- 硬件产生中断（可选）
- 软件 Polling CQ
- 软件读取硬件更新后的 CQE，得知 WQE 完成



### nvme 抹盘

nvme 刷盘大概提供了两个命令， 一个是 ```nvme format```, 一个是 ```nvme sanitize``` 

linux 文档对这两个命令的说明个人感觉有点暧昧，不过起码 sanitize 里面有明确的对数据清理的说明。

nvmeformat: Secure erase the data on an SSD, format an LBA size or protection information for end-to-end data protection.

nvmesanitize: Securely eliminate all data on device, cannot be stopped. support block, crypto, and overwrite

https://manpages.debian.org/testing/nvme-cli/nvme-format.1.en.html
https://man.archlinux.org/man/nvme-sanitize.1
https://man.archlinux.org/man/nvme-sanitize-log.1


基本上sanitize 就是个异步, 需要通过检查 sanitize-log 来判断完成的状态

其实个人觉得通过加密 & 删除加密秘钥的方式抹数据可能也挺好， 但是感觉加密这个好像没有现成的方案， 网上找了很多都没有具体的example

简单总结下好像需要先对 nvme 进行 format， 然后在 sanitize 的时候抹掉

```
nvme format /dev/nvme0 --namespace-id=1 --ses=1 --pi=1
nvme sanitize /dev/nvme0 --sanact=0x04
```

### 工作

#### 关于存储现阶段的研究方向
目前的磁盘大概5-10年, 5年机械硬盘， 10年光盘

氧化硅玻璃存储为大容量，长寿命存储，场景主要是温冷数据
1Gbit/s 是单个激光器写入最大频率，跟吞吐有关
氧化硅介质本身稳定,不易破坏，但是服务介质的组件相对脆弱
数据持久时间跟做功大小相关， 意味着消耗电力可能较高， 使用成本可能较高， 使用飞秒 10的-15方每秒激光进行carving， 能量密度比较高，但是时间很短，并且实际的 carving 面积更小（单位bit占用面积越小，信息密度就越大，存储容量就更多）。结果做功较少, 
读取的时候，结构已经写入，使用另外一套激光器进行读取， 耗能更少

玻璃存储和全息存储对比，玻璃存储介质的误码率如何，写入的数据需要使用LDPC等算法编码吗

全息存储索尼已经在用
