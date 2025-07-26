---
layout: post
title:  "weekly report"
date:   2025-07-12 22:30:08 +0800
categories: weeklyreport
---


# 读书

最近一直在出差，没有读什么有用的书，倒是随机读了几篇文章，有的感觉还不错，这里记录下

### vllm 

kvcache: 是大模型推理优化的一个常用技术，缓存每一次生成的token(会用于生成下一个token) 可以在不影响任何计算精度的前提下，提高推理性能， 提高端到端的时延。 主要包含以下两个阶段
- pre-load 阶段: 计算时需要为每一个 transformer layer 计算并保存 key cache & value cache, 在输出 token 时，cache 完成填充。该阶段的FLOPS 和没有 kvcache 的时候一致，有非常多的 gemm， 推理速度很慢，跟算力强绑定
- kv-cache 阶段：在计算第二个到最后一个token的过程中，会读取 kvcache, FLOPs 降低。推理变快，属于跟内存相绑定


page-attention: 上面 cache 存在一些问题
- Large: 对于 LLaMA-13B 参数而言， 单个序列就有1.7GB 的显存
- Dynamic: token大小长度不一致，并且不可预测，导致整体 cache 大小不好预测。

内存碎片化和过度预留会浪费60%以上的内存。为了解决这个问题，vllm 引入了 PageAttention, 与操作系统虚拟内存和分页的算法类似。pageAttention 可以将连续的 kvcache 存储在非连续的内存空间里面。具体措施就是分块，每块包含固定数量的token，pageattention kernel可以并发加速访问这些内存。可以将这些块看成页，标记看做字节，序列看做进程，序列的延长可以动态按需分配块。

由于进行了分块，上面所说的碎片化和过度预留只会发生在最后一个块里面。极大的提高了内存的使用效率。还有一个优势就是可以共享内存。在并行采样的时候，从相同的 prompt 生成的多个序列，通过其块表，PagedAttention能够自然地实现内存共享。类似于进程共享物理页，PagedAttention中的不同序列可以通过将它们的逻辑块映射到相同的物理块来共享块。为确保安全共享，PagedAttention跟踪物理块的引用计数并实现 Copy-on-Write 机制。

continuous batching

传统的批处理方法（static batching），在推理开始前固定批次大小，并且必须等到这个批次所有的任务完成之后才会继续，会浪费 GPU 的空闲时间，由于用户输入的token和输出的token无法确定，这个背景会进一步放大GPU 的等待时间，造成 LLM 推理成本进一步提高
Continuous batching 的原理即为，当批处理里面的某个任务完成之后，会将下一个批处理认为加到批处理里面，从而让GPU不会等待任何任务完成



# 社区

### nodeAffinity
这周周会上又提了一嘴，后面说是线下聊了


### CSI AIO
说是可以对外了，回头再整理下FAQ 和新增的需求

# 工作
反而没什么更新，一直在开会

### 尝试将pod内的挂载点映射到host

发现映射到 host 上的挂载点无法回收，阻塞了 kubelet globalmount 的回收，这点有点麻烦

```
          volumeMounts:
          - mountPath: /mnt
            mountPropagation: Bidirectional
            name: host-mnt
          - mountPath: /mnt/bmcpfs01
            name: data-bmcpfs01
          - mountPath: /mnt/bmcpfs02
            name: data-bmcpfs02
          - mountPath: /mnt/bmcpfs03
            name: data-bmcpfs03
        volumes:
        - hostPath:
            path: /mnt
            type: Directory
          name: host-mnt
        - name: data-bmcpfs01
          persistentVolumeClaim:
            claimName: bmcpfs01
        - name: data-bmcpfs02
          persistentVolumeClaim:
            claimName: bmcpfs02
        - name: data-bmcpfs03
          persistentVolumeClaim:
            claimName: bmcpfs03
```