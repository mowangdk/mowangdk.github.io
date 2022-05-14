---
layout: post
title:  "weekly report"
date:   2022-05-01 22:30:08 +0800
categories: weeklyreport
---

# 读书

这周还是继续读 Understanding Linux Network Internals 这本书, 差不多把第一章都读完了,因为第一章主要介绍了一些基础语法和概念, 这些对我来说都比较熟悉了,所以即使是英文也很容易读完. 比如 mutual exclusion 这一节, 主要介绍了自旋锁, 读写自旋锁, 写时复制 这三种技术. 对于读写自旋锁有些忘了,其他的两个概念还是可以记住的, 除了 这个之外还介绍了一些常用工具, 比如 iputils, net-tools, IPROUTE2. 

# 日常

### Once.Do

这周遇到了一个神奇的问题, 当然问题出在我自己身上. 就是我这边尝试使用 kubernetes Lister 缓存 apiserver 中的资源数据的时候出现了 nil pointer 的问题, 而且这个 nil pointer 的问题还是出现在一个三方包里面.这导致定位起来比较麻烦, 后来通过在 vendor 中加 log 的方式定位出来原来是传入的 kubernetes.client 的值为空, 导致通过 client. 调用方法的时候出现了空指针异常. 但是相关的代码确实已经初始化了才对, 于是继续加 log 确定相应的 client 是否初始化成功, 然后再次进行测试, 没想到, 代码完全没有执行到这个地方,这里也困扰了很久,然后发现初始化外层包了一个 once.Do 方法, 那么是否是 sync.Once 导致的? 经过一番测试之后发现的确是这样, 我在一个 sync.Once 实例中调用了两次 once.Do 方法, 只有第一个 Do 方法执行了, 第二个完全没执行.简单以代码说明就是

```golang
import sync

once sync.Once

once.Do(func() {fmt.Printf(“run..”)})
once.Do(func() {fmt.Printf(“never run…”)})
```

本以为 once.Do 只有在多线程之间才生效, 完全没想到会直接忽略掉第二次 Do 调用. 

### runAsNonRoot vs runAsUser
这周答疑的时候发现 securityContext 配置有两个配置比较疑惑就是标题的那两个, 一个是指定 pod 要以非 0 用户启动, 一个是指定 pod 要以特定的 user 启动. 用户同时指定了 runAsNonRoot & runAsGroup 发现没有生效, 该用户依旧无法读取到被 fsGroup 设置的文件目录, 报没权限, 只有声明了特定的 runAsUser 才可以读得到, 实验了一下才知道, runAsGroup 必须和 runAsUser 绑定使用才可以生效,否则在创建 sandbox 的时候就会报错. runAsNonRoot 只是限制了启动用户不能是 root, 这个启动用户既可以是 Dockerfile 里面定义的 User, 也可以是 pod 上面声明的 runAsUser, 这两者在实际效果上没有区别, 可能唯一的区别是 Dockerfile 里面的 User 是跟着镜像走的, pod 里面的声明只是跟着 yaml 走的.

# 社区

-  还是上周的 [pr](https://github.com/kubernetes/autoscaler/pull/4798), 这周终于合并了, 可喜可贺