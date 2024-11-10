---
layout: post
title:  "weekly report"
date:   2024-11-02 22:30:08 +0800
categories: weeklyreport
---


# 读书

# 开源

### graceful shutdown integration test

这两周写了一个这个pr， 因为这个pr 是有前驱的， 所以写起来不是很麻烦，也接着这个机会重新确认了 attachdetachcontroller 的机制， 确实 csi 对 adcontroller 来说也只不过是一种plugin。原先站在 csi maintainer 的视角来看确实有一些狭隘了。当然在实现的过程中也遇到了一些问题，还需要后面继续跟社区沟通下。

https://github.com/kubernetes/kubernetes/pull/128404


### csi aio
准备写一个 kep 了，后面主要做这个事情

# 工作

### SBOM

全程 Software Bill of Materials， 也就是 SBOM， 是一个软件清单， 里面包含软件包的依赖关系， 以及软件包的元数据， 比如版本， 作者， 许可证等。同时最重要的一点是 machine readable， 这个清单应该尽可能的全面， 如果没有办法做到的话，就需要声明说具体哪个地方不行。

SBOM 的主要目的是唯一，并且明确的识别组件们以及其相互关系。 为此，我们需要对基线组件的基本信息进行某种程度的组合，  下面列出了一些baseline, 当然这些并不是全部， 可以根据需求来添加自定义的 info

```
author name
supplier name
component name
version string
component hash
unique identifier
relationship
```

同样， 如果这个 inventory 只有人类可读的话其实意义不大， 我们必须要在软件供应链上做到机器可读，这样才能在自动化的流程中有意义。这就需要标准协议

以下三种格式 侧重于识别软件实体和传递相关元数据的核心问题--并具有满足基线 SBOM 需求的必要字段。有工具可用于生成、使用和转换这些 SBOM。

|  格式  | 规格  | 工具 | 
|  ----  | ----  | ---- |
| SPDX  | https://spdx.github.io/spdx-spec/ | https://tiny.cc/SPDX |
| CycloneDX | https://cyclonedx.org | https://tiny.cc/CycloneDX |
| SWID  |   ISO/IEC 19770-2:201 | https://tiny.cc/SWID

btw, image 的SBOM生成工具在这里 https://github.com/anchore/syft, 同时提供golang 版本的sdk， 对于云原生来说还是很方便的

那么有这个清单的好处是什么？
一个应用的完整的声明周期都将通过SBOM 获得益处， 从一开始的应用开发，供应链管理，漏洞管理，到后期的资产管理, 采购等流程。有了这个清单，比如漏洞扫描就不需要去特定位置读取制品的元信息了， 他可以在软件的描述信息中找到， 从而可以知道这个软件是否安全。


### 列出所有 ns 挂载点

command: 

```
lsns -t mnt | tail -n +2 | aws '{print $4}' |  xargs bash -c "cat /proc/{}/mountinfo | grep xxx && echo '-> found in /proc/{}/mountinfo'" 
```
