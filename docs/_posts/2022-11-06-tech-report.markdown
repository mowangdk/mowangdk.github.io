---
layout: post
title:  "weekly report"
date:   2022-11-06 22:30:08 +0800
categories: weeklyreport
---


# 读书

### 闪客的操作系统源码分析

```c
void main(void) {
    move_to_user_mode()
    if (!fork()) {
        // 1 号进程逻辑
        init()
    }
    for(;;) pause();
}
```

这周从 1 号进程的 init 方法开始看起, 首先 init 方法使用 setup() 从指定内存(之前 setup.s 将硬件信息存储在内存的指定位置0x90080)处读取硬盘信息, 它将硬盘信息放在 hd_info []struct{} , 将每一块盘的分区信息放在 hd []struct{} 数据结构里面. 然后调用 rd_load() 方法初始化 ramdisk.  如果没有ramdisk则直接跳过. 最后就直接开始加载根文件系统. 说白了,加载根文件系统就是要将硬盘上的文件系统的信息跟内存中的数据结构对应起来, 这样我们在可以通过在内存中操作数据结构来实施对硬盘的操作. 下面我们以 MINUX 来简单列一下

引导块 -> 超级块 -> inode位图 -> 数据块位图 -> inode0 -> ... inodeN -> block0 -> .... -> blockN

这些其实也是某一个分区内的一个数据基本分布情况. 其中超级快存储这个分区内的一些基本元信息(比如, inode 节点数量, 块数量, 两种位图占用的块数, 起始数据的位置). 从这里可以看出来数据盘里面的空间并没有完全占用, 还是需要有一些信息来指导我们如何找到这些数据的, 当然, 如果你可以在外部构建数据结构直接使用 dd command 读取裸设备. 

然后就是内存中的数据结构, 简单列一下

```
struct file {
    unsigned short f_mode;
    unsigned short f_flags; 
    unsigned short f_count; # 每一个文件被引用的次数
    struct m_inode * f_inode; #对应当前文件的 inode 信息
    off_t fpos;
}

file file_table[64]{}
```

接着 mount_root 将文件系统内的超级块, 引导块, inode 位图, block 位图, 根 inode 信息, 并且将根 inode 信息设置为当前进程(1号进程)的工作目录和根目录. 就此整个根文件系统的挂载就完成了

其中 file_table[64] 代表当前机器里面最多只能同时打开 64 个文件, 打开进程打开一个文件的具体步骤如下

task_struct ->filp[20] -> file_table[64] -> file_inode_by_name(/dev/tty0) -> 填充 file struct 信息 -> 根据 inode 信息找到数据块 -> 返回 fd 对象供进程读写


# 工作

这周主要是在整理文档, 没有做什么实质性的工作. 基本这样了. 下周双十一出差中. 希望可以获取到一些新输入