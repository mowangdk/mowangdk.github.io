---
layout: post
title:  "weekly report"
date:   2023-02-04 22:30:08 +0800
categories: weeklyreport
---

这周连上了七天班, 书没读多少, 问题倒是遇到了一些, 先从问题开始吧

# 读书

### 大话存储

本周无进展

### 30 天自制操作系统

同样进展不多

# 工作


### ISCSI

本周尝试在集群中安装 longhorn, 但是在安装的时候报了如下错误.  

```
time="2023-01-30T09:07:14Z" level=fatal msg="Error starting manager: environment check failed: failed to execute: nsenter [--mount=/host/proc/1/ns/mnt --net=/host/proc/1/ns/net iscsiadm --version], output , stderr nsenter: failed to execute iscsiadm: No such file or directory\n: exit status 127"
```

经确认是节点上没有安装 ISCSI , 那么 ISCSI 是什么呢?
ISCSI (Internet Small Computer System Interface) 是基于 SCSI 协议而形成的共享存储协议, 他可以在 ISCSI initiator 和 ISCSI Target 之间传输块设备层的数据. ISCSI 协议包裹了 SCSI 的指令, 并且收集了 TCP/IP 层 package 的数据, package 通过点对点的链接在网络中传输

数据抵达后, ISCSI 协议会解析 package 数据包, 这样 os 就会看到传输的数据, 就像链接一个本地的 SCSI 的设备一样. 目前 ISCSI 的一些主要是在存储池化的协议中使用. 集群中的所有节点都可以访问共享存储池.

ISCSI 由 initiator 和 tatget 两部分组成
ISCSI Initiator
是安装在应用服务侧的软件/硬件. 用于向 ISCSI 的存储阵列 (ISCSI Target) 发送或者请求数据, 如果是软件的话, 相应的性能开销则下放到了 CPU 上, 如果是硬件的话, 虽然可以卸载 CPU Load, 但是整体机器的价格就更贵了  

ISCSI Target
实际的数据存储阵列, 提供相关接口供 ISCSI Initiator 访问
