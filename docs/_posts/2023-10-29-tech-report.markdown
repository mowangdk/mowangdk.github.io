---
layout: post
title:  "weekly report"
date:   2023-10-29 22:30:08 +0800
categories: weeklyreport
---


# 读书

### 大话存储

本周无进展

### rust 

没有什么进展

# 社区

CIS: Center for internet security. 估计以后想要走这个认证难了...认证是多么重要..


# 工作

### 编译器 vs 解释器 vs JIT

compilter: It's a program translator that **converts a program from one language to  another language** (high level -> low/high level, low level -> high level) Classic compilation normally is made up of several pass: frond-end (language process), middle-end (optimization), and back-end (code generation). The modern compilation might only have less/more passes. For Java, Javac is compiler that converts .java to .class.

interpreter: a program that fetches one instruction, runs it, and then fetches the next. For Java, the instruction is bytecode instruction. Normally, an interpreter is a software interpreter (CPU is a hardware interpreter for assemble instruction ).

JIT: Runtime + compilter, For Java, the JIT compiler is a compiler that translate bytecode to native machine code at program runtime. The JIT compiler has more features than a classic compiler, e.g., profiling to identify hot bytecode blocks.


### 定位 container running, 但是相关进程不存在的问题

这周发现了一个新的问题, 在节点上使用 docker 命令看一个容器是存在的, 但是这个容器对应的进程不存在. 需要定位

首先:

```docker ps | grep xxx```

723d7351c06e        registry-vpc.cn-hangzhou.aliyuncs.com/xxxx      "/entrypoint.sh --en…"   14 months ago       Up 14 months         k8s_xxx_kube-system_c4eea4e5-5410-4344-91f1-ed289c4d29d6_0

然后查看这个容器的 shim & runc 进程(均可以通过 containerid 拿到) 

```ps aux | grep 723d7351c06e | grep -v grep```

root       25229  0.0  0.0 109104  7344 ?        Sl    2022  15:49 containerd-shim -namespace moby -workdir /var/lib/containerd/io.containerd.runtime.v1.linux/moby/723d7351c06e5a3a7629b6d350c055fc20f480db77a58209eff7543b7b41a05f -address /run/containerd/containerd.sock -containerd-binary /usr/bin/containerd -runtime-root /var/run/docker/runtime-runc -systemd-cgroup
root     4013983 86.1  0.0 309840 16476 ?        Sl   Sep08 58124:15 runc --root /var/run/docker/runtime-runc/moby --log /run/containerd/io.containerd.runtime.v1.linux/moby/723d7351c06e5a3a7629b6d350c055fc20f480db77a58209eff7543b7b41a05f/log.json --log-format json --systemd-cgroup kill --all 723d7351c06e5a3a7629b6d350c055fc20f480db77a58209eff7543b7b41a05f 9

查看runc 进程以及日志, 查看runc 进程 stack

```cat /proc/4013983/stack```

[<0>] futex_wait_queue_me+0xbd/0x120
[<0>] futex_wait+0x141/0x210
[<0>] do_futex+0x5b9/0xca0
[<0>] __se_sys_futex+0x6e/0x150
[<0>] do_syscall_64+0x5b/0x1b0
[<0>] entry_SYSCALL_64_after_hwframe+0x44/0xa9
[<0>] 0xffffffffffffffffca

查看runc进程相关子进程
```ls /proc/4013983/task/```

out: 4013983/ 4013984/ 4013985/ 4013986/ 4013987/ 4013988/ 4014094/ 75581/


```pstree -sp 4013983```

systemd(1)───containerd(18362)───containerd-shim(25229)───runc(4013983)─┬─{runc}(4013984)
                                                                        ├─{runc}(4013985)
                                                                        ├─{runc}(4013986)
                                                                        ├─{runc}(4013987)
                                                                        ├─{runc}(4013988)
                                                                        ├─{runc}(4014094)
                                                                        └─{runc}(75581)


查看进程持有的 fd 文件

 ls /proc/4013983/fd

total 0
lr-x------ 1 root root 64 Sep 12 02:23 0 -> /dev/null
l-wx------ 1 root root 64 Sep 12 02:23 1 -> pipe:[108217989]
l-wx------ 1 root root 64 Sep 12 02:23 2 -> pipe:[108217989]
l-wx------ 1 root root 64 Sep 12 02:23 3 -> /run/containerd/io.containerd.runtime.v1.linux/moby/723d7351c06e5a3a7629b6d350c055fc20f480db77a58209eff7543b7b41a05f/log.json
lrwx------ 1 root root 64 Sep 12 02:23 4 -> anon_inode:[eventpoll]
lrwx------ 1 root root 64 Sep 12 02:23 5 -> socket:[108255841]
lrwx------ 1 root root 64 Sep 12 02:23 6 -> socket:[108245729]
l-wx------ 1 root root 64 Sep 12 02:23 7 -> /sys/fs/cgroup/freezer/kubepods.slice/kubepods-besteffort.slice/kubepods-besteffort-podc4eea4e5_5410_4344_91f1_ed289c4d29d6.slice/docker-723d7351c06e5a3a7629b6d350c055fc20f480db77a58209eff7543b7b41a05f.scope/freezer.state

发现有一个 cgroup 一直在尝试冻结, 查看 cgroup 状态

```cat /sys/fs/cgroup/freezer/kubepods.slice/kubepods-besteffort.slice/kubepods-besteffort-podc4eea4e5_5410_4344_91f1_ed289c4d29d6.slice/docker-723d7351c06e5a3a7629b6d350c055fc20f480db77a58209eff7543b7b41a05f.scope/freezer.state```

FREEZING

发现是进程一直在尝试冻结 pod 进行回收, 但是一直失败. 查看对应 cgroup 目录下存在什么文件

```ls /sys/fs/cgroup/freezer/kubepods.slice/kubepods-besteffort.slice/kubepods-besteffort-podc4eea4e5_5410_4344_91f1_ed289c4d29d6.slice/docker-723d7351c06e5a3a7629b6d350c055fc20f480db77a58209eff7543b7b41a05f.scope/```

cgroup.clone_children    cgroup.procs             freezer.parent_freezing  freezer.self_freezing    freezer.state            notify_on_release        tasks                    

查看 cgroup procs 进程

```cat /sys/fs/cgroup/freezer/kubepods.slice/kubepods-besteffort.slice/kubepods-besteffort-podc4eea4e5_5410_4344_91f1_ed289c4d29d6.slice/docker-723d7351c06e5a3a7629b6d350c055fc20f480db77a58209eff7543b7b41a05f.scope/cgroup.procs```

...
4185068
4190846
4191596
4192631

从上面随意选取一个进程, 查看相关命令, 发现进程 d 住了, 这也就是 runc 回收子进程失败的原因
#ps aux | grep 4185068
root     2586346  0.0  0.0 112824  2316 pts/0    S+   16:37   0:00 grep --color=auto 4185068
root     4185068  0.0  0.0   4408   768 ?        D    Aug02   0:00 rm -rf /p00/local-bc24ac68-xxxx


这里详细看下 cgroup freezer 的作用, 其实主要也就是批量操作 group 组内进程. 这里只是被容器用于回收cgroup 内的进程资源.
