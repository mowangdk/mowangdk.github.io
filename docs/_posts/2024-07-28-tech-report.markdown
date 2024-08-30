---
layout: post
title:  "weekly report"
date:   2024-07-28 22:30:08 +0800
categories: weeklyreport
---

# 读书

### 计算机网络, 自顶向下

忘了之前读的是那一本书了, 当时只觉得计算机网络啃起来跟天书一样. 读的很慢, 过了几年之后, 今天在首图居然读了一大半, 这种回头再读同样的书的感觉确实很令人惊奇.

回归到这本书, 并没有都读完, 还剩下了网络层的控制面一章和后面的安全的章节没有读. 其他的倒是大致的都过了一遍, 收获还是有一些的. 首先, 路由器的接收 buffer 和发送 buffer 存在分组, 如果在没有发送端的 buffer 的情况下, 接收端的 buffer 中发往同一个发送端的多个包会进行排队. 例如, 多个不同接收分组的首个包都是要发往同一个发送端, 这时候就会排队, 这时候不同接收分组的下一个待发送的包就会进行等待. 那是否加了发送端的 buffer 就万事大吉了呢? 在某种程度上是的. 但是这个 buffer 也不能无限的增长, 因为路由器的性能是有限的, 一味地增加 buffer 看似可以避免某个程度上排队的问题. 但是无法从根本上解决问题.也会影响其他路由器,甚至于客户端的相关路由选择以及客户端策略的指定.

还有一个就是 http2.0 的分帧传输的问题, 主要是为了解决 (HeadOfLineBlocking) 的问题. 他举了一个很好的例子, 假如一个网页上是由一个大视频和多个小组件共同组成的, 在 http2.0 之前, 如果用一个 tcp 传输链接的话这个大视频的加载就会阻塞后面所有小组件的加载. 为了解决这个问题只能建立多个 tcp 链接. 但是这样多个 tcp 链接又会造成服务端的资源耗尽.实在不是一个很好的做法. 那么 http2.0 引入了帧的概念. 将所有的传输内容分帧. 简单来说就是对传输对象进行分片. 这样大视频就会被分成数十帧甚至更多, 小组件和视频帧在同一个 tcp 信道中传输.这样就不存在阻塞加载的问题, 本身视频这种形式也是需要长时间才能消费完的对象.这种修改完全可以接受

# 社区
### kep: Recovery from resize failure 
https://github.com/kubernetes/enhancements/blob/master/keps/sig-storage/1790-recover-resize-failure/README.md


详细看了下这个 kep, 主要是为了解决扩容失败的时候用户无法介入的问题. 但是同时也引入了一个问题, 当一个恶意用户快速进行扩缩容的时候会导致底层文件系统已经扩容了, 但是用户把相关的声明改回来了, 大多数底层存储都没有提供相关接口进行缩容. 会导致声明和实际的存储空间不一致. 算是这个 kep 引入的问题. 因为这个他们在 pvc status 里面新增了一个字段, ```pvc.Status.AllocatedResources```  AllocatedResources 这个字段只能有 resize-controller 在扩容失败的时候才能缩小, 最后实际的值是 max(pvc.Spec.Capacity, pvc.Status.AllocatedResources) 可以某种程度上防范这个值


一些限制
 - 缩小后的 pvc.Spec.Resoures 的值必须 > pvc.Status.Capacity
 - 只有 apiserver 和 resize-controller 才能设置 AllocatedResources 这个值

# 工作

### mount error no space left on device

原因就是 /proc/sys/fs/mount-max 这个文件导致, 应该是 docker 运行时的一个 bug, 切换成 containerd 就正常了

说明: https://www.kernel.org/doc/Documentation/sysctl/fs.txt

This denotes the maximum number of mounts that may exist
in a mount namespace.

这个文件是一个 pseudo-system 用于在 os 运行时调整 os 系统参数. 这里 no space left on device 不仅仅是实际的存储空间, 并且也是内核允许的最大挂载点


docker issue: https://github.com/moby/moby/pull/38993


### mount discard

If the filesystem is mounted with discard, then deleting files will automatically cause the TRIM command to be issued. This often has a negative performance impact, so it's generally better not to use that mount option and to instead run fstrim periodically

TRIM is a command for the ATA interface. As you use your drive, changing and deleting information, the SSD needs to make sure that invalid information is deleted and that space is available for new information to be written. Trim tells your SSD which pieces of data can be erased.

If the Trim command did not exist (as was the case before Windows® 7), then the solid state drive would not know that certain sectors in the drive contained invalid information until the computer told the drive to write new information to that location. The drive would need to erase the existing information, then write the new information. This takes slightly more time to do than just writing the new information, so using Trim and Active Garbage Collection helps your SSD perform write commands more quickly.

Trim also affects the longevity of the solid state drive. If data is written and erased from the same NAND cells all the time, those cells will lose integrity. For optimum life, each cell should be utilized at roughly the same rate as other cells. This is called wear leveling. The Trim command tells the SSD which cells can be erased during idle time, which also allows the drive to organize the remaining data-filled cells and the empty cells to write to to avoid unnecessary erasing and rewriting. 
