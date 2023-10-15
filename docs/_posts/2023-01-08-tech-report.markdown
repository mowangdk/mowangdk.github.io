---
layout: post
title:  "weekly report"
date:   2023-01-08 22:30:08 +0800
categories: weeklyreport
---

快一个月没写总结了. 有点惭愧

# 读书

### 闪客的操作系统源码分析

放假的时候把整个系列都读完了. 整体来看还是学到了不少东西的. 除了之前的那些总结之外, 还有很多需要了解的. 因为这些东西平时并不会经常遇到, 所以如果不伏羲的话会很快忘记. 我个人的经验是首先还是要巩固, 在一个就是在平时的基础行为中回想相关的知识点, 将其串联起来. 之前还买了一本 30 天自制操作系统的书. 是买这本书的原因是. 看之前有一个 github 仓库火了, 据说是抄袭的这本书的代码. 然后就想买回来看看这本书究竟有什么神奇之处. 接下来就从这个地方入手了


### 大话存储

#### raid

raid 是现代存储阵列中不可缺少的一环. 因为它的数据冗余和它的多磁盘 striping 能力基本上已经成为各大云厂商(当然个人存储阵列也是)的标配了. 所以这本书里面花了很大的篇幅去讲了. 这里很多就是一些设计上的介绍. 包括如何实现数据校验, 如何通过数据校验找回丢失的数据. 这些感觉都是形而上的东西. 对现在的我来说并没有太大的用处, 所以还是简单过一遍, 直接进入下一篇.

# 论文

### It’s Time to Replace TCP in the Datacenter

[url](https://arxiv.org/pdf/2210.00714.pdf)

这周开坑了这篇论文, 主要是列举了 tcp 的几大缺点(也算是众所周知了). 以及为何不适宜在 datacenter 中使用 tcp 的原因. 把它当做一个优化型的论文看还是很油必要的. 可以看下目前的主要瓶颈点都在哪些, 同样可以复习一下 tcp 的基本情况和功能. 这个等下周看完之后再详细总结一下


# 工作

### subpathexpr

才知道 k8s 有一个 subpathexpr 的配置, 在 k8s 1.17 就已经 GA 了. 主要功能就是将 pod 的一些 env 拿出来 用作在指定的 volume 上创建子目录. 如下


```yaml
    volumeMounts:
    - name: test
      mountPath: /logs
      subPathExpr: $(POD_NAME)
  volumes:
  - name: test
    hostPath:
      path: /mnt
```
这里 ```subPathExpr``` 会在 /mnt 目录下创建 podname 的子目录, 并且将 容器内的 logs 目录映射到 /mnt/$(POD_NAME) 下. 目前试过volume 为 hostpath & 云盘都是没问题的. 同理可推 nas 和 oss 也没什么太大问题

> 这里再插一嘴 在 用户 pod 里面, volumeMounts 在容器内只有挂载点是可见的(因为存在于/proc/mountinfo). 但是相关设备符是找不到的, 因为是独立的 /dev 目录. 并且用户也是无法再 pod 里面读写块设备自身的. 符合需求

### 定位磁盘热插拔问题

- 可以使用如下命令增加内核打印命令
```
echo "file drivers/pci/hotplug/* +p" > /sys/kernel/debug/dynamic_debug/control
```
- 使用 cat /proc/interrupts 命令查看相关SCI中断是否有下发到 vm 内部
```
  9:          0          8          0          0          3          0          0          0   IO-APIC   9-fasteoi   acpi
```
主要看 acpi 中断是否有增加
- 如果有增加, 则大概率是内核方面的问题了


### du 和 df 计算出来的磁盘占用率有差距

> https://unix.stackexchange.com/questions/45771/df-vs-du-why-so-much-difference

du 是计算所有目录下的文件的容量综合, 而 df 则是统计了所有 superblock 的使用量(之前在文件系统章节有提到过基本组成) 所以这个差距基本上就是不可见文件的差距了. 可以使用上述方式统计进程 hold 的文件句柄

```
lsof -Fn -Fs |grep -B1 -i deleted | grep ^s | cut -c 2- | awk '{s+=$1} END {print s}'
```

### 端口占用问题

发现机器上一直有端口占用, 于是登录机器排查原因

1. 使用 lsof 找到被占用的端口所属的进程

``` lsof -i: xxx```

但是惊奇的发现,没有相关进程

2. 不死心, 使用netstat 进一步确认

``` netstat -ntlpe```

![netstat](/assets/img/port_occupied.png)

发现端口的确是开着的, 但是后面却没有相关进程号, 但是有相关的 inode 号.

3. 根据 inode 号找到相关进程

```find /proc/ | xargs ls -lh 2>/dev/null| grep 41775```

最后发现居然是内核线程在持有这个 inode, 怪不得没有显示出进程号. 不过这下的解决方式也只有重启了.