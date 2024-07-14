---
layout: post
title:  "weekly report"
date:   2024-07-06 22:30:08 +0800
categories: weeklyreport
---


# 工作

### 整理一下状态机轮转

用户删除 pod -> pod Terminating -> kubelet 调用 NodeUnpublishVolume 卸盘 -> pod 删掉 -> 触发 ownerreference 级联删除 -> kubelet 调用 NodeUnstageVolume 卸载盘 -> 触发 Node 节点上的 VolumeInUse 字段删除 -> kcm 检测到 Node 已不再使用 -> kcm 删除 VolumeAttachments -> external-attacher 调用 ControllerUnpublishVolume 卸载盘 -> external-attaher 去掉 VolumeAttachments 的 attach finalizer, 并且重置 volume 的 attach 字段为 false -> kcm 检测到 VolumeAttachments 已删除, 清理 node 上的 VolumeAttach 字段.


### Dockerfile copy --link

benifits:

- Reduces Disk Space Usage: Reduces the amount of disk space used during the build process because of hard-link
Improves Build Efficiency: COPY --link can speed up the build process since it avoids the actual copying of files, especially for large files or projects with many dependencies.

- Optimizes Layer Size: Using COPY --link helps in keeping these layers lean by not duplicating data, leading to smaller Docker images.

- Minimizes Data Redundancy: It minimizes redundancy within the Docker image, ensuring that each piece of data is stored only once, even if it is referenced in multiple stages of the build.

- Consistency Across Stages: Ensures consistency across different stages of a Docker build. All stages referencing the same file will see the same data, as the hard link points to a single source of truth.

- Environmentally Friendly: Indirectly, it contributes to environmental sustainability. By reducing the amount of disk space and processing power needed for Docker builds, it can lead to lower energy consumption in data centers.

- Simplifies Image Management: Smaller images are easier to manage, transfer, and store. This can be particularly beneficial in environments with limited bandwidth or storage resources.

- Cost-Effective for Cloud Environments: In cloud-based development environments where resources are metered and billed, optimizing for space and build time can lead to cost savings.

- Enhanced Performance in Continuous Integration (CI) Pipelines: Faster build times and smaller images can significantly improve the performance of CI/CD pipelines, leading to quicker deployment and testing cycles.

- Fallback to Regular Copy: Provides a safe fallback mechanism. If hard linking is not possible, Docker will automatically revert to a regular copy, ensuring the build process continues without interruption.


### k8s new informer cache

简单看了下 cache 这块, 之前没注意过 cr 里面的 list 里面的 key 也可以作为 index 索引,  直接遍历一遍记录下来就好. 这块需要记录下

```
l = List()
for k in templates:
    if k.metadata.name != "":
        l.append(k)
return l
```

### KUBERNETES_SERVICE_HOST, KUBERNETES_SERVICE_PORT

在做一个测试环境, 遇到了如下错误

```
kubelet  MountVolume.SetUp failed for volume "d-xxxx" : rpc error: code = Unknown desc = Get "https://22.10.0.1:443/apis/xxxxx": dial tcp 22.10.0.1:443: i/o timeout
```

调研后发现, 现在pod应该默认走了service ip, 但是因为节点上 kube-proxy 没有启动. 所以访问失败了, 只能通过. KUBERNETES_SERVICE_HOST, KUBERNETES_SERVICE_PORT 两个环境变量直接配置 SLB 的地址.绕过 kube-proxy. 配置过后即可访问.