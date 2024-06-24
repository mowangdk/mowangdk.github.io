---
layout: post
title:  "weekly report"
date:   2024-06-23 22:30:08 +0800
categories: weeklyreport
---


# 读书

# 社区
上上周顺利的 poc demo 完成, 其实也算是没完成吧, 因为就简单的讲了下在做什么, 实际并没有将用例实际跑一遍, 不过时间有限, 就这样了, 下周准备把之前留下的东西再做下, 到时候再看

# 工作

### sriov 解疑

本周在做 sriov pf/vf 的工作, 在测试机上遇到了两个问题. 随带请教了下相关负责的同学. 特此记录

#### bdf 端口数量问题

在测试机上存在着 500+ 个 storage bdf code. 但是实际上挂载的盘数量并没有这么多. 请教了下负责的同学知道, 这些 bdf 端口都是提前开好的. 这样的话在热插拔的时候就直接映射就好了. 不需要重新开端口. 算是提高挂载/卸载效率的一环吧.

#### 云盘 id 映射问题

看之前的云盘 id 映射逻辑有一些问题, 于是拿了相关逻辑去请教, 在 sriov main pf 下面通过```lspci <bdf> -vvvv``` 可以看到整个 bdf 的基本信息. 在基本信息里面包含存储的 sriov 里面使用的设备内存列表. 我们就是通过遍历 region 内存列表找到 云盘 id  和 bdf 的映射关系的


### check volumemounts path

最近一些客户遇到了存储块设备 hang 的问题, 其实一旦遇到了这些问题我们处理起来还是比较麻烦的, 只能提前暴露问题, 等待后端恢复.

```bash
#!/bin/bash
volume_sources=$(findmnt --task 1 -o source --noheadings --nofsroot | grep -E '^/dev/' | sort | uniq)
for source in $volume_sources; do
  if [ ! -e "$source" ]; then
    echo "$source is mounted, but does not exist"
    exit 1
  fi
done

block_targets=$(findmnt --task 1 -o target -t devtmpfs --noheadings | sort | uniq)
for target in $block_targets; do
  if [ "$(stat -c %h "/proc/1/root$target")" == "0" ]; then
    echo "$target has 0 link"
    exit 1
  fi
done
```
