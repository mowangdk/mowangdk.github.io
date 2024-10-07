---
layout: post
title:  "weekly report"
date:   2024-10-04 22:30:08 +0800
categories: weeklyreport
---


# 读书

昨晚的会上 maricio 推荐了 google 的一本书 Software Engineering at Google， 正好我们最近在做 CSI 组件的 deprecated 的工作， 正好读一下相关的章节整理一下

https://abseil.io/resources/swe-book/html/ch15.htm04l

首先第一章节介绍了为什么我们要有 deprecation 机制， 还有就是可以在立项之初就考虑到 deprecated，因为任何的软件都要有 deprecated 的时候，确实是这样，不过在实现中基本上也不太可能了。。
作者在这里关注的是废弃和移除过时系统的技术和政策方面，其中系统所有者可以观察到其使用情况。不过我觉得里面的很多规则对Openapi 也适用，我们可以不用过于纠结这点。针对于 deprecation， 作者的观点是，所有代码都是一种负债，而不是资产，如果是资产我们就没有必要去费劲心力去维护，淘汰。随着维护软件生命周期所不断增加的成本，我们需要抉择，评估是否有必要继续维护，还是要推倒重来。deprecation 与与软件存在时间没有关系，最适合那些明显过时且已有替代品提供相似功能的系统。Code 本身不带来任何价值， Code 提供的功能带来价值，简单并且简洁的代码可以提供更好的维护性，在保证相同功能的前提下，代码越简单越好

通常弃用不是一个单一的行动或者过程，而是一个连续体， 大致可以简单分为两个阶段， Advisory Deprecation & Compulsory Deprecation.

Advisory Deprecation

应该告诉用户存在新的替代方案，这个替代方案必须是 production 可用的。

Compulsory Deprecation

这种积极的鼓励表现为强制淘汰。通常，淘汰会伴随着一个移除过时系统的最后期限：如果用户在那之后仍依赖它，他们的系统将无法正常工作。（在我们的场景下则表现为相关 bug/cve 不会被修复）, 这里强制的意思是可以对不合规的用户采取措施。如果没有这种权力，客户团队很容易忽视废弃工作，转而关注新功能或其他更紧迫的任务。强制淘汰但没有人员支持的举措，可能会让客户团队觉得有些刻薄，这通常会阻碍淘汰工作的完成。客户只会视这种淘汰工作为一项无资金支持的强制任务，需要他们将自己的优先事项放在一边，只是为了维持服务的运行而工作。这感觉就像“原地跑步”现象，会在基础设施维护者和客户之间产生摩擦。因此，我们强烈建议，强制淘汰应由专门的团队全程提供支持。

Deprecation Warnings

经常会有人直接标记某个东西为过时，期待它的使用最终会消失，但要记住：“希望不是策略。”弃用警告或许能防止新的使用，但很少引导现有系统迁移。当然我们可以通过警告的方式告知用户，但是同样要警惕 warning fatigue, 也就是看了太多的deprecated 警告导致的常规性忽视

任何提给客户的 warnings 一般要有两个 properties(特性)， 可操作性和相关性（actionability and relevance）也就是只给警告推给有相关操作的用户， 确保看到警告的用户都是相关的用户。 而不是全覆盖，并且是否可以根据这个警告采取相关行动


Managing the deprecation process

Process Owners
Without explicit owners, a deprecation process is unlikely to make meaningful process.

Milestones
一个弃用过程常常让人感觉唯一的里程碑就是彻底移除过时的系统。deprecation 团队成员可能觉得在收工回家之前，他们都没有取得任何进展。尽管这对团队来说可能是最有意义的一步，但如果他们工作做得好，这往往也是团队外部人员最不容易注意到的，因为到那时，过时的系统已经没有任何用户了。弃用项目的管理者应该抵制把这作为唯一可衡量的里程碑的诱惑，尤其是考虑到在某些弃用项目中，这一步可能根本就不会发生。为了维持团队士气等因数， 我们还是要设置一些milestone 的， 尽管这个比较困难



# 社区

### ReadWriteOncePod bug

issue: https://github.com/kubernetes/kubernetes/issues/127170

貌似当 pvc 声明为 ReadWriteOncePod 的时候，pod 上的fsgroup 会不生效，声明正常，实际执行的时候报权限错误. disk 会 mount 成 root:root owner


### scheduler bug

issue： https://github.com/kubernetes/kubernetes/issues/126502

其实这个一直都有， 因为 pod 是先于 volume umount 的时候被删除的， 如果这个时候节点上的quota 已经满了，那么这个时候调度器是可能将一个pod 调度到一个目前还不可以挂载的节点上的. 但是按理说这个应该是可以自动重试掉的，所以这个 pr 是否真的有必要依旧保持怀疑。


# 工作

### mips64le, ppc64le, s390x

都是一些操作系统架构名称， le一般代表 little endian。 对我们来说都是比较小众的架构。











