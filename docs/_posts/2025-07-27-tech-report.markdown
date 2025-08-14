---
layout: post
title:  "weekly report"
date:   2025-07-27 22:30:08 +0800
categories: weeklyreport
---


# 读书

### ACP 

学习下大模型课程，几个点记录一下, 这个课程主要关注的是推理流程。

- 分词(Tokenization)：简单来说就是把用户输入进行分类归纳，这里并不是单纯的每一个单词或者是每一个汉字作为一个词，比如选择，并不会选一个token, 择一个token. 而是选择一起作为一个token，这个分词的规则是根据上下文文字同时出现的频率来决定的。 这个流程是需要用预训练的模型自动来完成的.

- Token 向量化(Embedding): 将上面 token 的结果向量化存储到向量数据库里面. 同样，这里 embedding 的模型也是事先创建的

- 模型推理： 将上述向量化的 token 传递给大模型的时候, 大模型自动计算下一个 token 可能出现的概率集合。 目前普遍是通过 temperature(0-2,可以理解为熵增，温度越高，token分布概率越平均，回答多样性越大) 和 top_p(top_p （小于1）是根据传入的值来控制概率的 ，比如设置了0.1 大概率只会返回第一个token, 如果返回了 0.7 会输出前几个概率加起来小于0.7的结果) 来调整的。这里需要注意，生成第二个token是需要以用户输入以及第一个推理出来的 token 作为参数输入的，所以这里是存在自回归的流程的

- RAG（Retrieval Augmented Generation）检索增强式生成, 用于回答私域问题, 流程并不是一股脑的将私域信息都塞给大模型， 而是先检索出有关问题的片段，再把片段塞给大模型进行生成最终的答案。大概两个阶段
   - 建立索引: 文档加载 -> 内容分割 -> 向量化 -> 存储 
   - 检索生成: 用户提问 -> 内容检索 -> 提示词 -> 大模型生成答案
      - 在检索到相关的文本段后，RAG应用会将问题与文本段通过提示词模板生成最终的提示词，由大模型生成回复，这个阶段更多是利用大模型的总结能力，而不是大模型本身具有的知识。这个提示词模板的设计，是上下文工程的另一个关键环节。我们不仅要提供检索到的“资料”，还要明确地“指导”模型如何使用这些资料来回答问题。一个典型的提示词模板为：请根据以下信息回答用户的问题：{召回文本段}。用户的问题是：{question}。

  RAG 其实是上下文工程(context engineering)中的一个技术
- 
经历了些波折，算是过了吧

# 社区
### volume attach fast failed
https://github.com/kubernetes/kubernetes/pull/132933/files
https://github.com/torredil/website/blob/552c5311d134da90f3a45cc54db5216361b00a49/content/en/blog/_posts/2025-07-07-mutable-csi-node-allocatable.md#immediate-updates-on-attachment-failures

When a volume attachment operation fails due to a ResourceExhausted error (gRPC code 8), Kubernetes immediately updates the allocatable count instead of waiting for the next periodic update. The Kubelet then marks the affected pods as Failed, enabling their controllers to recreate them. This prevents pods from getting permanently stuck in the ContainerCreating state.

the pod cleanup process is independent of the feature gate and its part of the Kubelet state machine. The feature gate simply controls whether we check for and react to ResourceExhausted errors (by transitioning pods to a terminal, Failed state in SyncPod). Note that VerifyExhaustedResource will only return true iff both the CSI plugin has opted in to this feature && attachment.Status.AttachError.ErrorCode == codes.ResourceExhausted (which the external-attacher is responsible for patching in the VA, and it only does so if the feature gate is enabled).Cleanup / teardown is performed in SyncTerminatedPod (the reverse of SyncPod).

变更是在 WaitForAttachAndMount 方法里面实际判断的，开启了 featuegate 才会在 WaitForAttachAndMount 方法外侧返回

### DisruptionTarget

其实某种情况下跟上面相关, 如果给pod 打上了这个DisruptionTarget 可以触发kubelet 自动回收资源，并且标记 pod 为 Failed

DisruptionTarget: the pod is about to be terminated due to a disruption (such as preemption, eviction or garbage-collection).

btw， 这次确认了下， 不能把 pod 直接标记为 Failed， 因为节点上的流程需要有机制回收，如果 controller 直接标记会跳过 kubelet 回收流程，上述代码里面通过直接更新 node 的 condition 中的代码是因为如下逻辑会自动触发pod 删除流程
```golang
					// Return error to the kubelet, which will then trigger the pod termination logic.
					return &VolumeAttachLimitExceededError{
						UnmountedVolumes:  unmountedVolumes,
						UnattachedVolumes: unattachedVolumes,
						VolumesNotInDSW:   volumesNotInDSW,
						OriginalError:     err,
					}
``` 
并且由于 设置 pod.Status.Phase【通过ResourceExhausted】 & 数据清理【通过featuregate MutableCSINodeAllocatableCount】始终是在 kubelet 侧， 所以也不存在集群升级过程中的

### bugfix review
https://github.com/kubernetes-sigs/sig-storage-lib-external-provisioner/pull/190

# 工作

### exec /usr/bin/plugin.csi.alibabacloud.com: exec format error

镜像拉取失败，今天花了一个晚上算是确定了基本原因，是因为用户在拉取镜像的时候重启了节点（并且是强制重启）这种重启节点会中断镜像拉取，等到重新启动之后，containerd会认为镜像已经拉取完成。从而使用一个不完整的镜像，导致 exec format error

