---
layout: post
title:  "weekly report"
date:   2023-09-24 22:30:08 +0800
categories: weeklyreport
---


# 社区

之前的 pr 已经被合并了, 不过马上就来了一个新的

最近在搞两个 task, 一个是之前提过的 kubeadm 的 bug. 目前社区还没有决定究竟用那种方式. 所以还算在讨论中. 还有一个 csi-host-path-driver 的升级. 要适配 1.28 的测试. 

https://github.com/kubernetes/test-infra/pull/30867

https://github.com/kubernetes/kubernetes/pull/120377

# 工作

### volumeDevice blockfile

近期有人反馈重启节点之后 pod 通过 volumeDevice 声明的文件就会变成空白文件, 这个需要实际复现下. 目前看起来只在 1.18 的集群里面有问题. 

> 在 1.26 集群上试了一下, 没有发现复现, 可能是 kubelet 的版本有 bug

### nas 文件系统挂载失败问题

近期遇到了一个问题, 用户发现自己的 nas 文件系统无法挂载, 报错是 lookup xxx.com 失败, 域名无法解析, 原因经过定位是用户使用了自定义的域名, 解析该域名的 dns 被注册到了 cordns 里面, 但是 csi-plugin 并没有走容器网络, 所以在 csi-plugin 里面挂载的时候就失败了. 需要在节点上配置同样的 dns.

### vendor 过滤

本地有 vendor 文件夹, 不想被打包到容器镜像里面. 直接便捷 dockerignore  file 即可.
