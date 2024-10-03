---
layout: post
title:  "weekly report"
date:   2024-09-20 22:30:08 +0800
categories: weeklyreport
---


# 读书

### build cache

https://github.com/moby/buildkit/issues/1673

这个issue author 想问的问题也是我一直想问的， 因为没有深入的看过 buildkit 的代码， 不清楚这个 --mount=type=cache 会存储在哪里， 以及有着什么样的过期逻辑。

#### buildkit

buildkit 是第二代 build 后端， 目前已经默认使用在 Docker Desktop 里面集成，它相对前一代的 build 工具提供了很多额外的功能

- 自动跳过执行未使用的 build stage
- 并发执行独立的 build stage
- incrementally transfer only the changed files in your build context between builds 在不同的 build stages 之间只增量传输变化的上下文
- 检测并跳过在构建上下文中传输未使用的文件

Dockerfile frontend 可以动态的从容器镜像中加载，也就是我们可以编写自己的 dockerfile 命令，将代码打包到镜像里面。最后在 dockerfile里面使用我们自定义的命令， 常用方式

```
# syntax=[remote image reference]
# syntax=docker.io/docker/dockerfile:1
```

自定义Dockerfile实现可以

- 无需升级 Dockfile Daemon，就可以修复bug，使用最新版本的功能
- 同一个 Dockfile 在不同版本的 Docker 命令里面都可以有着同样的行为
- 可以使用外部实现的 Dockerfile 功能，而不用等待相关功能进入 Docker 主线


#### buildkit vs buildx vs DOCKER_BUILDKIT

Docker Buildx is a CLI plugin that extends the docker build command with the full support of features provided by BuildKit. You can see more information in Docker's official documentation Buildx is not Buildkit! Rather Buildx uses Buildkit.

Buildkit is a general artifact builder that can be used to build many other output formats (aside from just Dockerfiles). General information can be found in readme.md.

DOCKER_BUILDKIT is an environment variable that tells docker to use BuildKit when building images. In contrast, Buildx builds using the BuildKit engine and does not require DOCKER_BUILDKIT=1 environment variable to start the builds. 


#### Run Mount

RUN --mount allows you to create filesystem mounts that the build can access. This can be used to:

- 可以创建一个 bindmount 挂载点到 host filesystem 或者是 其他 build stage
- 可以访问 build secrets 或者是 ssh-agent socket
- 使用本地包管理缓存来加速构建

bind (default)	Bind-mount context directories (read-only).
cache	        Mount a temporary directory to cache directories for compilers and package managers.
tmpfs	        Mount a tmpfs in the build container.
secret	        Allow the build container to access secure files such as private keys without baking them into the image or build cache.
ssh	            Allow the build container to access SSH keys via SSH agents, with support for passphrases.

#### Run --mount=type=cache

id	      Optional ID to identify separate/different caches. Defaults to value of target.
target,   dst, destination1	Mount path.
ro,       readonly	Read-only if set.
sharing	  One of shared, private, or locked. Defaults to shared. A shared cache mount can be used concurrently by multiple writers. private creates a new mount if there are multiple writers. locked pauses the second writer until the first one releases the mount.
from	  Build stage, context, or image name to use as a base of the cache mount. Defaults to empty directory.
source	  Subpath in the from to mount. Defaults to the root of the from.
mode	  File mode for new cache directory in octal. Default 0755.
uid	      User ID for new cache directory. Default 0.
gid	      Group ID for new cache directory. Default 0.


#### mount type cache mechanisim

different ID or different mode -> different cache object - if you change the id (or the mode), you get a new empty cache mount
same ID and same mode -> by default, same cache object (including across unrelated, different Dockerfiles), EXCEPT in the following circumstances:
   - you are building with --no-cache, in which case you get a new clean mount, that will then be used for that id moving forward
   - you are using mode private and there is already another build already running using that exact mount, in which case you also get a new one (that similarly will be used by subsequent builds using that mount ID) - that last part is especially confusing
