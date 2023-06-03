---
layout: post
title:  "weekly report"
date:   2023-05-26 22:30:08 +0800
categories: weeklyreport
---

# 读书

### Linux命令行与shell脚本编程大全

还是有很多收获的, 

#### 文件默认权限

- umask
默认为 0022 如果没有特别指定的话, 默认创建的文件权限就是 0666 - 0022 = 0644, 默认创建的文件夹的权限就是 0777 - 0022 = 0755

- shared
可以通过 ```chgrp shared testdir/ && chgrp g+s testdir/```  来创建共享目录, 这里还需要再确认下, 尤其是 nfs 或者是 fuse 这种共享文件组的行为

#### 文件系统

这部分其实没啥好说的, 文件系统也发展了这么多年了, 不断地在数据安全(日志系统) 和 读写效率之间做优化. 我认为在上面这层已经没有什么可以继续研究的. 现在更热门的趋势感觉是将所有的文件系统底层替换成网络存储 or 按需加载的形式. 属于锦上添花类型的了

#### shell 的介绍

其实这部分也只是大致的过一遍, 算是查缺补漏了. 其实没有什么可以详细说的. 当初开始看这本书也只是因为最近需要用到 shell 的地方有点多, 需要捡起来而已


# 工作

### 处于 terminating 状态的 pod 卸载问题

总算是弄明白了之前为啥总会出现的一个问题了, 当 pod 处于 terminating 状态, kubelet 进行重启的时候, kubelet 内部会对节点上的 pod 进行 reconstruct, terminating 状态的 pod 不会放到 node DSOW 里面, 但是在 ASOW 里面 pods 是根据 volumeHandler 放到 一个 mapping 结构里面, 并且这个 mapping 结构里面 volumes 关联的 pod 也不是一个 array. 这就导致 kubelet 在 reconcile 的时候只会处理一个 pod, 导致了使用了这个存储的其他 pod 无法被处理. 


### lvm pv vg 问题

之前一直以为重启之后 相关设备的设备符是可以按照挂载顺序重新挂载到机器上, (如果设备符已经被 mount 到路径上确实会这样, 因为在 /etc/fstab 里面会根据 uuid(文件系统) 去关联这个设备符, 进而关联设备符挂载路径), 同理, vg 也是有 uuid 的, 会维护设备符和vg 的对应关系, 所以在节点重启的时候, 设备符依旧会按照 uuid 来绑定对应的 vg



