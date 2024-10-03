---
layout: post
title:  "weekly report"
date:   2024-08-21 22:30:08 +0800
categories: weeklyreport
---

# 读书


### semantic version 2.0

https://semver.org/

详细看了下这个 2.0 的协议, 还是发现了之前没有注意过的一些点.

1. 只有存在公开 API 的应用/包才能使用semantic version.
2. 预发布版本可以通过在补丁版本之后立即添加破折号和一系列点分隔的标识符来表示。标识符必须仅包含ASCII字母数字和破折号[0-9A-Za-z-]。标识符不能为空。数字标识符不能有前导零。预发布版本的优先级低于相应的正常版本。预发布版本表示该版本不稳定，可能无法满足其关联正常版本所表示的兼容性要求。例子：1.0.0-alpha, 1.0.0-alpha.1, 1.0.0-0.3.7, 1.0.0-x.7.z.92, 1.0.0-x-y-z.--。

几个有趣的问题: 

Doesn’t this discourage rapid development and fast iteration?

Major version zero is all about rapid development. If you’re changing the API every day you should either still be in version 0.y.z or on a separate development branch working on the next major version.


If even the tiniest backward incompatible changes to the public API require a major version bump, won’t I end up at version 42.0.0 very rapidly?

This is a question of responsible development and foresight. Incompatible changes should not be introduced lightly to software that has a lot of dependent code. The cost that must be incurred to upgrade can be significant. Having to bump major versions to release incompatible changes means you’ll think through the impact of your changes, and evaluate the cost/benefit ratio involved.

What do I do if I accidentally release a backward incompatible change as a minor version?

As soon as you realize that you’ve broken the Semantic Versioning spec, fix the problem and release a new minor version that corrects the problem and restores backward compatibility. Even under this circumstance, it is unacceptable to modify versioned releases. If it’s appropriate, document the offending version and inform your users of the problem so that they are aware of the offending version.

What if I inadvertently alter the public API in a way that is not compliant with the version number change (i.e. the code incorrectly introduces a major breaking change in a patch release)?

Use your best judgment. If you have a huge audience that will be drastically impacted by changing the behavior back to what the public API intended, then it may be best to perform a major version release, even though the fix could strictly be considered a patch release. Remember, Semantic Versioning is all about conveying meaning by how the version number changes. If these changes are important to your users, use the version number to inform them.

Is “v1.2.3” a semantic version?

No, “v1.2.3” is not a semantic version. However, prefixing a semantic version with a “v” is a common way (in English) to indicate it is a version number. Abbreviating “version” as “v” is often seen with version control. Example: git tag v1.2.3 -m "Release version 1.2.3", in which case “v1.2.3” is a tag name and the semantic version is “1.2.3”.



# 工作

### fencing

"围栏"是指将节点从集群的共享存储中断开连接。它阻止了节点对共享存储的输入/输出操作，以此确保数据的完整性。这一过程由集群基础设施通过“围栏守护进程”（fenced）来执行。 主要分成两类， 一个是 storage fencing， 另一个是 power fencing， power fencing 比较简单， 直接远程关机掉fencing节点， storage fencing 则是禁掉相关的端口