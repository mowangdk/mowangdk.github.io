---
layout: post
title:  "weekly report"
date:   2024-02-24 22:30:08 +0800
categories: weeklyreport
---


# 读书

### Rust 程序设计

最近在回家的路上还是读了一些章节的, 主要是一些关于 Rust 语言的一些特性. 简单写一下给我印象比较深刻的几个点吧

- 错误处理相对于 go 来说, rust 的错误处理确实优雅了很多, 不需要满屏幕的 if err != nil 这样的判断, 可以直接优雅的以 ? 来处理错误返回. 
- rust 的枚举感觉可以做任何事情. 可以通过枚举来自定义错误, rust 的枚举还可以携带值, 甚至是不同的值. 算是见到了新玩意, 在 rust 里面一般函数的返回值也是枚举, 比如它可以返回一个元组, 也可以返回一个 error. 这也是与其他我接触过的语言中最不一样的.


# 社区

### csi sidecar AIO

上周四跟 mauricio 开了一个会, 简单介绍了一下项目目前的进度. 以及后面要做的事情. 相关的文档整理也已经做了一部分了. 关于后面的的事情这周还没开始推进, 准备明天去杭州的时候仔细搞下. 假期写了如下的文档

https://docs.google.com/document/d/1z7OU79YBnvlaDgcvmtYVnUAYFX1w9lyrgiPTV7RXjHM/edit#heading=h.7itdhlmd5uwp

https://docs.google.com/document/d/1AKqJeAlBL8PkH8D9zABCZ82Bk1N46EygKPvVh5p4-qU/edit#heading=h.exd9zenqe3rr



# 工作

### sriov 模式新理解

sriov 是将设备进行虚拟化的技术. 也就是会将一个物理端口虚拟化成 256 个逻辑接口, 这些逻辑接口都会关联一个实际的物理设备. 这就可以满足多挂载的需求, 但是这里问题来了. 因为都是虚拟化的接口, 这些接口在实际设备释放的时候并不会直接释放, 而是直接回绑定到一个空设备. 所以这个时候这个设备上存在的配置(driver)并不会 reset. 导致如果这个端口重新被复用的时候. 还是会使用之前的配置. 这个跟我之前的理解是不一样的

### panic: protobuf tag not enough fields in Status.state

工作的时候遇到了这个错误. 是 ttrpc 的一个报错, ttrpc 是基于 grpc 的一个协议. 拥有更加轻量级的特征. 但是他内部依赖了一个名为 gogo/protobuf 的库, 并且这个库已经被废弃了. 上面的问题便是由上面的那个库报的. 但是问题是这个库可能还没办法直接升级. 因为依赖侧没有办法升级. 所以这个问题解决起来可能还比较麻烦 https://github.com/containerd/ttrpc/issues/62

### ephermeral volume limit

目前在 runc 场景下确实没有什么好方式可以在不驱逐节点的情况下限制 pod ephemeral volume 的使用量. 社区的方案算是最简单也是最易行的方案了. 不过也确实有使用了 du, 再加强制驱逐的问题.

