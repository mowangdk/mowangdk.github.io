---
layout: post
title:  "weekly report"
date:   2023-10-22 22:30:08 +0800
categories: weeklyreport
---


# 读书

### 大话存储

本周无进展

### rust 

之前就有慢慢在看了, 这次干脆买一本书来看. 逐渐感觉到 rust 近几年的势头越来越好. kata 也搞了一套 rust 的实现. 甚至有人叫嚣要用 rust 重写操作系统. 虽然个人感觉是没啥必要吧. 但是可以说 rust 是目前最接近底层性能的一种语言了.学一学总没有坏处.  

# 社区

还是在做一些 csi sidecar 的 csi-release-tools 的升级工作. 

# 工作

### iohub-sriov vs pci_pf_stub

每个 pf 占用一个 pci bus, 物理意义上有限. 限制了插入设备的总数量. vf 的话可以把每一个端口设备可以变成 255 个 vf (当然都是复用同一个物理接口). 增加挂载设备的数量


### load 值异常

通过机器上的告警发现机器的 load 非常高, 通过 ```sysak loadtask -s``` 观察到 csi 的进程load 非常高, 进一步通过 ```ps -Lf```观察csi plugin 的线程发现, 有3k+/4k+ D(Uninterruptible sleep (usually IO)) 状态的线程.
```
UID          PID    PPID     LWP  C NLWP STIME TTY      STAT   TIME CMD
root     1765976 1764871 1765976  0   52 Oct10 ?        Sl     0:06 /bin/plugin.csi.alibabacloud.com --endpoint=unix://var/lib/kubelet/csi-plugins/driverplugin.csi.alibabacloud.com-replace/csi.sock
root     1765976 1764871 1765977  0   52 Oct10 ?        Sl     0:18 /bin/plugin.csi.alibabacloud.com --endpoint=unix://var/lib/kubelet/csi-plugins/driverplugin.csi.alibabacloud.com-replace/csi.sockroot     1765976 1764871 1765978  0   52 Oct10 ?        Sl     0:06 /bin/plugin.csi.alibabacloud.com --endpoint=unix://var/lib/kubelet/csi-plugins/driverplugin.csi.alibabacloud.com-replace/csi.sock
root     1765976 1764871 1765979  0   52 Oct10 ?        Sl     0:06 /bin/plugin.csi.alibabacloud.com --endpoint=unix://var/lib/kubelet/csi-plugins/driverplugin.csi.alibabacloud.com-replace/csi.sockroot     1765976 1764871 1765980  0   52 Oct10 ?        Sl     0:06 /bin/plugin.csi.alibabacloud.com --endpoint=unix://var/lib/kubelet/csi-plugins/driverplugin.csi.alibabacloud.com-replace/csi.sock
```

简单观察日志没有明显的报错, 观察测试集群正常节点基本上没有这么多异常进程. 但是线上确实出现了很多异常. 基本上可以判断确实是使用过程出现了问题. 本来想通过 pprof 来看下这些. 奈何用户不太买这个账, 于是只能继续观察日志. 发现有很多线程执行到一半突然都没有日志了. 于是观察源代码具体日志中断位置. 发现是 umount 的时候出现了异常. 尝试 umount 对应的挂载点, 发现相应的后端存储已经 unreachable. 算是定位到了具体原因, 相关优化为超时使用 -f 进行强制 umount
