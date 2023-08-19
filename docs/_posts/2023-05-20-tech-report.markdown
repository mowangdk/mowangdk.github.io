---
layout: post
title:  "weekly report"
date:   2023-05-20 22:30:08 +0800
categories: weeklyreport
---

# 读书

### Linux命令行与shell脚本编程大全

这周开始看这本书了, 忘记了之前是否有看过, 当做查缺补漏了, 

#### 内置命令
命令有内置命令和文件系统里面可执行的二进制, 可以通过 ```type cd``` 命令来观察
#### GNU(GUN's Not UNIX) 
是一个组织, 开发了一整套完整的 Unix 工具(基本上和 linux 同时进行)但是开发完了发现没有能够运行他们的内核(Unix 是闭源的) 正好这时候 Linux 出现了. 这两个形成了完美的搭配 (比如 GUN shell)
#### ps
读到这节的时候其实是解决了我多年的一个疑问的, ps 有很多常用参数, 比如 ```ps aux```, ```ps -ef```. 为什么有的带 "-", 有的不带 "-" , 有的还带 "--" , 那是因为 现代大多数操作系统里面的 ps 都是融合了以下三种类型的参数
- Unix风格的参数，前面加单破折线
- BSD风格的参数，前面不加破折线；
- GNU风格的长参数，前面加双破折线。

所以导致 ps 可能是最复杂的命令之一

#### 文件夹权限
- chown 修改文件所属用户
- chgrp 修改文件所属文件组
- chmod 修改文件的属性(具体权限) owner 的操作权限, 文件所属的用户组的操作权限, 剩余人的操作权限


# 工作

### k8s volumeDevices

volumeDevice 在宿主机上主要有四个主要挂载关系(路径) + 一个 ln -s 软链

- devtmpfs on ${kubeRootPath}/plugins/kubernetes.io/csi/volumeDevices/staging/${pvName}/${volumeName} type devtmpfs 

- devtmpfs on ${kubeRootPath}/plugins/kubernetes.io/csi/volumeDevices/publish/${pvName}/${poduid} type devtmpfs 

- devtmpfs on ${kubeRootPath}/plugins/kubernetes.io/csi/volumeDevices/${pvName}/dev/${poduid}

- /dev/loop0: [0006]:11044296 (${kubeRootPath}/plugins/kubernetes.io/csi/volumeDevices/${pvName}/dev/${poduid})

mount -o bind /dev/xxx /mnt/test 之后源路径都会变成 /dev/devtmpfs, 但是 通过 findmnt SOURCE heading 可以观察到实际关联的 /dev 设备

### nfs 

记录一下, 用户在 nas 上生成了一个保密文件, 但是这个文件无法被其他用户读取(包括 root 用户, acl 是有权限的) 会导致有问题. 这个是有些 nfs 文件系统开启的某种特性.


### github copilot

今天去听了下 ms 的分享, 简单了解了下 copilot 的使用场景, 总结了下如下

1. 正则表达式
2. Copilot 无法直接修改代码, 需要修改只能改 comment
3. 代码读取, 解释成自然语言
4. Html/css 编写
5. 编写不熟悉的算法, 不熟悉的编程语言
6. 按照常识完善 struct 定义, 实体类定义
7. 示例, 测试数据生成(测试用例生成)
8. 复杂参数的填写和上下文匹配(api params)
9. 代码文档编写
10.生成单元测试

