---
layout: post
title:  "weekly report"
date:   2022-11-18 22:30:08 +0800
categories: weeklyreport
---


# 读书

### 闪客的操作系统源码分析

前四章基本都读完了. 操作系统的基本启动流程也大概了解了. 最后一部分主要在写 shell 的启动方式. 启动最重要的也就是 ```execve()``` 方法. 这里将 /bin/sh 二进制文件传入 ```execve()``` 方法里面. 

```execve()``` 也是系统调用, 它会触发一个系统调用中断, 触发 ```execve()``` 处理函数, 将当前程序运行的 eip 压栈. ***并且将 eip 传入处理函数中***  然后将相关的参数(filename, argv, envp) 传入```do_execve()``` 里面. 

```do_execve()```函数就是主要的处理逻辑. 
- 检查权限
- 读取文件的第一块数据,用来区分文件类型
- 区分了一下二进制和 shell 文件. 两个是走的不同逻辑的(通过 shell "!#" 来区分). 
- 然后对二进制进行基本校验, 保证二进制可以正常被运行
- 将参数存放在当前进程的末尾 128K 的空间中
- 构建参数表, 也就是整理上面进程空间中的参数
- 读取/bin/sh 二进制文件的可执行文件头. 将 eip[0] 的位置指向他
- 同时修改 eip[3] 参数位置,指向我们刚刚构建好的 /bin/sh 参数

由于这里我们是处在中断处理函数中, 当中断处理函数结束之后, 之前被压栈的 eip 会重新被弹出, 由于我们已经在中断处理函数中修改了 eip 的值. 所以返回的时候会直接执行 /bin/sh 二进制. 这就是 ```execve()``` 函数的魔法, 这里是否有点既视感, 因为 当时从内核态切换到用户态的时候也是靠的这个套路.

/bin/sh 文件其实没有什么好说的, 就是不断的从键盘读取命令, fork 一个新的进程. 然后通过 ```execve()``` 函数去运行命令.基本上就是上面的重复了


# 工作

### kcm

  这周跟同事一起处理了一个 kcm 的问题, 当 kcm adcontroller log level 设
成 5 的时候会不断的刷 attached--touching xxxx 的日志, 平均 1s 会刷好几次.然而在 adcontroller 的入口 syncPeriod 则是几分钟一次. 所以疑问就来了. 代码里面既没有多个 goroutine 并发处理, resync 时间又那么长. 那么这个日志是哪里报出来的.

结果很简单, 是我这边找错了. adcontroller 的同步时长是 100ms 一次, 并且这个配置还是没有办法修改的(除非重新编译). 由于一开始找错了地方, 我还反复问了社区的人好几次. 有够社死的


### kubelet

  这周同样出现了 kubelet 的报错. 说是找不到 volumes 的声明, 报如下错误

```
Nov 15 20:20:04 ack-prod-node-1 kubelet: E1115 20:20:04.147758    1364 kubelet_pods.go:154] Mount cannot be satisfied for container "golang", because the volume is missing (ok=false) or the volume mounter (vol.Mounter) is nil (vol={Mounter:<nil> BlockVolumeMapper:<nil> SELinuxLabeled:false ReadOnly:false InnerVolumeSpecName:}):
 {Name:bd-prod-buriedpoint-vol ReadOnly:false MountPath:/wwwroot/log SubPath: MountPropagation:<nil> SubPathExpr:}

Nov 15 20:20:04 ack-prod-node-1 kubelet: E1115 20:20:04.810310    1364 kubelet_pods.go:154] Mount cannot be satisfied for container "golang", because the volume is missing (ok=false) or the volume mounter (vol.Mounter) is nil (vol={Mounter:<nil> BlockVolumeMapper:<nil> SELinuxLabeled:false ReadOnly:false InnerVolumeSpecName:}):
 {Name:configcm ReadOnly:false MountPath:/configs SubPath: MountPropagation:<nil> SubPathExpr:}
```
翻了下社区代码, 相关位置是在 [kubernetes](https://github.com/kubernetes/kubernetes/blob/d1c0171aed848900daa07212370c991c19c318b1/pkg/kubelet/kubelet_pods.go#L167) 这里, 但是不清楚什么时候会触发这个问题.

### udev

上周第一次接触, 这周整理一下, udev 是一个 systemd 守护进程. 主要是监听内核发出的设备修改的事件, 处理所有的状态变更. 相关的变更可以通过 ```dmesg -T```获取.

我们可以使用 ```systemctl status systemd-udevd``` 来看守护进程的状态.

严格来说, udev 的工作方式是试图将它收集到的每一个系统事件与 ```/lib/udev/rules.d``` 目录下的规则进行匹配. 规则文件包括匹配符合分配键. 我们以下面的规则为例

```
ACTION=="add", SUBSYSTEM=="net", SUBSYSTEMS=="usb", NAME=="", \
    ATTR{address}=="?[014589cd]:*", \
    TEST!="/etc/udev/rules.d/80-net-setup-link.rules", \
    TEST!="/etc/systemd/network/99-default.link", \
    IMPORT{builtin}="net_id", NAME="$env{ID_NET_NAME_MAC}"
```

这个规则匹配 设备 Add 动作, 只要这个设备属于网络子系统, 是 一个 USB的设备, 没有名称, & 匹配剩余规则. 我们就给他的 Name 赋值为 NAME="$env{ID_NET_NAME_MAC}



