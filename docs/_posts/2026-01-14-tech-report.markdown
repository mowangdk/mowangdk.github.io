---
layout: post
title:  "weekly report"
date:   2026-01-14 22:30:08 +0800
categories: weeklyreport
---


# 读书


# 工作

### modprobe

busybox 将标准的utils 进行了封装，同样， modprobe 命令也不例外， modprobe加载的模块的行为 会同标准的 modprobe 行为不一致，导致加载模块的版本不对

下面是由于 容器 没有将 host pid映射导致
```
~ # nsenter --mount=/proc/1/ns/mnt modprobe -V
modprobe: invalid option -- 'V'
BusyBox v1.37.0 (2024-09-26 21:31:42 UTC) multi-call binary.

Usage: modprobe [-rq] MODULE [SYMBOL=VALUE]...

	-r	Remove MODULE
	-q	Quiet
~ # modprobe -V
modprobe: invalid option -- 'V'
BusyBox v1.37.0 (2024-09-26 21:31:42 UTC) multi-call binary.

Usage: modprobe [-rq] MODULE [SYMBOL=VALUE]...

	-r	Remove MODULE
	-q	Quiet
```

```
diff -q <(cat /lib/modules/$(uname -r)/extra/fs/fuse/fuse.srcversion) <(cat /sys/module/fuse/srcversion) >/dev/null && echo "consistent" || echo "inconsistent"
```

# 社区