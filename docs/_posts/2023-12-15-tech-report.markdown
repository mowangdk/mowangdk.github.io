---
layout: post
title:  "weekly report"
date:   2023-12-15 22:30:08 +0800
categories: weeklyreport
---


# 读书
这周主要还是做一些社区的工作。 书读的不是很多。 

# 社区

### AIO

这周看 all  in one 的项目, 想参与下, 下周打声招呼。 看看是否能参与进来

### scheduler support csinodes driver

https://github.com/kubernetes/kubernetes/pull/122109

现在单侧跑过了， 但是e2e 还是不行， 断断续续搞了一周的kind 结果还是不太行， 自己编译一个scheduler 在自己的机器上跑e2e试试吧

# 工作


### securityContext
  k8s 里面的 fsgroup 只适用于设置了 readwriteonce & 设置了fstype 的存储卷，相关代码在 https://github.com/kubernetes/kubernetes/blob/4cff5e8d2dc7df19b1cdceb7146a40e7ebc50085/pkg/apis/storage/types.go#L430`

### golang io.copy
  这周遇到了一个问题， 内核同学在抓包的时候发现在不同os 上golang runtime 系统调用时不一样的， 详细追了一下代码。 发现 golang 会对内核 < 5.3 的版本在调用copy 方法的时候走默认的逻辑。 当大于5.3 的时候， 走 copy_file_range 的逻辑， 也就是不经过Userspace 直接在内核空间复制数据的方式. 相关源码
https://github.com/golang/go/commit/7be3f09deb2dc1d57cfc18b18e12192be3544794
https://github.com/golang/go/commit/1c7650aa93bd53b7df0bbb34693fc5a16d9f67af

但是这里用户是升级完之后才导致异常， 换句话说本应该是一个优化的方案却导致了性能变差， 无论如何是不可能的。 所以目前只能确定相关的问题并非上面事情导致。
后来经内核同学确认， 是由于 ```read_ahead_kb``` 配置发生了变动导致， 修改了相关的参数之后就可以生效
https://access.redhat.com/solutions/5953561

### klog
  这两天同事遇到了一个问题，有一些问题日志在程序退出之后没有写到stdout里面， ```os.exit(1)```  失败. 发现klog 的 logger 没有办法fatal， 虽然提供了替代方法， 但是貌似还是不行。 

### tcp_slot_table_entries

这个参数导致的nfs 性能下降， 详细文档

https://www.alibabacloud.com/help/en/nas/modify-the-maximum-number-of-concurrent-nfs-requests

### 如何进入到容器的namespace

1. 在节点上通过  ```crictl inspect <container-id> | grep pid```找到容器的主id
2. 通过 ```nsenter -t <pid> -m -n -p -u bash``` 进入到pid 内部的namespace


### ftrace 

ftrace 是一个内部的tracer 用于帮助用户定位具体内核中发生了什么事情。 主要是用于定位发生在用户空间之外的问题或者是性能指标. 与其相对的， strace 是用于分析用户空间发生事态的工具。如果与fstrace 配合得当， 则可以分析出基本上所有的问题. 
fstrace 最常见的用途就是事件跟踪。 整个内核有数百个静态事件点， 可以通过tracefs 文件系统启用这些时间来查看内核部分的状态。 ftrace 使用 tracefs 来控制。 我们可以通过以下命令启动 ```mount -t tracefs nodev /sys/kernel/tracing``` 不过我们有更方便的工具 https://github.com/brendangregg/perf-tools
