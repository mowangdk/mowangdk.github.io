---
layout: post
title:  "weekly report"
date:   2025-10-01 22:30:08 +0800
categories: weeklyreport
---


# 读书

# 社区

### nodeaffinity


我们在修改 pv affinity 的时候注意到了，kubelet 在调用 NodePublishVolume 接口之前会调用 https://github.com/kubernetes/kubernetes/blob/5161bf00587eb36cd3ce64635126f47a98068b5c/pkg/volume/util/operationexecutor/operation_generator.go#L455 CheckNodeAffinity 接口进行判断， 如果我们在 pod running 的时候改了 nodeAffinity， 可能会影响相关逻辑判断, 导致同一个volume 的其他 operation 阻塞

```golang
func checkNodeAffinity(og *operationGenerator, volumeToMount VolumeToMount) error {
	pv := volumeToMount.VolumeSpec.PersistentVolume
	if pv != nil {
		nodeLabels, err := og.volumePluginMgr.Host.GetNodeLabels()
		if err != nil {
			return err
		}
		err = storagehelpers.CheckNodeAffinity(pv, nodeLabels)
		if err != nil {
			return err
		}
	}
	return nil
}

func CheckNodeAffinity(pv *v1.PersistentVolume, nodeLabels map[string]string) error {
	if pv.Spec.NodeAffinity == nil {
		return nil
	}

	if pv.Spec.NodeAffinity.Required != nil {
		node := &v1.Node{ObjectMeta: metav1.ObjectMeta{Labels: nodeLabels}}
		terms := pv.Spec.NodeAffinity.Required
		if matches, err := corev1.MatchNodeSelectorTerms(node, terms); err != nil {
			return err
		} else if !matches {
			return fmt.Errorf("no matching NodeSelectorTerms")
		}
	}

	return nil
}
```


### CSI AIO

#### csi-lib-utils
https://github.com/kubernetes-csi/csi-lib-utils/pull/202

slack： https://kubernetes.slack.com/archives/C8EJ01Z46/p1759222955408599

突然有个人想要合并所有 sidecar 中的 lease ， 大家基本同意将一些共性的 common code 放到 csi-lib-utils 里面。 不过我对于是否要将所有的controller 放到一起表示有一些疑问， 已经在 slack 上回复了一下， 后面看结果吧

### Prefill Decode 分离


预填充（Prefill）阶段：此阶段负责一次性处理用户输入的全部 Prompt，计算并生成初始的键值缓存（KV Cache）以及第一个输出 token。Prefill 阶段是计算密集型（Compute-intensive）任务，能充分利用 GPU 的并行计算能力，其性能通常用“首词元延迟”（TTFT）来衡量。

解码（Decode）阶段：在 Prefill 之后，模型进入自回归的迭代过程，逐一生成后续的 token 此阶段是内存密集型（Memory-intensive）任务，每次迭代计算量较小，但需要频繁访问和更新巨大的 KV Cache，因此内存带宽成为主要瓶颈. 其性能由“每词元生成时间”（TPOT）衡量.

通常一次推理只有一次预填充，多次解码

原本以为如果只是简单的推理，其实没有必要分离，但是调查了下发现现在不仅 agent 会对单轮对话进行多次推理，并且还有对复杂对话的自拆分，叠加多用户同时请求，如果不做分离确实可能没办法。

https://github.com/kvcache-ai/Mooncake?tab=readme-ov-file

简单看了下 readme 这部分的描述，感觉的确 kv-cache 重点不在 cache， 而是在调度，其实cache本身有多重方案可以替代，跟非 AI 时代并没有什么区别




# 工作

### multi_tenant plan

https://www.vcluster.com/blog/comparing-multi-tenancy-options-in-kubernetes

文中对比了三种多租场景的方案， Hierarchical Namespace Controller(HNC), Vcluster and Karmada

> kubernetes 的 namespace 方案不在多租方案的范围之内，因为它是为了在 Kubernetes 中的 a fundamental building block for grouping 来做的

#### Hierarchical Namespace Controller

这个方案的解决方法是嵌套 namespace， 例如如果在 parents namespace 里面创建一个role，就会自动在 child namespace 里面创建。不过这个方案并未解决全局资源的问题
类似像是 PersistentVolume 类型的资源还是没有办法， 初次之外，这个方案没有办法处理资源 overwrite 的情况。

#### vCluster a control plane per tenant

简单来说就是管控面变成pod,针对每个用户单独进行部署

- Each tenant has an entire control plane and the flexibility of a real Kubernetes cluster.
- This control plane is only used to store resources in a database.
- The controller can be instructed to copy only specific resources.

但是每一个tenant 集群没有 node 节点概念, 类似 serverless 方案

#### karmada

存粹就是多集群管理方案了， 每个tenant都有着不同的控制面，不同的节点


以上三种方案相关的成本也越来越高，隔离性，成本，以及管理成本都是需要在管理层面上决策的