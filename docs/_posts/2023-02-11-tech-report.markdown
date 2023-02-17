---
layout: post
title:  "weekly report"
date:   2023-02-11 22:30:08 +0800
categories: weeklyreport
---

# 读书

拜提前上班所致, 地铁上人多的只想让人闭目养神, 真的没啥精力读书了. 下周一定要纠正一下

# 工作

### k8s clientgo watch

之前看这部分代码看到过 Listener.Watch 是有 list 方法和 resync 方法的. 对于我们朴素的认知来看, 既然有本地cache. 那么在初始化的时候肯定会有一次 list. 那么这个 resync 是什么逻辑呢? 原本我以为是每隔 10min 都会重新 list 一遍然后再处理一遍, 但是相关的 list 请求并没有在apiserver 的日志里面看到,所以这个 resync 可能指的是其他的事情

首先查了下, 这个 resync 是 cache.Store interface 声明的方法

然后 这个 cache 分为很多种, 目前常用的有以下几种, 这几种都实现了 cache.Store 的 interface

- expiration_cache (应该已经抛弃了)
- FIFO_cache (应该已经抛弃了) - Delta_FIFO_cache (默认使用)

目前只有 delta_FIFO 在 informer 和 ShareIndexInformer 中 被使用, deltaFIFO 的 resync 方法就是将相关cache 中的 key 重新入队一遍.然后重新广播一下. 并没有重新从 apiserver 中重新 list 数据
