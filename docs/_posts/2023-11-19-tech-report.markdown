---
layout: post
title:  "weekly report"
date:   2023-11-19 22:30:08 +0800
categories: weeklyreport
---


# 读书

### 大话存储

本周回家了, 没有什么实际进展


# 社区

本周组内同学提了一个关于 adcontroller 的优化 pr. 实际看了之后发现实际是个很白痴的问题. 不知道当时为啥会出现这种问题. pr: https://github.com/kubernetes/kubernetes/pull/121576

# 工作

### 云盘挂载了, 但是无法在机器上找到相关设备符

上周遇到了这个问题. 当用户反复挂载同一块云盘的情况下, 重新插盘的时候大概率会出现云盘在机器上找不到设备符的情况. 排查流程基本如下

1. 首先看了下机器上现有的设备符对应的 bdf.
```
ls -l /sys/block/{dev-name}
```

2. 再看对应 vfio 下面对应的 bdf

```
ls /dev/vfio

342/
349/
350/

lsof /dev/vfio/342
Command  PID  USER   FD    TYPE  DEVICE  SIZE/OFF NODE NAME
xxx      xxx  root   62u   CHR   238,26   0t0   xxxx   /dev/vfio/342

ls /sys/kernel/iommu_groups/342/devices
0000:1d:01.4
```

将所有的 vfio 对应的 bdf 和 deviceid 对应的 bdf 进行匹配, 再跟管控侧的数据进行比对, 可以得出缺失的 bdf code(bdf 按序递增)


3. 经判断, 确实是在插入的时候就没有绑定 nvme/vfio 上, 找了driver 同学看了下相关日志

11:09:32 bdf对应的盘被detach, bdf 对应的盘资源释放，bdf 21:01.4 处于空闲
11:12:10 host对这个bdf一次nvme驱动加载， 这次加载报错
11:12:14 收到attach 盘，分配的bdf是21:01.4，加载驱动时候发现已经设备已经被绑在nvme上，驱动没有正常加载，host上无法看到盘

4. 为了追踪这台机器上具体的一个插入时序, 开启如下命令 tracing

```
### 设置trace function
echo function > /sys/kernel/debug/tracing/current_tracer

### 设置需要trace的函数，bind_store 当有用户要绑驱动的时候会调用，nvme_probe 在nvme 加载驱动的时候会调用

echo bind_store nvme_probe > /sys/kernel/debug/tracing/set_ftrace_filter 

cat /sys/kernel/debug/tracing/set_ftrace_filter

### 开始trace
echo 1 > /sys/kernel/debug/tracing/tracing_on

### 出现问题可以查看trace 记录

cat /sys/kernel/debug/tracing/trace

```

5. 结论(后续更新)


### 云盘回收无法被正常处理


现象: 
sh-4.2# rm -r /var/lib/kubelet/plugins/kubernetes.io/csi/diskplugin.csi.alibabacloud.com//globalmount
rm: cannot remove '/var/lib/kubelet/plugins/kubernetes.io/csi/diskplugin.csi.alibabacloud.com/***/globalmount': Device or resourc
e busy

