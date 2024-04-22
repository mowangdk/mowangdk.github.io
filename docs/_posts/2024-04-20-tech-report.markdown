---
layout: post
title:  "weekly report"
date:   2024-04-20 22:30:08 +0800
categories: weeklyreport
---


# 读书


## nvme (https://nvmexpress.org/wp-content/uploads/NVM-Express-Base-Specification-2_0-2021.06.02-Ratified-4.pdf)

首先 MetadataRegion 需要再确认一下, 它既可以作为 namespace 逻辑卷的组成元素, 又可以作为 namespace 隔离的数据缓存 需要再确认一下, 后面就简单介绍了一下 Completion Queue Entry, 没啥可讲的, 他们的返回码倒是可以参考下, 不过如果是在我们这种上层应用上面使用这种方式的返回码总感觉有点怪怪的. 不过返回值的定义倒是可以参考下

### status Field bit description

31
- Do Not Retry (DNR): If set to ‘1’, indicates that if the same command is re-submitted it is expected to fail. If cleared to ‘0’, indicates that the same command may succeed if retried. If a command is aborted due to time limited error recovery (refer to section 5.21.1.5), this field should be cleared to ‘0’. If the SCT and SC fields are cleared to 0h then this field should be cleared to ‘0’.
30
- More (M): If set to ‘1’, there is more status information for this command as part of the Error Information log that may be retrieved with the Get Log Page command. If cleared to ‘0’, there is no additional status information for this command. Refer to section 5.14.1.1.
29:28
- Reserved
27:25
- Status Code Type (SCT): Indicates the status code type of the completion queue entry. This indicates the type of status the controller is returning.
24:17
- Status Code (SC): Indicates a status code identifying any error or status information for the command indicated.


### Status Code Type

0h
- Generic Command Status: Indicates that the command specified by the Command and Submission Queue identifiers in the completion queue entry has completed. These status values are generic across all command types, and include such conditions as success, opcode not supported, and invalid field.

1h
- Command Specific Status: Indicates a status value that is specific to a particular command opcode. These values may indicate additional processing is required. Status values such as invalid firmware image or exceeded maximum number of queues is reported with this type.

2h
- Media and Data Integrity Errors: Any media specific errors that occur in the NVM or data integrity type errors shall be of this type.

3h – 6h
- Reserved

7h
- Vendor Specific

### Status Code

- 00h – 7Fh: Applicable to Admin Command Set, or across multiple command sets.
- 80h – BFh: I/O Command Set Specific status codes.
- C0h – FFh: Vendor Specific status codes.


然后就是 namespace list 和 controller list 的结构, 这里对我来说确实有点神奇, 没想到 namespaceid 和 controllerid 真的是以这种紧挨着的链表形式存在的. 当然, 这两个值的大小都是动态的. controller list 应该是取决于 pcie 的端口, namespace list 应该是取决于用户创建的 namespace.



# 社区

### csi sidecar AIO

相关文档已经提了供大家 review, 昨天晚上 review 了一遍 Migration Overview, 大体流程没有问题. 可能有一些细节需要补充下

# 工作

### aws 和 gcp 的镜像加速方案

这两天主要调研了一下 aws 和 gcp 的镜像加速方案. 简单在这里记录下

首先是 aws 的[方案:](https://aws.amazon.com/cn/blogs/containers/reduce-container-startup-time-on-amazon-eks-with-bottlerocket-data-volume/)

实际试了一下, 居然是用快照给每一个节点都创建一块盘的方案, 确实有点 low...忽略


然后是 gcp 的[方案:](https://cloud.google.com/blog/products/containers-kubernetes/accelerate-your-kubernetes-workloads-with-google-container-optimized-os)

有点像我们的 dadi 方案. 流式加载镜像存储. 


### aksk

Access Key（AK）：这是公开的身份标识符，类似于用户名。它标识了一个特定的用户或者应用程序。

Secret Key（SK）：这是对应于 Access Key 的私有密钥，必须保密，类似于密码。它用于对请求进行签名，以证明请求的发起者拥有相应的 Access Key。

当进行 API 调用时，用户需要将 Access Key 和 Secret Key 一起包含在请求中，或者在设置好凭证后，在请求中不直接携带，但需要使用 Secret Key 对请求的一个特定部分（通常是请求的 URL、方法、时间戳等）进行哈希签名。

服务端接收到请求后，会使用存储的 Access Key 查找对应的 Secret Key，然后使用同样的方法对请求进行签名。如果计算出的签名与收到的签名匹配，服务端就会认为该请求是合法的，并允许进行下一步操作。
