---
layout: post
title:  "weekly report"
date:   2023-03-19 22:30:08 +0800
categories: weeklyreport
---

> 上周偷了个懒, 没写总结, 这周补上


# 读书


### 大话存储

本周继续读, 这周主要看了下外部存储阵列和 FibreChannel. 算是整理了一下磁盘阵列, 书中介绍的磁盘阵列倒是跟我认为的没太大区别. 不过的确了解到了几个新名词
- LUN: LUN 是 SCSI 协议中的名词, 是 SCSI ID 之下一级的地址号, 一个 SCSI ID(Target ID) 下面还可以有更多的 LUN ID. 对于大型阵列来说, 可以生成几百上千个虚拟磁盘. 不过后来逐渐变成所有的虚拟磁盘都变成这个名称了
- 人们一般会将 raid 放在机头阵列里面, 然后对这个机头阵列加多个扩展柜, 扩展柜里面基本上只有云盘. 而机头则包含芯片和 raid 控制器
- SAN: StorageAreaNetwork 磁盘阵列和主机之前构建出来了一个新的容器存储网络

还有一个是新认识到的就是磁盘网络构建也是使用的 OSI 模型. 是我这边之前认知狭隘了. 只不过远程存储阵列使用了不同于 TCP/IP 的新的协议FC(网络通道) 相比于 tcpip 协议组来说,它有着不同的路由规则, 网络组成架构,以及不同大小的请求头, 这些都能够给SAN 带来不少的性能提升


# 工作

### 容器内部挂载流程

runc 容器启动流程
1. kubelet 检测到 pod 被调度到宿主机上
2. kubelet 准备 cni & csi 相关"硬件"
3. kubelet 开始拉取镜像
4. kubelet 调用 cri 进行容器创建
5. containerd/docker runtime manager 收到请求开始指定调用 containerd shim
6. containerd shim 调用 runc create 命令开始创建运行时(在这里执行 overlay mount, 准备运行时使用的 rootfs)
7. runc create 开始
8. nsexec 开始, 调用 unshare 方法创建一个新的 mount namespace
9. 调用 runc init 命令初始化 runc
10. loop 所有的 device 等待 runc 的挂载点 ready
11. pivotRoot, 将上面的 overlay fs 挂载点更改为 rootfs
12. 拉起 容器主进程


### kubelet 权限问题

之前一直很好奇, 正常的 k8s 集群里面是没有 kubelet 相关的 clusterrole 的, 那么 kubelet 本身的资源访问究竟是怎么做的. 上周正好调研了一下, 发现其实 Kubelet 的权限其实是由 apiserver 直接管理的, 相关代码在 ```apiserver/node_authorizer.go``` 文件中的 Authorize 方法, 里面用代码写死了 kubelet 能够得到的权限, 并且相关的 token 是由 apiserver 直接轮转下发到节点上.并且负责之后的更新

```golang
func (r *NodeAuthorizer) Authorize(attrs authorizer.Attributes) (authorizer.Decision, string, error) {
	nodeName, isNode := r.identifier.NodeIdentity(attrs.GetUser())
	if !isNode {
		// reject requests from non-nodes
		return authorizer.DecisionNoOpinion, "", nil
	}
	if len(nodeName) == 0 {
		// reject requests from unidentifiable nodes
		klog.V(2).Infof("NODE DENY: unknown node for user %q", attrs.GetUser().GetName())
		return authorizer.DecisionNoOpinion, fmt.Sprintf("unknown node for user %q", attrs.GetUser().GetName()), nil
	}

	// subdivide access to specific resources
	if attrs.IsResourceRequest() {
		requestResource := schema.GroupResource{Group: attrs.GetAPIGroup(), Resource: attrs.GetResource()}
		switch requestResource {
		case secretResource:
			return r.authorizeReadNamespacedObject(nodeName, secretVertexType, attrs)
		case configMapResource:
			return r.authorizeReadNamespacedObject(nodeName, configMapVertexType, attrs)
		case pvcResource:
			if r.features.Enabled(features.ExpandPersistentVolumes) {
				if attrs.GetSubresource() == "status" {
					return r.authorizeStatusUpdate(nodeName, pvcVertexType, attrs)
				}
			}
			return r.authorizeGet(nodeName, pvcVertexType, attrs)
		case pvResource:
			return r.authorizeGet(nodeName, pvVertexType, attrs)
		case vaResource:
			return r.authorizeGet(nodeName, vaVertexType, attrs)
		case svcAcctResource:
			if r.features.Enabled(features.TokenRequest) {
				return r.authorizeCreateToken(nodeName, serviceAccountVertexType, attrs)
			}
			return authorizer.DecisionNoOpinion, fmt.Sprintf("disabled by feature gate %s", features.TokenRequest), nil
		case leaseResource:
			if r.features.Enabled(features.NodeLease) {
				return r.authorizeLease(nodeName, attrs)
			}
			return authorizer.DecisionNoOpinion, fmt.Sprintf("disabled by feature gate %s", features.NodeLease), nil
		case csiNodeResource:
			if r.features.Enabled(features.CSINodeInfo) {
				return r.authorizeCSINode(nodeName, attrs)
			}
			return authorizer.DecisionNoOpinion, fmt.Sprintf("disabled by feature gates %s", features.CSINodeInfo), nil
		}

	}

	// Access to other resources is not subdivided, so just evaluate against the statically defined node rules
	if rbac.RulesAllow(attrs, r.nodeRules...) {
		return authorizer.DecisionAllow, "", nil
	}
	return authorizer.DecisionNoOpinion, "", nil
}
```