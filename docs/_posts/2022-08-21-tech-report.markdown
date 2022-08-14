---
layout: post
title:  "weekly report"
date:   2022-08-21 22:30:08 +0800
categories: weeklyreport
---

# 读书

### 论文

https://www.usenix.org/system/files/conference/fast17/fast17-chen.pdf

本来想找一下 NFS 的论文看一下 NFS 的实现原理. 不过没想到找到了这个 vNFS. 那就顺带看了一下. 其实并没有完整看完, 只看了前面的理论部分, 后面的对比和性能相关的指标没有详细看. 简单理解一下 vNFS 是针对 NFS4.0 协议做的一个优化, 它针对小文件和文件 metadata 请求做了合并, 这样可以将多个请求合并到一起发送(降低了请求头 & 链接建立的开销). 几个点总结一下
- vNFS 只有对客户端做了修改. NFS4.0 的服务端天生支持对多个 compound 内容解析
- vNFS 通过在用户空间新建 api 来支持合并请求
    - 1. 默认 vNFS driver 是内置在内核中的
    - 2. 在用户空间扩展是因为研发/测试比较方便
- vNFS 不使用 start_component, end_component 之类的包裹函数暴露这个功能是因为这个功能涉及到过多的代码变动(复杂度过大). 为了方便起见使用了默认的


# 日常

### fsck

fsck 相当于一个协议, 接口定义, 每个文件系统都会实现这个接口. e2fsck 适用于所有的 ext文件系统, 跟 fsck.ext4/fsck.ext3/fsck.ext2 没有本质区别. Dosfsck, fsckvfat 也只是不同文件系统的 fsck 罢了

### virtIO

Virtio 是kvm中IO虚拟化的主要平台, 主要思想是为 io 虚拟化提供一个通用的框架, 目前仅支持 network/block/balloon 设备, host 侧的实现是在用户空间的 qemu 中, 所以在 host 不需要特殊的驱动, 但是在 guest 侧是需要virtio驱动的. 这周看 rund 他们的一些文档. 学习到了很多, 所以简单都记录一下

### xfs

 xfs 是一个高性能日志文件系统, 由 Silicon Graphics 创建, 由于可以触发并行的 IO流, 所以它的生产力很高.  xfs reflinks 功能可以在不同的文件之间复用同样的数据块的能力 Data Block Sharing (快速生成 vmimage/目录树 的快照,), 也提供重复数据删除和快速写时拷贝的能力. Xfs 将 iopath 的功能从基础设施中抽出. 进行了优化, 并且替换了文件系统的数据结构, 改成了 btree ?