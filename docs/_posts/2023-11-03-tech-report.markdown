---
layout: post
title:  "weekly report"
date:   2023-11-03 22:30:08 +0800
categories: weeklyreport
---


# 读书

### 大话存储

本周无进展

### rust 

看了下基本类型, 同时也看了些 rust 代码, Some, None 本身对我来说还是有些新奇. 之后打算边看 kata 的代码边学习. 应该可以上手的更快一些.  

# 社区

社区会议没持续多长时间, 估计大家都在搞北美的 kubecon. 不知道 wiz 那边最后怎么样了. 之前搞的升级也被合并了. 目前基本上待合并的pr 都已经合并了. 下周找时间做点新的


# 工作

### 通过设备号获取 bdf number

通过读取 /sys/block/xxx 下对应的块设备对应的软连接可以找出对应的 bdf number. 但是这种方式只适用于节点上有相关的 rules 并且盘已经被绑定到某个 driver 上. 但是如果盘没有被绑定 driver , 或者绑定的 driver 不是个标准的存储 driver, 亦或是没有对应的 rules 则还是需要通过 mmap 的方式获取 seriesid 和 bdf 的对应关系. 但是有点奇怪, 有一些 link 的 bdf 会很多, 有一些只有一个. 不知道是哪里决定的. 需要再看下

比如:

normal:
```
nvme0n1 -> ../devices/pci0000:00/0000:00:04.0/nvme/nvme0/nvme0n1
```

special: 

```
nvme3n1 -> ../devices/pci0000:16/0000:16:02.0/0000:17:00.0/0000:18:08.0/0000:21:00.3/nvme/nvme3/nvme3n1
```

确认了下, 可以确认一个块设备只会对应一个 bdf number , 但是一个 bdf number 可以对应多个块设备/网卡. 类似 nvme namespaces. 上面的默认只看最后一个 bdf number 就可以, 中间都是一些目录, 想要确认这些 bdf number 是什么代表,还需要继续探索下

### nvme-reset

最近在讨论什么条件下会触发 nvme reset 命令. 这里有整理的一个列表 

1. nvme probe, 在 nvme 驱动加载 nvme 盘的时候调用, probe 会调用 reset 来初始化内存
2. keepalive , nvme 驱动下发 keepalive 命令的时候如果申请不到内存(admin队列满了) 会触发 reset
3. nvme 命令超时, 当下发的 nvme 命令出现超时的时候, block 是会触发 timeout 机制, 流程里面可能会触发 reset
4. pcie 错误恢复, pcie硬件出现问题. 如果硬件上报 reset slot. 则会调用 nvme 驱动注册的 slot reset 则会触发 reset
5. pcie 电源管理, 同样, nvme 会注册方法到 dev_pm_ops(device power manager ops) 中, 当底层有事件触发, 就会进行回调, 触发 reset. 

### 文件系统目录级别的监控

ebpf 可以用于解决 hostpath + emptyDir 统计目录级别的数据本身没问题. 但是它能统计的也只是 vfs 这一个层次的监控(pagecache), 至于这个数据具体什么时候落盘. 落盘引起的实际 io 流量就不是 ebpf 能做的事情了. 除了 ebpf 之外. 其实也可以通过 cgroup + device 设备的方式来区分流量来源以及相关监控. 当然这种方式也有其局限性. 需要实际上线然后观察相关效果
