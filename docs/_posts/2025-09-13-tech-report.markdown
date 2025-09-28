---
layout: post
title:  "weekly report"
date:   2025-09-13 22:30:08 +0800
categories: weeklyreport
---


# 读书

最近没有读什么书， 有些懈怠

# 社区

### git log

前几周提了一个单独的 shell 脚本， 没想到上周 mauricio 直接提了一个合并好的 pr， 速度还挺快， 这里简单分析下

# 工作

### CAP_SYS_ADMIN

通过在 pod 上指定 allowPrivilegeEscalation 来授予容器内的进程 CAP_SYS_ADMIN 权限， CAP_SYS_ADMIN 是一种新的 root 权限。 基于 capabilitys 可以控制给用户的权限范围，仅授予用户需要的部分特权。可以保证即使出现泄露的情况，受影响的权限依旧可控， 细分的权限可以参考： https://man7.org/linux/man-pages/man7/capabilities.7.html， 权限被 attach 到二进制，相关二进制运行的时候就可以以指定的权限运行.

---
CAP_SYS_ADMIN 不能做什么（它需要其他权能）：

不能 绑定到 1024 以下的端口（这需要 CAP_NET_BIND_SERVICE）。
不能 覆盖文件的所有权或权限检查（这需要 CAP_DAC_OVERRIDE）。
不能 杀死任意进程（这需要 CAP_KILL）。
不能 加载内核模块（这需要 CAP_SYS_MODULE）。
不能 修改系统时间（这需要 CAP_SYS_TIME）。


可以做什么？
- perform operations on ***trusted*** and ***security*** extended attributes (see xattr(7));

一旦挂载了CAP_SYS_ADMIN， 节点上的 /sys 目录将会直接映射到容器内， 这部分是通过 xxx capacity 完成的

---
如果容器上设置了 privileged 权限，容器可以直接访问宿主机上的所有设备，即 /dev 目录被完整挂载进来, 同样 /sys 目录也会直接映射到容器内

### 