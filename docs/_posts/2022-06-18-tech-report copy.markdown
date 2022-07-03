---
layout: post
title:  "weekly report"
date:   2022-06-18 22:30:08 +0800
categories: weeklyreport
---

# 读书

### 操作系统
继续读操作系统部分的内容, 本周继续. 操作系统启动时由 bios 开始触发, 加载启动区, 也就是 bootsect.s, 其中 bootsect.s 部分的代码负责将 setup.c, 操作系统内核加载到内存指定位置. 初始化栈顶位置. 并且通过 bios 提供的默认中断方法执行功能(读取硬盘, 读取光标). 将操作系统的一些初始化配置放到 0x90000开始的内存中(硬编码, offset 是死的). 将操作系统放到 0x00000-0x80000 的代码中. setup 代码放到 0x90200 部分代码中. 然后开始进入保护模式运行 setup.s 

这里附上公众号里面的两张图片

![bootsect_load](/assets/img/bootsect_load.JPG)

![bootsect_fin_state](/assets/img/bootsect_fin_state.PNG)

[公众号地址](https://mp.weixin.qq.com/s/5s_nmrWRZbA_4mkNKOQ2Cg)

### QUIC
最近随着 http3.0 协议的推出, QUIC 这个基于 UDP 的协议也出现在了大家的面前,便看了一下它具体的协议内容, 据说是可以在公网环境下得到比 tcp 性能更高的有状态链接. 但是在可靠网络环境下还是 TCP 更胜一筹, 不过因为 UDP 在流媒体出现之前受重视度不高. 导致内核里面的优化全部都是 TCP 的(过去三十年). UDP 的基本上没有. 那么可以说这个协议包括后面的 http3.0 的性能和安全性都是大有可期的.

[协议文档](https://datatracker.ietf.org/doc/html/rfc9000)

这篇文档看了前部分, 还没看, 篇幅还挺长,慢慢啃


# 日常

### livenessprobe

这周遇到了一个 livenessprobe 的配置问题, 原本是想通过脚本的方式检测 pod 的存活状态的. 但是由于粗心将参数和脚本写在一行之中, 结果在运行的时候报错了, 显示一直找不到文件.但是进到容器里面文件是存在的. 困扰了快一天了. 后来仔细观察报错信息发现,stat 找不到文件的并不是脚本, 而是脚本+参数这整个一行. 解决方法就是将参数换行即可. 真是低级的错误 

### 安全容器

这周稍微了解了一下安全容器的情况, 原来安全容器也是通过 containerd-shim 调用运行时创建虚拟机的, 原本以为安全容器走的是另一套逻辑. 不过后来想了想也对, 毕竟是要跟 kubelet 通信的. 还是要实现运行时的基本接口的, 那么在 containerd-shim 下一层应该是最简单的选择了


### lvm Thinly-Provisioned lv

这周第一听说有这种类型的 lvm , 换句话说是可以超卖的 lvm. 在写入数据的时候分配数据块, 这就代表可以分配比实际空间大的多的 lv 给应用. 并且后续可以自动扩容物理存储. 尚不知道稳定性如何


[linuxManPage](https://man7.org/linux/man-pages/man7/lvmthin.7.html)