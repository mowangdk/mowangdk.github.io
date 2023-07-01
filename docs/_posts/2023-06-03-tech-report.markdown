---
layout: post
title:  "weekly report"
date:   2023-06-03 22:30:08 +0800
categories: weeklyreport
---

# 读书

### Linux命令行与shell脚本编程大全

这本书算是读完了, 整体来看看是属于工具书的级别, 最后就是全部关于 sed 和 gawk. 有点不太尽如人意. 前面还挺好. 后面好像进入了文本编辑模式.有大概 1/3 都是讲这个得

# 工作


### fuse

fuse 对单个请求(FUSE_READ/FUSE_WRITE) 可以包含的 pages 数量是有限制的, 默认为 32 个. 这个值在一开始是确定的, 不过后续的 commit 增加了配置, 允许 fuse server 和内核通过 fuse 协议来协商这个 pages 的大小. 在协商过程中，fuse server 必须在 fuse_init_out.flags 字段设置上 FUSE_MAX_PAGES 标记，通过 fuse_init_out.max_pages 协商单个 FUSE 请求可以包含的 page 的最大数量，但是最大不能超过 @max_pages_limit(256)

```
process_init_reply
    if arg->flags & FUSE_MAX_PAGES:
        fc->max_pages = min_t(fc->max_pages_limit, arg->max_pages)
```

### ownerReference

这周遇到了一个 pvc 自动被删除的问题, 在之前的印象里面, 除非 pvc 被打上了 deletionTimestamp 之外, 不应该有其他被删除的途径. 但是事发的 pvc 并没有这个 labels. 于是, 查看审计日志看看这个 pvc 究竟是被谁删了. 结果发现是 kcm 里面的 generic-garbage-collector 删的. 删除的原因是 pvc 上面有一个 ownerReference, 一旦 ownerReference 不存在, pvc 就会被 gc 程序清理



