---
layout: post
title:  "weekly report"
date:   2024-12-10 22:30:08 +0800
categories: weeklyreport
---


# 读书

### ebpf learning

之前团队同学有分享了一下这本书，正好有一个bdf，就在上下班的路上开读，现在差不多读完了1/4, 整体看算是个入门书籍吧，先给你个 helloworld 用例，给你大概讲一下能做什么以及怎么做，随后开始剖析相关原理了，目前也就是大概扫一遍吧
最近发现了确实要速读，速读的效率比去纠结每一个知识点要快上许多。但是也不是那种把文章一股脑丢给gpt，只总结一个大纲，这样我觉得还是差点东西，差点东西的意思是能否把知识串联成体系，自己融汇贯通。一些文章我觉得用速读看下还挺好的。书籍的话还是算了


### CAP in etcd

首先，CAP 定义是：

- C: Consistency (data consistency)
- A: Availability (high availiability of access data)
- P: Partition tolerance

CAP 是指在网络分区无法避免的情况下，只能一致性或者是高可用，两者无法同时满足。 也就是说，他们要么牺牲数据一致性来保证网络分区时的服务可用性，要么确保数据一致性但可能牺牲服务的可用性。

Etcd是一个强一致性的系统。它提供了线性化读取和写入，以及事务的串行化隔离。具体来说，根据PACELC定理（CAP定理的扩展），etcd是一个CP/EC系统。在正常情况下，它优先保证一致性而牺牲部分延迟；在网络分区发生时，它更倾向于保持一致性而牺牲可用性。

[pacelc](/assets/img/pacelc.jpg)
It states that in case of network partitioning (P) in a distributed computer system, one has to choose between availability (A) and consistency (C) (as per the CAP theorem), but else (E), even when the system is running normally in the absence of partitions, one has to choose between latency (L) and loss of consistency (C).

Etcd 使用 raft 算法来保证分布式系统一致性, raft 主要机制 
1. 选主, 每一个时刻只有一个主节点， 所有写请求都发给主节点，写请求被多数节点承认之后才会返回给客户端. 每次选主都会产生一个 term number 标记。被承认后的 entry 将会写到日志里面，无论后面主节点如何切换，这条记录都会出现在主节点的日志里面。并且被承认后的 entry 会被 leader 写到 CommittedIndex 里面，并且同步到集群内所有其他节点，
2. 日志复制（从主节点到从节点）. 日志里面的每一个entry 都被一个term number 和该项在日志里面的 offset 唯一标记. 一旦两个节点上的日志的某一个entry 的 term number 和 offset 一直，代表这条记录上面的所有记录都是一样的。
3. 日志执行的准确性和安全性。日志只能append。 

大概有两个条目， 一个是 Append Index, 一个是 CommitedIndex. 通过这两个指针的状态来保证一致性, appendIndex 一直在追 commitedIndex， 只有 CommitedIndex 写进本地 statemachine 的时候才是数据完整持久化的时候

etcd 使用 bolt 来持久化 statemachine 

ETCD 本质上还是一个强一致性的文件系统， 我们在 kubernetes 上看到的高可用其实指的是 ETCD 的三副本，任何一个副本挂掉了都会立即进行选主。进行重新服务。 etcd本质上还是只能有一个主进行写操作，针对写操作的行为，还不是高可用的。

# 工作

### git osxkeychain

今天被这个东西卡了一晚上，本来就是想把 https 的请求换成 ssh 的请求的. 但是不知道怎么搞得，突然就执行了一个 cache ``` git config credential.helper cache``` 然后这个一直提示我输入密码，然后就一直卡住了，一开始并不知道是什么原因，但是后来发现 ``` git config -l``` 输出的 credentail.helper 一直是 osxkeychain, 怎么删都删不掉
试了如下方法

```
git config --local --unset credential.helper
git config --global --unset credential.helper
git config --system --unset credential.helper
```
没用， 类似方法将 credentailial.helper 改为空，依旧没用。 osxkeychain 还是依旧出现在git config 里面。
甚至删除了钥匙串里面的 配置也没有用。
最后找到了这个， 总算是解脱了 https://stackoverflow.com/questions/16052602/how-to-disable-osxkeychain-as-credential-helper-in-git-config

首先通过 ```git config --show-origin --get credential.helper``` 找到这个配置的位置， 然后编辑这个文件 删除掉这个配置， 就好了。

去了这个配置之后， 直接执行
```
git config --global url."git@xxx.com:".insteadOf "https://xxxx.com/"
```
立马就好了

--show-origin 还可以 list 所有的配置的来源，这个确实不错 ```git config --get-all --show-origin credential.helper```