Note that the cache object is the same even if you change the mount path... only the ID+mode matters...

So, same ID and mode, the cache object should not "disappear" on its own...
... that is, unless:
a. you do a buildctl prune against your buildkitd, which destroys the cache objects
b. or you run your build with --no-cache, in which case all cache objects used by that build will be reset (including for other concurrent builds that are still in-flight)
c. or garbage collection decided to evict that cache entry
d. or you are using a "private" mount and starting your build while another build is already accessing the mount
e. OR... the apt-get configuration itself, inside your build is purposefully deleting the cache entries after a run

Furthermore, in the OP question, it looks to me like the sharing mode is shared (which is the default behavior).
For use with apt, this is probably wrong.
According to Docker documentation "apt needs exclusive access to its data" and their documentation suggests using locked instead https://github.com/moby/buildkit/blob/master/frontend/dockerfile/docs/reference.md#run---mounttypecache

Of course, if you are running a lot (a few?) concurrent builds on that buildkitd, locked will definitely slow them all down and make you loose some / most of the caching benefits in the first place...

Whether or not locked (or private) is enough in the apt world... If you have multiple different apt versions in different images using the same mount (id+sharingmode) for your apt, you might be in for trouble...

So, given OP used "shared":
f. maybe concurrent apt access to the cache mount makes apt drop all the data in some cases?

Furthermore, the OP mounts /var/cache/apt.
But if you look at cat /etc/apt/apt.conf.d/docker-clean inside the Debian official image, it is clear that this actually prevents any caching of the packages.
So, solely mounting into /var/cache/apt is useless.

Finally, mounting and reusing /var/lib/apt also seems useless to me.
The cache benefit is about 3 seconds on Debian (3.7s on empty cold start, vs. 0.7s once /var/lib/apt is populated), in the rare case where the build instruction itself is not cached.
And since you cannot trust the content of the cache folder... you cannot save yourself the apt-get update operation anyway in further instructions...

In a shell, if you want to use cache properly for apt:

use at least locked, possibly private but do NOT expect it to be actually private to this specific image you are building
use an ID that is truly unique to your Dockerfile or build process, unless you absolutely understand which other images you may be building that are using the same ID are not going to mess it up for your usage (different apt version for example)
do not cache /var/lib/apt - the unlikely cache benefit does not make much / any sense - use a tmpfs instead, if the objective is to minimize the amount of stuff that gets baked into your final image
do cache /var/cache/apt if you want (you do get the best bang for the buck here, by caching actual packages downloads), but be sure to ALSO configure apt to use it, as it is disabled in apt-conf.d in the official images (eg: that is option Dir::Cache)
Assuming you get the above right, the only remaining reasons for cache to disappear are a. b. or c from above (pretty much explicit prune or cache bust, or garbage collection).

# 工作

### pod container namespace

看起来不同 container 之间是不同的 mnt namespace， 在一个 container 内部的挂载点是无法被传递到另一个container 里面的。 需要通过 bidirectional 等 mount shared 参数区分, 如果想要在多个 contaienr 之间共享挂载点的话，可以在 emptydir/hostpath 上通过 bidirectional mount 进行共享，但是有一个问题， 必须在 container 内部或者是 prestop hook 里面将挂载点解挂载掉，但是这就意味着pod无法原地升级（比如修改image）因为原地升级的过程中不会触发initcontainer,但是会umount掉挂载点


### DeletedFinalStateUnknown

今天在集群中的组件出现了watch error 异常，排查了下根因是因为节点上的网络有异常。但是这个报错比较奇怪 DeletedFainalStateUnknown. 在 watch 的接受队列里面突然出现了一个非监听对象的错误对象，看起来这个是 kubernetes 的标准处理方案, 该错误定义在 client-go 代码中， 当一个删除事件报出的时候，如果本地因为网络原因没有捕捉到的话，也就是本地还有缓存，但是apiserver 已经没有了的话。就会抛出这个。原则上所有的watch 方法都要捕捉这个异常， 否则就会出现 crash 异常.
