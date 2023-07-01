---
layout: post
title:  "weekly report"
date:   2023-06-09 22:30:08 +0800
categories: weeklyreport
---

# 读书

本周摸鱼

# 社区

本周开始做一些社区的工作了, 先从一些简单的工作做起吧

issue: https://github.com/kubernetes-csi/external-snapshotter/issues/843

这个 issue 倒是没啥, 不过会牵扯一些 ci 的东西, 还是可以学习下社区的做法的


# 工作

### https://sum.golang.org/lookup/xxxx 404 Not Found

最近在做内部项目二方包更新的时候遇到了这个错误, 在  go get xxx/xxx 的时候报错. 开始看到这个问题还是很懵的, 不过查了下, 发现是开启了 GOSUMDB, 只要设置 GOSUMDB=off 就可以解决了. 这个命令是做安全性校验的, 保证二方包没有被篡改