通过 ```lsof /var/lib/kubelet/plugins/kubernetes.io/csi/diskplugin.csi.alibabacloud.com/***/globalmount``` 没有找到任何输出, 
反而通过  
```
sh-4.2# fuser -mv /var/lib/kubelet/plugins/kubernetes.io/csi/diskplugin.csi.alibabacloud.com/***/globalmount
                     USER        PID ACCESS COMMAND
/var/lib/kubelet/plugins/kubernetes.io/csi/diskplugin.csi.alibabacloud.com/***/globalmount:
                     root     kernel mount /
                     root          1 .rce. systemd
                     root          2 .rc.. kthreadd
                     root          6 .rc.. ksoftirqd/0
                     root          7 .rc.. migration/0
                     root          8 .rc.. rcu_bh
                     root          9 .rc.. rcu_sched
                     root         10 .rc.. lru-add-drain
                     root         11 .rc.. watchdog/0
                     root         12 .rc.. watchdog/1
                     root         13 .rc.. migration/1
                     root         14 .rc.. ksoftirqd/1
                     root         16 .rc.. kworker/1:0H
                     root         17 .rc.. watchdog/2
                     root         18 .rc.. migration/2
                     root         19 .rc.. ksoftirqd/2
                     root         22 .rc.. watchdog/3
                     root         23 .rc.. migration/3
                     root         24 .rc.. ksoftirqd/3
                     root         26 .rc.. kworker/3:0H
                     root         29 .rc.. netns
                     root         30 .rc.. khungtaskd
                     root         31 .rc.. writeback
                     root         32 .rc.. kintegrityd
                     root         33 .rc.. bioset
                     root         34 .rc.. bioset
                     root         35 .rc.. bioset
                     root         36 .rc.. kblockd
                     root         37 .rc.. md
                     root         38 .rc.. edac-poller
                     root         39 .rc.. watchdogd
                     root         46 .rc.. kswapd0
                     root         47 .rc.. ksmd
                     root         49 .rc.. crypto
                     root         57 .rc.. kthrotld
                     root         59 .rc.. kmpath_rdacd
                     root         60 .rc.. kaluad
                     root         61 .rc.. kpsmoused
                     root         62 .rc.. ipv6_addrconf
                     root         75 .rc.. deferwq
                     root        112 .rc.. kauditd
                     root        141 .rc.. ib-comp-wq
                     root        142 .rc.. kworker/u9:0
                     root        144 .rc.. ib-comp-unb-wq
                     root        145 .rc.. ib_mcast
                     root        146 .rc.. ib_nl_sa_wq
                     root        216 .rc.. mlx5_ib_sigerr_
                     root        223 .rc.. rdma_cm
                     root        280 .rc.. ttm_swap
                     root        298 .rc.. jbd2/vda3-8
                     root        299 .rc.. ext4-rsv-conver
                     root        421 frce. systemd-udevd
                     root        532 .rc.. nfit
                     root        567 Frce. auditd
                     root        571 .rc.. rpciod
                     root        572 .rc.. xprtiod
                     root        586 ....m runsvdir
                     polkitd     593 .rce. polkitd
                     root        595 .rce. systemd-logind
                     dbus        596 .rce. dbus-daemon
                     rpc         615 .rce. rpcbind
                     chrony      617 .rce. chronyd
                     root        621 Frce. gssproxy
                     root        668 .rce. sh
                     root        809 .rc.. bond0
                     root        927 ....m runsv
                     root        928 ....m runsv
                     root        929 ....m runsv
                     root        932 ....m runsv
                     root        933 ....m runsv
                     root        934 ....m runsv
                     root        935 ....m runsv
                     root        936 ....m runsv
                     root        939 ....m calico-node
                     root        940 ....m calico-node
                     root        941 ....m calico-node
                     root        943 ....m calico-node
                     root        944 ....m calico-node
                     root        945 ....m calico-node
                     root        951 Frce. dhclient
                     root       1019 Frce. tuned
                     root       1087 .rc.. kworker/0:1H
                     root       1186 Frce. master
                     postfix    1188 .rce. qmgr
                     root       1311 .rce. atd
                     root       1314 .rce. crond
                     root       1333 .rce. agetty
                     root       1433 Frce. rstatd
                     root       1442 ....m bird
                     root       1443 ....m bird6
                     root       1796 ....m sh
                     root       2275 .rce. sh
                     root       4655 .rc.. jbd2/vdc-8
                     root       4656 .rc.. ext4-rsv-conver
                     root       5862 .r.e. containerd-shim
                     (unknown)   5882 ....m pause
                     (unknown)   5937 ....m filebeat
                     root       6661 .rc.. kworker/0:2H
                     root       7095 ....m sh
                     root       7464 .rc.. kworker/0:2
                     root       7731 .rce. sh
                     root       8005 .rc.. kworker/u8:0
                     root       8301 .rc.. kworker/1:2H
                     root       8748 .rce. kubelet
                     root       9353 .rc.. kworker/1:2
                     postfix    9409 .rce. pickup
                     root      10315 .rc.. kworker/2:0
                     root      10615 ....m sh
                     root      10903 .rc.. jbd2/vdd-8
                     root      10904 .rc.. ext4-rsv-conver
                     root      11036 Frce. AliYunDunUpdate
                     root      11281 .rce. sh
                     root      11325 .rce. bash
                     root      11347 .rc.. kworker/u8:1
                     root      11749 ....m sh
                     root      12170 .r.e. containerd-shim
                     root      12323 .rce. sh
                     root      15211 .rce. rsyslogd
                     root      17974 .rce. kube-proxy
                     root      18478 Frce. AliYunDun
                     root      18490 Frce. AliYunDunMonito
                     root      19725 .rc.. kworker/3:3
                     root      19791 .rc.. kworker/2:3
                     root      19970 .r.e. containerd-shim
                     (unknown)  19991 ....m pause
                     root      20015 ....m sh
                     root      20023 ....m csi-node-driver
                     root      20055 ....m csi-node-driver
                     root      20088 ....m csi-node-driver
                     root      20116 .rc.. kworker/1:0
                     root      20122 ....m entrypoint.sh
                     root      20241 Frce. csiplugin-conne
                     root      20313 F...m plugin.csi.alib
                     root      20746 .rce. sh
                     root      22574 .rc.. kworker/0:0
                     root      23279 .rc.. kworker/3:1H
                     root      25126 Frce. aliyun-service
                     root      25466 Frce. assist_daemon
                     root      26688 .r.e. containerd-shim
                     root      28903 .rc.. nfsiod
                     root      29199 Frce. kube-lb
                     root      29200 Frce. kube-lb
                     root      29728 Frce. etcd
                     root      29930 .rce. sshd
                     root      30026 .r.e. containerd-shim
                     (unknown)  30045 ....m pause
                     root      30116 .rc.. jbd2/vdb-8
                     root      30117 .rc.. ext4-rsv-conver
                     root      30622 Frce. containerd
                     root      30887 .rc.. kworker/3:1
                     root      31108 Frce. systemd-journal
                     root      31198 ....m node-cache
                     root      31591 .rc.. kworker/2:0H
                     root      31684 .rc.. kworker/2:1H
                     root      31721 .rce. login
                     root      31934 ....m sh
                     root      32187 .r.e. containerd-shim
                     (unknown)  32210 ....m pause
```

