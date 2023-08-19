---
layout: post
title:  "weekly report"
date:   2023-08-06 22:30:08 +0800
categories: weeklyreport
---

# 读书

### 大话存储

# 社区

kubernetes-csi/csi-release-tools 这个 pr 已经被合并了, 多亏了这个, external-snapshotter 的 pr 也顺利通过了

# 工作

### bash 问题

本周遇到了个奇怪的问题, 在使用 mount 命令通过 grep 获取挂载点的时候, 出现了奇怪的问题

```
[root@xx1 /]# mount | grep "on /home/t4/kubernetes/lib/kubelet/plugins/kubernetes.io/csi/pv/d-xxxxx/globalmount"
/dev/vdc on /home/t4/kubernetes/lib/kubelet/plugins/kubernetes.io/csi/pv/d-xxxxx/globalmount type ext4 (rw,relatime)
[root@xx1 /]# mount | grep "on /home/t4/kubernetes/lib/kubelet/plugins/kubernetes.io/csi/pv/d-xxxxx/globalmount" | grep -v grep
```

也就是经过 grep -v grep 之后 mount 命令没有输出了. 查看内核日志也没有异常, 先在这里记录下. 后续如果有定位出来相关问题在这里补充


> 原因是 diskid 里面包含 grep 字符. 导致 grep -v grep 之后相关挂载路径失效

### 云盘挂载路径异常

现象是用户重启了应用 pod 的时候发现应用的 pod 无法启动. 查看报错信息为: /var/lib/containerd/io.containerd.runtime.v2.task/k8s.io/2270161faeabafc9275adea0ca1b71bab444ad34ccbf03ad6da12d9d2e98db4f/xxx (其中中间这串 id 是 container 的 id. 可以通过 crictl ps 获取)

挂载点残留

复现流程
就是在有 pod 挂载云盘的节点上, 重启 csi-plugin pod 之后, 节点上就出现了超长路径(runc rootfs + disk mount)

问题原因
原因是 csi-plugin 新增了 /var/run (ln -s /run) 的挂载路径, csi-plugin 同时挂载了 /var/lib/kubelet & /var/run/container 两个路径, 这两个路径在 containerd 的先后挂载顺序是完全根据路径长短来决定的, 如果路径相同, 可能会有先后挂载之分, 当 /var/lib/kubelet 先于 /var/run/container 挂载的时候,就会出现这个问题, 因为这时, container 里面已经包含 disk 的挂载路径了

### 查看容器挂载信息

可以在 宿主机上 通过 cat /proc/<process-id>/mountinfo 来查看 process-id 相关的挂载点, 在容器内可以 cat /proc/self/mountinfo 查看挂载点

### fuse 类进程回收

在卸载掉 fuse 挂载点的时候, fuse 进程会自动回收