---
layout: post
title:  "weekly report"
date:   2023-02-25 22:30:08 +0800
categories: weeklyreport
---

# 读书

本周没有读书, 主要看了一篇论文 + 研究一下 NVMe over Fabric

### Extension Framework for File Systems in User space
还在阅读当中, 看简介部分和前半部分主要是介绍了下目前 fuse 系统为了兼容一般环境做了很多通用性配置, 但是如果在某些特殊场景下这些配置是可以省略或者是不做地. 那么这样就带来了一些性能提升. (怎么感觉最近读的几篇论文都是这样...) 后面再继续看下


# 工作


### NVMe over Fabric 

NVMe over Fabric是NVMe的互联存储协议，是一种全新的基于NVMe协议的远程访问存储协议，由NVMe Express组织制定，其支持互联的底层网络协议包括：RDMA，TCP，FC等。以RDMA技术为例，4k报文的传输延时在10us左右，而NVMe SSD本地的4k读报文延时是80us左右，RDMA网络传输延时占比非常小。因此NVMe over RDMA的技术其性能非常接近本地盘的性能，而目前RDMA技术已经是非常成熟的技术， 相比 NFS 来说, NVME over fabric 是提供远程块的使用, NFS 是提供远程文件的使用