正常的输出

```
sh-4.2# fuser -mv /var/lib/kubelet/plugins/kubernetes.io/csi/diskplugin.csi.alibabacloud.com/***/globalmount
                     USER        PID ACCESS COMMAND
/var/lib/kubelet/plugins/kubernetes.io/csi/diskplugin.csi.alibabacloud.com/***/globalmount:
                     root     kernel mount /var/lib/kubelet/plugins/kubernetes.io/csi/diskplugin.csi.alibabacloud.com/***/globalmount
ount
```


这里明显是有问题的

顺带贴一下 fuser 的 ACCESS meaning

```
       fuser displays the PIDs of processes using the specified files or
       file systems.  In the default display mode, each file name is
       followed by a letter denoting the type of access:

              c      current directory.
              e      executable being run.
              f      open file.  f is omitted in default display mode.
              F      open file for writing.  F is omitted in default
                     display mode.
              r      root directory.
              m      mmap'ed file or shared library.
              .      Placeholder, omitted in default display mode.

       fuser returns a non-zero return code if none of the specified
       files is accessed or in case of a fatal error.  If at least one
       access has been found, fuser returns zero.

       In order to look up processes using TCP and UDP sockets, the
       corresponding name space has to be selected with the -n option.
       By default fuser will look in both IPv6 and IPv4 sockets.  To
       change the default behavior, use the -4 and -6 options.  The
       socket(s) can be specified by the local and remote port, and the
       remote address.  All fields are optional, but commas in front of
       missing fields must be present:

       [lcl_port][,[rmt_host][,[rmt_port]]]

       Either symbolic or numeric values can be used for IP addresses
       and port numbers.

       fuser outputs only the PIDs to stdout, everything else is sent to
       stderr.

```

确认了下, 这种情况一般是有其他的 namespace 中还存在相关挂载点, 导致 delete 失败. 需要进一步排查相关引用