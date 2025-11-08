---
layout: post
title:  "weekly report"
date:   2025-10-25 22:30:08 +0800
categories: weeklyreport
---


# 读书

### e2b 存储相关
https://e2b.dev/docs/sandbox/persistence

正好最近都在搞这个，随带也看了下， 周五同事还分享了下使用方式， 这个东西确实更像是一个 tools kit， 封装了各种agent场景的常见行为， 用各种算力实现背后的逻辑，当然这种算力不一定要是kubernetes，甚至可以直接对接ecs， 那么这个事情可能说白了跟 kubevirt 和 kata 可能的确也没那么大关系。 但是还是老想法， 如果终端用户用的不希望依赖平台，而是自己构建， 那么这种场景其实 Kubernetes 确实是一个比较好的选择

大部分的用法可能主要集中于定义template， 不过不知道大多数用户怎么想，我是觉得他们目前提供的接口有点过于简单了， 当然，可能还是需求不强烈的情况，不知道后面是否会慢慢扩展出一些新东西. e2b 的主要差异还是在template 这里

>  For each layer command (.copy(), .runCmd(), .setEnvs(), etc.), we create a new layer on top of the existing.
```python
Template.build(
    template,
    alias="my-template",
    skip_cache=True,  # Configure cache skip (except for files)
)
```

关于 template 的还有一个比较有趣的是 cache，是借鉴了 docker layer 的 cache. 看起来的确跟 commit 比较像， 这里的 cache 其实更像是是否在远端复用 layer， 并且对于文件确实有额外的优化，必须在copy中强制声明upload 才会上传，否则只会复用远端的文件
```python
    .copy("config.json", "/app/config.json", force_upload=True)
```

这个 Template 就是容器镜像， 这个地方可能跟 kubernetes 是割裂的, 更像直接对接的docker 的生态 可能也是正确的，毕竟整体上，镜像， 它这里并没有容器组的概念(目前看起来) 如果后续再添加新的容器组的概念的话，没准就可以直接对接kubernetes 了

```python
dockerfile_content = """
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y curl
WORKDIR /app
COPY . .
ENV NODE_ENV=production
ENV PORT=3000
USER appuser
"""

template = (
    Template()
    .from_dockerfile(dockerfile_content)
    .set_start_cmd("npm start", wait_for_timeout(5_000))
)

# From object
Template().from_gcp_registry(
    image="ubuntu:22.04",
    service_account_json={"project_id": "123", "private_key_id": "456"},
)
```
这里相当于直接构建镜像，完成之后推送到 ACR 或者 GCR

```python
Template.build(
    template,
    alias="my-template",  # Template alias (required)
    cpu_count=2,  # CPU cores
    memory_mb=2048,  # Memory in MB
    skip_cache=False,  # Configure cache skip (except for files)
    on_build_logs=default_build_logger(),  # Log callback receives LogEntry objects
    api_key="your-api-key",  # Override API key
    domain="your-domain",  # Override domain
)
```
在build 的时候， 可以指定cpu， memory， 缓存， 构建日志， api key， domain 好像把运行时的信息跟build镜像的信息耦合到一起了， emmm

```python
# template.py
from e2b import Template

template = (
    Template()
    .from_image("image-tag")
)
```
fromimage 可以直接用现成的容器镜像进行构建， 不过看起来也需要进行预热之类的动作

结果翻文档翻了一圈没找到 volume 这个东西, 说他 template 是一个运行时 + 镜像的组合，但是却没有提供外置volume 的接口？太奇怪了， 问了下gpt, 给我瞎编了一个 persistent_storage_path 还挺像那么回事的. 关联的官方文档已经不存在了


agent team： 指的是多个agent，并不是一个agent 多个llm

# 社区

### WaitFor CSINode removal to register CSI plugin
https://github.com/kubernetes/kubernetes/pull/131098

### csi aio
kep已经通过， 下面要实际开始开工了， 

- 在 kep 合并的情况下， 我们可以开始对 individual 进行改造了
    - revisited csi sidecars parameters
    - 我们之前定义的 common args 已经被放到了 csi-lib-utils 里面了，剩下的需要对 individual 改造, 使其可以使用common args
    - sidecar error handling 处理， 需要参考 kcm-controller[TODO]
       - kcm 在 controller 初始化的步骤是会记录相关错误信息
       - kcm 会挂历所有的controller 的状态，healthcheck 
       - 协程内部不会做任何报错，会记录信息
    - 调整 main container 的架构，使其尽可能避免出现冲突
- csi-sidecars  本身改造
    - 将 snapshotter 和包含 goclient & CRD 的其他 controller 集成进 csi-sidecars
    - 集成 node-driver-registrar（不属于controller）
    - [option] use go problem 进行增量合并, 也许我们可以先保持全量同步尝试一段时间，没有 get 到切换成 go program 的优势
       - 我理解全量同步可以使得历史记录更干净？
- 在 csi-sidecars 里面构建 CICD 流程
    - CSI hostpath code modified so that it uses the CSI monorepo
    - 使用 csi-release-tools 构建
    - 变更 test-infra 跑通所有 e2e 测试
   


