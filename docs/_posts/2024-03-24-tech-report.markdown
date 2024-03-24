---
layout: post
title:  "weekly report"
date:   2024-03-24 22:30:08 +0800
categories: weeklyreport
---


# 读书


## nvme (https://nvmexpress.org/wp-content/uploads/NVM-Express-Base-Specification-2_0-2021.06.02-Ratified-4.pdf)

继续看了几章, 感觉都是介绍相关 register 能力的. 就没有详细看,毕竟我也不是专门开发的. 再往后看看吧


# 社区

### csi sidecar AIO

这周没开会, 转移到下周了, 这两周关于 feature management 写了一些简单的思考, 会议上分享下, 看看大家的看法


# 工作

### csi-plugin terminating

这周又遇到了 csi-plugin terminating 的情况. 一开始是用户升级升级不上去, 发现端口占用, 用 ```netstat``` 看了下, 发现 csi-plugin 线程还在在用这个端口, ```ps -ef``` 找了下相关进程也都在. 但是容器已经不在了. 进程的父进程 shim 也还在, 但是根据 shim 进程 taskid 查看相关 containerd 日志(/containerdroot/containerd/io.containerd.runtime.v2.task/k8s.io/) 发现已经不存在了. 说明这个任务已经不在 containerd 的管理之下了. 最后是清理了 csi 的进程, 清理了shim 父进程, 相关端口也就释放了.


### kcm 更新 pvc pv 状态问题

在测试环境中遇到了 pvc 不存在, pv 存在的情况. 也就是用户删除了 pvc, 但是没有触发 pv 的删除. 因为是 pvc/pv 静态绑定的场景, kcm 这块实现的确实有些复杂, 要做 pvc/pv spec 和 status 的更新, 在两个 goroutine 同时更新四次状态. 确实逻辑很复杂, 除了 informer 之外,还自己维护了一套缓存. 目前可能也没有更好的解决方法了.

还有一个 kcm 的问题就是从监控发现 kcm 的 workerqueue 每 8h 都会有一个尖刺. 相关 queue 的名称是 claims & volumes. 但是我们翻了监控时间段的日志也没有发现异常. 代码内我们可以想到的 reconcile 事件间隔都非常短, 应该不会相隔 8h. 算是一个疑难问题残留吧