---
layout: post
title:  "weekly report"
date:   2024-06-02 22:30:08 +0800
categories: weeklyreport
---


# 读书

### 大话存储

看了下 cifs(Common Internet File System) 的缓存, 书中主要对比了 bypass 本地缓存和不 bypass 本地缓存的网络请求, 通过 wireshark 抓包可以基本看出, bypass 本地缓存的请求都是以异步 io 的方式访问的, 而非 bypass 则是以同步 io 的形式运行. 同时也观察到了, 使用了本地缓存方式的话, 会少一些
网络请求, 并且, 由于使用了内存, 网络请求大小跟内存页进行了对齐, 基本上单个请求的大小都在 4KB/8KB 了. 并且每个请求的 offset 都是4KB 的整数, 代表整体请求跟内存是对齐的.

### prow

https://docs.prow.k8s.io/docs/components/

先放架构图

![prow](/assets/img/prow-cluster.jpg)

由图可知, prow 是由多个组件共同组成的.我们先来看下这几个核心组件的作用和简介

#### cier

> 吐槽下, 这个命名也是很贴切了, cry 的变形

主要用于向上汇报 prowjob 的状态变化, 主要看下 github 的 reporter

可以在 cier 中通过 ```--enable-workers=N``` 来使能 github reporter, 同样我们需要通过 ```--github-token-path``` 来放置 auth token. github 的接入还是比较简单的, 反而 slack 的配置比较复杂.

如果你要实现新的 cier 的话需要按照下面的 interface 进行实现

``` golang
type reportClient interface {
 Report(pj *v1.ProwJob) error
 GetName() string
 ShouldReport(pj *v1.ProwJob) bool
}
```

#### Deck

主要用来显示 prow 里面正在云盘或者最近运行的任务

deck 可以通过 ```./cmd/deck/runlocal``` 在本地运行


#### Prow-Controller-Manager

