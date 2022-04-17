---
layout: post
title:  "weekly report"
date:   2022-04-09 22:30:08 +0800
categories: weeklyreport
---

# 论文 

论文这周没有开新的坑，怠惰了。 不过这周买了一本深入理解 linux 网络内幕英文版的， 书今天到了， 看了一下非常厚，不知道要读到什么时候，慢慢啃吧.
还有下周准备在路上啃  virtio: towards a de-facto standard for virtual I/O devices. 这篇论文，先预习一下

# 日常

### 


# 社区

-  还是上周的 pr [pr](https://github.com/kubernetes/autoscaler/pull/4798),这周有人提了几个comments，需要修补一下之前的pr。 这个pr对我的帮助主要还是帮助我了解 resource.Quantity 这个对象， reviewer review的问题主要在于对 AsScale() 的方法的使用， 这个方法可以按照指定的单位进行scale， 如果出现了round 的情形会返回 true。 这个就可以帮助我判断这里是返回正确还是错误