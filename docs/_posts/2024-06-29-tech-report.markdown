---
layout: post
title:  "weekly report"
date:   2024-06-29 22:30:08 +0800
categories: weeklyreport
---


# 读书

### [proc](https://docs.kernel.org/filesystems/proc.html)

这个地址确实没有实时更新, 有些配置已经无效了(cpu), 并且 也新增了新的配置. 简单参考下字段就好
/proc目录包含系统上每个正在运行的进程的子目录，其名称以进程ID（PID）命名。

“self”链接指向读取文件系统的进程。每个进程子目录都有以下表格中列出的条目。

需要注意的是，对/proc/<pid> 或其包含的任何文件或子目录的打开的文件描述符. 并不阻止在<pid>退出, 也不能影响这个 pid 分配给其他进程。如果<pid>对应的进程已经死亡，对打开的/proc/<pid>文件描述符的操作永远不会对内核可能偶然分配了相同进程ID的新进程进行操作。相反，这些FD的操作通常会因ESRCH([ESRCH]No process or process group can be found corresponding to that specified by pid.)失败。 like this
```
[root@iZd7oi4drqklmlv87d7et0Z 2434496]# ls
ls: cannot open directory '.': No such process
```

|  文件  | 内容  |
|  ----  | ----  |
| cmdline  | 当前进程的命令行 |
| cwd  | 一个指向当前进程工作目录的软链 |
| exe  | 指向当前进程的可执行文件 |
| fd  | 当前进程持有的所有文件描述符 |
| status  | 当前进程的状态 human readable |
| stack  | 当前进程的堆栈信息 |
| statm  | 进程使用内存的详细信息 |


# 社区

这周简单对了下, 整体的 migration process 我还在设计, 初步分成了几个状态, 按照状态流转的方式进行说明, 预计下下周做一个内部的 presentation.

# 工作

### /proc/1/root

这个目录对应着指定进程的根目录. 如果这个进程是一号进程的话, 这个就是 os 的 rootfs.

### shared memory

/dev/shm is a Linux filesystem that stands for "Shared Memory." It is a temporary file system that resides in the system's memory instead of being stored on disk. This allows for faster access to data by multiple processes since they can share the same memory space.


### fsGroupChangePolicy.OnRootMismatch

 Only change permissions and ownership if the permission and the ownership of root directory does not match with expected permissions of the volume. This could help shorten the time it takes to change ownership and permission of a volume.