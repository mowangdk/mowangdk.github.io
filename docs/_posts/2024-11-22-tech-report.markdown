---
layout: post
title:  "weekly report"
date:   2024-11-22 22:30:08 +0800
categories: weeklyreport
---


# 读书

# 开源

### MGLRU

是一种内存管理算法，用于改进传统的 LRU（最近最少使用）页面替换算法。这种新的算法设计目的是为了提高内存管理的效率和性能，特别是在频繁内存访问和现代工作负载的大规模系统中。

MGLRU 的配置项 /sys/kernel/mm/lru_gen/enabled 用于启用或禁用该功能。通过修改此文件中的值，您可以控制 MGLRU 算法的开启状态：
写入 0 可以禁用 MGLRU。
写入 1 可以启用 MGLRU。

### git subtree pull

最近遇到了一个 git subtree pull 引起的问题， 在跑 CI 的过程中发现 gitsubtree 命令报错

```
remote: warning: lazy fetching disabled; some objects may not be available
remote: fatal: could not fetch b18c53581356c52a220a2baf1c8cf3fd9c57dda6 from promisor remote
error: git upload-pack: git-pack-objects died with error.
fatal: git upload-pack: aborting due to possible repository corruption on the remote side.
remote: aborting due to possible repository corruption on the remote side.�
fatal: protocol error: bad pack header
```
根据上面可以看出是 git upload-pack 报的错， 查一下相关的文档

https://man7.org/linux/man-pages/man1/git-upload-pack.1.html

又注意到我们的 lazy fetching 是 disable 的。于是我们看下对应的配置说明

```
       GIT_NO_LAZY_FETCH
           When cloning or fetching from a partial repository (i.e., one
           itself cloned with --filter), the server-side upload-pack may
           need to fetch extra objects from its upstream in order to
           complete the request. By default, upload-pack will refuse to
           perform such a lazy fetch, because git fetch may run
           arbitrary commands specified in configuration and hooks of
           the source repository (and upload-pack tries to be safe to
           run even in untrusted .git directories).

           This is implemented by having upload-pack internally set the
           GIT_NO_LAZY_FETCH variable to 1. If you want to override it
           (because you are fetching from a partial clone, and you are
           sure you trust it), you can explicitly set GIT_NO_LAZY_FETCH
           to 0.
```


```
# Prow checks out repos with --filter=blob:none. This breaks
# "git subtree pull" unless we enable fetching missing file content.
GIT_NO_LAZY_FETCH=0
export GIT_NO_LAZY_FETCH
```
最终我们用 git 的开关在 csi-release-tools 里面开启了 LAZY FETCH。 

### CSI AIO

最近开了一个issue，准备写 pr 了， FeatureManagement 部分也还有部分修改，期望可以在下次会议之前完成大部分。

# 工作

### /var/run 嵌套挂载问题

之前说过挂载容器运行时目录，如果pod 关联的hostpath. 在容器目录 /var/lib/kubelet 先于 /var/run/containerd 挂载的情况则会出现长路径的问题, 之前的问题是该长路径会导致 pod volume 卸载失败，或者是 新 pod volume 挂载异常， 不过现在居然无法复现了。先继续观察下吧