这是主要的 controller, 主要控制 prow job 执行以及生命周期的管理, 是 Plank (板子?)的替代品, 最终会取代 sinker 和 crier, 详见 [PR](https://github.com/kubernetes/test-infra/issues/17024)

Advantages
Eventbased rather than cronbased, hence reacting much faster to changes in prowjobs or pods
Per-Prowjob retrying, meaning genuinely broken prowjobs will not be retried forever and transient errors will be retried much quicker
Uses a cache for the build cluster rather than doing a LIST every 30 seconds, reducing the load on the build clusters api server


#### Tide

也是 prow 之中最重要的一个组件了, 如果有给 kubernetes sig 提过 PR 的人可能见过, 就是一般拦住你们 pr 的最后一个 action.  它主要是用来管理 符合一组条件的 github PR 池, 它会自动测试符合标准的 PR, 也会自动合并最新通过测试的 PR. Tide comes in  vs Tide goes out

##### 历史

Tide 是由 @spxtr 在 2017 年创建的，旨在替代 mungegithub 的 Submit Queue。它的设计目的是在不消耗大量 API 接口调用配额的情况下，通过使用 GitHub 的 v4 GraphQL API 实现的 GitHub 搜索查询，来管理多个组织内的大量仓库，识别可合并的 PR（拉取请求）。

文档中有一个流程图, 大概就是首先会对 PR 进行分类, 分组,分到不同的池子中, 不同类型的 PR 走不同的处理流程, 倒是没啥特殊的

tide 的基本配置位于 [tide](https://github.com/kubernetes/test-infra/blob/b3e7e2271ee1c289c159bd5c167f4386110da08b/config/prow/config.yaml#L638) 除了上面这些之外, 还可能还需要为Tide配置预提交任务，使其能够针对PR运行. 我们可以根据[这篇文档](https://docs.prow.k8s.io/docs/getting-started-deploy/)在我们自己的 org or repo 里面配置 prow.

#### prow job lifecycle

文档以用户在 github 上触发 /test all 为例, 讲述整个 prow job 的声明周期
- 用户发出 comment, github 通过 webhook 将 comment 发送给 prow
- Prow的 Kubernetes 集群使用ingress资源进行 TLS 加解密，并将流量路由到hook服务资源，最终将流量发送到定义为部署的hook应用程序. hook [应用程序](https://github.com/kubernetes-sigs/prow/blob/db89760fea406dd2813e331c3d52b53b5bcbd140/cmd/hook/main.go#L107)会以 pod 的形式部署在 pod 内部
- hook 程序接收到 /test all comments 之后, 会生成一个 [GenericCommentEvent](https://github.com/kubernetes-sigs/prow/blame/db89760fea406dd2813e331c3d52b53b5bcbd140/pkg/github/types.go#L1291) 发送给所有的 prow plugin
- prow plugin 会收到两个对象
    - a GitHub event object,
    - a [ClientAgent](https://github.com/kubernetes-sigs/prow/blob/db89760fea406dd2813e331c3d52b53b5bcbd140/pkg/plugins/plugins.go#L258) object. 翻看代码可知, 这个 agent 里面包含着各种各样的 client, 这些 client 都是 hook 在初始化的时候进行初始化的.
- Trigger 插件在运行测试之前验证 PR。例如，验证包括作者是否是组织成员，或者 PR 是否被标记为 'ok-to-test'。相关逻辑通过 [handleGenericComment](https://github.com/kubernetes-sigs/prow/blob/db89760fea406dd2813e331c3d52b53b5bcbd140/pkg/plugins/trigger/generic-comment.go#L33) 方法获取

-  handleGenericComment 在前置检查都通过之后, 开始运行 presubmit 的 job, 相关 job 通过 [getPresubmits](https://github.com/kubernetes-sigs/prow/blob/db89760fea406dd2813e331c3d52b53b5bcbd140/pkg/plugins/trigger/generic-comment.go#L57) 方法获取.
- 对于每一个 presubmits 任务我们都会通过 kubernetes api 启动一个 prowjob 来运行,相关 job 会带上 PR comments 的相关信息, prowjob 的组成就是由 presubmit 里面的 spec 和 status 字段构成的. 相关 CRD 定义如下

```
apiVersion: prow.k8s.io/v1
kind: ProwJob
metadata:
  name: 32456927-35d9-11e7-8d95-0a580a6c1504
spec:
  job: pull-test-infra-bazel
  decorate: true
  pod_spec:
    containers:
    - image: gcr.io/k8s-staging-test-infra/bazelbuild:latest-test-infra
  refs:
    base_ref: master
    base_sha: 064678510782db5b382df478bb374aaa32e577ea
    org: kubernetes
    pulls:
    - author: ixdy
      number: 2716
      sha: dc32ccc9ea3672ccc523b7cbaa8b00360b4183cd
    repo: test-infra
  type: presubmit
status:
  startTime: 2017-05-10T23:34:22.567457715Z
  state: triggered
```

- prow-controller-manager 根据 cr 的定义, 开始运行具体的任务. 等到 pod 运行结束, pcm 收集信息, 并把信息交由 crier 做上报.



# 社区

### NodeGetVolumeStats

还在继续做这块, 由于在 github 上通过自动触发的任务始终没有获取到 volumehealth 相关的 metrics, 所以只能重新再本地搭建了环境在本地跑e2e, 经过验证, 发现 kubernetes 里面通过 ```WithFeatureGate()``` 的方法没办法为 kubelet 设置. 只能通过 test-infra 中的 prow 配置来初始化环境, 趁机看了下 prow 的文档,还挺有趣. 待会统一整理一下 


### Statefulset template resize

owner 切换成了 sig-app. 但是没有人来 review 我们的 KEP. 

# 工作

工作上整体没有什么进展. 最近可能也无心工作了. 

### 压测问题

测试
sudo fio -direct=1 -iodepth=128 -rw=randwrite -ioengine=libaio -bs=4k -size=1G -numjobs=8 -runtime=1000 -group_reporting -filename=/dev/your_device -name=Rand_Write_Testing

初步测试结果: 
numjobs=4，avg rt 2ms，p95 rt 7ms
numjobs=1, avg rt 0.5ms，p95 rt 7ms.


其中 iodepth 为 io 请求队列的深度, 也就是可以同时进行未完成 io 命令的数量. 也就是 iops 会被打的很高. 在 iops 带宽打满的情况下, 进一步提高并发没有意义, 所有的请求都会堆积在队列中(队列深度很大). 大部分 io 延迟都是在队列中排队. 在 iops 被打满的情况下, 实验随着并发任务的升高而成倍增长符合预期

根据文档: https://help.aliyun.com/zh/ecs/user-guide/test-the-performance-of-block-storage-devices?spm=a2c4g.11186623.0.0.79202adcxWvHTd

测试云盘 iops 应该尽可能调大 iodepth, 设置bs 为 4k
测试云盘吞吐 iodepth 调到适中, 增大 bs 大小为 1024k
测试云盘时延, iodepth 调小, bs 调小. 