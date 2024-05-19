---
layout: post
title:  "weekly report"
date:   2024-05-09 22:30:08 +0800
categories: weeklyreport
---


# 读书



# 社区

### NodeGetVolumeStats

这个东西好像比想象的麻烦

- csi.NodeServiceCapability_RPC_GET_VOLUME_STATS

首先这个配置没有生效, 于是开始找原因, 一开始以为是 mock driver 的问题, 查了半天. 最后详细看 mock 的代码, 发现是只有带有 hook 的 mock driver 才会生效这个 capability 配置. 

增加了 capability 配置之后, 发现还没有生效, 又不断地 review. 发现是有一个地方赋值赋错了, 导致配置没有生效.

修改了 bug 之后, 总算是一切都正常了, 跑 e2e 的时候又发现一个毫不相干的 e2e 报错了, loadbalancer 相关的. 问了社区的人之后, 发现是他们新增的代码有问题, 他们临时把这个测试下掉了, 但是我这边还有一些记录没办法关掉 pr 重新来, 所以目前可能比较麻烦

提给社区之后发现还有 VolumeCondition 这个东西要测试, 简单看了下代码, 意思应该是如果 csi 开启了这个 capability, 那么 kubelet 就会把相关的状态贴到pod 的 status 里面. 我这边直接校验 status 即可.


### Statefulset template resize

上周把这个活揽下来了, 第一是确实想继续推进一下这个事情, 第二是想给我们组的组员加一些社区的事情, 否则只是专注于公司的事情还是有局限的. 不过当然这个事情可能对绩效关系不大. 看看效果吧. https://github.com/huww98/enhancements/blob/sts-update-claim/keps/sig-storage/NNNN-stateful-set-update-claim-template/README.md

# 工作

### nvme reservation

上周有个客户想要实践一下, 于是就顺便看了下. 组内同学输出了一个文档. 确实这块是之前没有看到的, 感觉通过代码来实际理解的确可以比存粹看书收获更多东西. 相关文档: https://help.aliyun.com/zh/ack/ack-managed-and-ack-dedicated/user-guide/multi-attach-and-reservation-of-nvme-cloud-disks?spm=a2c4g.11186623.0.0.638d4cebtxphhg