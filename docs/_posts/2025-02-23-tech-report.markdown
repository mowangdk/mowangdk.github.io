---
layout: post
title:  "weekly report"
date:   2025-02-23 22:30:08 +0800
categories: weeklyreport
---


# 读书

最近大模型又火了一波， 对就是deepseek，正好借这个机会熟悉下

### DeepSeek V3 技术创新

多头潜在注意力（Multi-head Latent Attention, MLA）， 它通过引入潜在空间来优化传统的多头注意力机制。在传统的多头注意力中， 模型会直接对输入数据进行多次不同角度的注意力计算。以捕捉数据中的复杂关系。 然而，这种方法在处理大规模数据的时候会导致计算效率低下。多头潜在注意力通过将输入数据映射到一个低纬的潜在空间中，然后在这个潜在空间中进行注意力计算，从而减少了计算量。 这种设计不仅提高了模型的推理效率，还保持了对输入数据的复杂关系的捕捉能力。通过这种方式， DeepSeek V3 能够在大规模数据上进行高效的训练和推理， 还能够保持对输入数据的复杂关系的捕捉能力。

> 混合专家模型 (MoEs):
与稠密模型相比， 预训练速度更快
与具有相同参数数量的模型相比，具有更快的 推理速度
需要 大量显存，因为所有专家系统都需要加载到内存中
在 微调方面存在诸多挑战，但 近期的研究 表明，对混合专家模型进行 指令调优具有很大的潜力。稀疏 MoE 层: 这些层代替了传统 Transformer 模型中的前馈网络 (FFN) 层。MoE 层包含若干“专家”(例如 8 个)，每个专家本身是一个独立的神经网络。在实际应用中，这些专家通常是前馈网络 (FFN)，但它们也可以是更复杂的网络结构，甚至可以是 MoE 层本身，从而形成层级式的 MoE 结构。门控网络或路由: 这个部分用于决定哪些令牌 (token) 被发送到哪个专家。例如，在下图中，“More”这个令牌可能被发送到第二个专家，而“Parameters”这个令牌被发送到第一个专家。有时，一个令牌甚至可以被发送到多个专家。令牌的路由方式是 MoE 使用中的一个关键点，因为路由器由学习的参数组成，并且与网络的其他部分一同进行预训练。

DeepSeek MOE(Mixture of Experts), DeepSeekMoE 是 DeepSeek V3 的另一个核心架构， 用于实现成本效益的训练。 MoE 架构的核心思想是将模型拆分为多个专家网络，每个专家网络负责处理数据的特定部分， DeepSeekMoE 架构核心是采用了更细粒度的专家分配策略， 每个 MoE 层有一个共享专家和 256 个路由专家（共有 58 个 MoE 层 14906 个路由专家）每个token可以激活8个路由专家


专家构成: 标准 MoE 架构的类型和分工比较宽泛， 一般不会强调共享专家的设置， 各个专家相对独立的处理不同的数据， 在处理不同类型数据特征时专家之间缺少专门的设计和写作的共享机制。而 DeepSeekMoE 架构中的共享专家可以对不同输入数据中的共性数据进行处理， 在不同类型的数据输入之间实现共性特征和知识共享， 以便减少模型参数冗余， 而路由专家则负责处理具有特定模式或者特征的数据。 提高模型对不同数据的适应性和处理能力。

专家分配: 简单来说就是对数据的解析更加精确，可以选择更加适合的专家进行处理， 提升模型对复杂数据的处理能力

专家激活: 标准 MoE 架构对于数据激活的专家没有固定标准， 在某些情况下 输入数据可能激活过多非必要的专家。 DeepSeekMoE 则明确了每个token激活8个路由专家. 在具体实现的方式上， 共享专家会对每个输入Token进行处理， 以便提供通用的基本特征， 路由专家则会根据输入 Token 的特征而决定是否被激活并参与计算

### DeepSeek R1 技术创新


# 工作

### PVC is being deleted

kubelet 在初始化 desired state of world 的时候会去查找 pod 引用的 pvc 的状态， 如果 pvc 状态异常（terminating), 则 kubelet 不会把 volume 加到 dsow 里面。 后面reconcile 的时候会自动调用csi接口弹盘， 不过社区本身不支持（建议）客户原地升级， 这个事情就比较复杂了


### 挂载点超限

```
for p in $(lns -t mnt |grep -v PID|awk '{print $4}'); do cat /proc/$p/mountinfo|wc ; done
```

可以通过 sysctl -w fs.mount-max=200000 来修改


### subpath 清理

subpath 声明

```
    - mountPath: /usr/share/redis
      name: persistent-data
      subPath: app-data

```
mount | grep csi output
```
/dev/vdc on /var/lib/kubelet/plugins/kubernetes.io/csi/diskplugin.csi.alibabacloud.com/b8b51917291e7a4dc6b2fc601622fffb0e21d09e71b23bd7115be136edf55fee/globalmount type ext4 (rw,relatime)
/dev/vdc on /var/lib/kubelet/pods/{pod-uid}/volumes/kubernetes.io~csi/{disk-id}/mount type ext4 (rw,relatime)
/dev/vdc on /var/lib/kubelet/pods/{pod-uid}/volume-subpaths/{disk-id}/csi-test/0 type ext4 (rw,relatime)
```

pid-1 mountinfo
```
614 100 253:32 / /var/lib/kubelet/plugins/kubernetes.io/csi/diskplugin.csi.alibabacloud.com/b8b51917291e7a4dc6b2fc601622fffb0e21d09e71b23bd7115be136edf55fee/globalmount rw,relatime shared:342 - ext4 /dev/vdc rw
1255 100 253:32 / /var/lib/kubelet/pods/{pod-uid}/volumes/kubernetes.io~csi/{disk-id}/mount rw,relatime shared:342 - ext4 /dev/vdc rw
1517 100 253:32 /app-data /var/lib/kubelet/pods/{pod-uid}/volume-subpaths/{disk-id}/csi-test/0 rw,relatime shared:342 - ext4 /dev/vdc rw
```

pid-container mountinfo
```
sh-4.4# cat /proc/xxxx/mountinfo  | grep app
2117 2041 253:32 /app-data /usr/share/redis rw,relatime - ext4 /dev/vdc rw
```

kubelet logs 
```
Feb 20 19:08:01 kubelet[2166]: I0220 19:08:01.078630    2166 subpath_linux.go:236] Bound SubPath
/var/lib/kubelet/pods/{pod-uid}/volumes/kubernetes.io~csi/{disk-id}/mount/app-dat
a into /var/lib/kubelet/pods/{pod-uid}/volume-subpaths/{disk-id}/csi-test/0
```

subpath 的基本流程, 

```golang
    fd, err := safeOpenSubPath(mounter, subpath)
	// Do the bind mount
    mountSource := fmt.Sprintf("/proc/%d/fd/%v", kubeletPid, fd)
	options := []string{"bind"}
	mountFlags := []string{"--no-canonicalize"}
	klog.V(5).Infof("bind mounting %q at %q", mountSource, bindPathTarget)
	if err = mounter.MountSensitiveWithoutSystemdWithMountFlags(mountSource, bindPathTarget, "" /*fstype*/, options, nil /* sensitiveOptions */, mountFlags); err != nil {
		return "", fmt.Errorf("error mounting %s: %s", subpath.Path, err)
	}
	success = true
    klog.V(3).Infof("Bound SubPath %s into %s", subpath.Path, bindPathTarget)

```

整体来看下，还是很优雅的，在 publishVolume的目录下面创建子目录， 打开这个目录， 并且持有这个目录的fd (防止被卸载), 然后将fd 挂载到目标子目录，所以如果当我尝试卸载 or 删除 publish 目录的时候
```umount /var/lib/kubelet/pods/{pod-uid}/volumes/kubernetes.io~csi/{disk-id}/mount``` 可以正常卸载, 卸载 globalmount 的时候， 也正常, pod 内部依旧可写。 globalmount 和 mount 在 subpath 场景下依旧不会影响容器内的读写

#### test1
如果我们删除了子目录， 如何？

```
sh-4.4# mount /dev/vdc /data/test
sh-4.4# ls /data/test
aaa  app-data  bbb  lost+found
sh-4.4# ls /data/test
aaa  app-data  bbb  lost+found
sh-4.4# rm /data/test/app-data
rm: cannot remove '/data/test/app-data': Is a directory
sh-4.4# rm -rf /data/test/app-data
sh-4.4# ls /data/test
aaa  bbb  lost+found
```
同样可以删除。容器里面继续写会怎么样？新建的文件不存在了， 但是没有ioerror报错

```
sh-4.4# ls /usr/share/redis
aaa
sh-4.4# ls /usr/share/redis
sh-4.4#
```

#### test2
我们新建一个实例，并且尝试删除 fd 引用的路径，顺利删除了， fd起码没有阻止我们删除的能力

```
sh-4.4# ls /var/lib/kubelet/pods/{pod-uid}/volumes/kubernetes.io~csi/{disk-id}/mount/
aaa  bbb  lost+found
sh-4.4# ls /var/lib/kubelet/pods/{pod-uid}/volume-subpaths/{disk-id}/csi-test/0
sh-4.4# ls -a /var/lib/kubelet/pods/{pod-uid}/volume-subpaths/{disk-id}/csi-test/0
sh-4.4# mount | grep /var/lib/kubelet/pods/{pod-uid}/volume-subpaths/{disk-id}/csi-test/0
/dev/vdd on /var/lib/kubelet/pods/{pod-uid}/volume-subpaths/{disk-id}/csi-test/0 type ext4 (rw,relatime)
sh-4.4# touch /var/lib/kubelet/pods/{pod-uid}/volume-subpaths/{disk-id}/csi-test/0/test-new
touch: cannot touch '/var/lib/kubelet/pods/{pod-uid}/volume-subpaths/{disk-id}/csi-test/0/test-new': No such file or directory
```
节点上创建失败

```
sh-4.4# cd /var/lib/kubelet/pods/{pod-uid}/volume-subpaths/{disk-id}/csi-test/0
sh-4.4# ls
sh-4.4# touch test-new
touch: cannot touch 'test-new': No such file or directory
```

容器内找不到文件，写入失败
```
sh-4.4# mount | grep redis
/dev/vde on /usr/share/redis type ext4 (rw,relatime)
sh-4.4# ls /usr/share/redis
sh-4.4# touch /usr/share/redis/test-data5
touch: cannot touch '/usr/share/redis/test-data5': No such file or directory
```

containerd-id = {container-id}



containerd 正常回收相关数据
```shell
Feb 21 17:44:37  containerd[1919]: time="2025-02-21T17:44:37.422323762+08:00" level=info msg="CreateContainer within sandbox \"{container-id}\" for &ContainerMetadata{Name:csi-test,Attempt:0,} returns container id \"{container-id}\""
Feb 21 17:44:37  containerd[1919]: time="2025-02-21T17:44:37.423026795+08:00" level=info msg="StartContainer for \"{container-id}\""
Feb 21 17:44:37  containerd[1919]: time="2025-02-21T17:44:37.493897929+08:00" level=info msg="StartContainer for \"{container-id}\" returns successfully"
Feb 21 17:58:10  containerd[1919]: time="2025-02-21T17:58:10.444822865+08:00" level=info msg="StopContainer for \"{container-id}\" with timeout 30 (s)"
Feb 21 17:58:10  containerd[1919]: time="2025-02-21T17:58:10.445266387+08:00" level=info msg="Stop container \"{container-id}\" with signal terminated"
Feb 21 17:58:40  containerd[1919]: time="2025-02-21T17:58:40.453045010+08:00" level=info msg="Kill container \"{container-id}\""
Feb 21 17:58:40  containerd[1919]: time="2025-02-21T17:58:40.515145516+08:00" level=info msg="shim disconnected" id={container-id}
Feb 21 17:58:40  containerd[1919]: time="2025-02-21T17:58:40.515191857+08:00" level=warning msg="cleaning up after shim disconnected" id={container-id} namespace=k8s.io
Feb 21 17:58:40  containerd[1919]: time="2025-02-21T17:58:40.527059565+08:00" level=info msg="StopContainer for \"{container-id}\" returns successfully"
Feb 21 17:58:40  containerd[1919]: time="2025-02-21T17:58:40.527608205+08:00" level=info msg="Container to stop \"{container-id}\" must be in running or unknown state, current state \"CONTAINER_EXITED\""
Feb 21 17:58:40  containerd[1919]: time="2025-02-21T17:58:40.764421009+08:00" level=info msg="RemoveContainer for \"{container-id}\""
Feb 21 17:58:40  containerd[1919]: time="2025-02-21T17:58:40.768944768+08:00" level=info msg="RemoveContainer for \"{container-id}\" returns successfully"
Feb 21 17:58:40  containerd[1919]: time="2025-02-21T17:58:40.769315916+08:00" level=error msg="ContainerStatus for \"{container-id}\" failed" error="rpc error: code = NotFound desc = an error occurred when try to find container \"{container-id}\": not found"
```
最后的 notfound 是相关container 被回收掉了。


kubelet 报错

```
Feb 21 17:59:45  kubelet[1204031]: I0221 17:59:45.963926 1204031 kubelet.go:2209] "Pod is terminated, but some volumes have not been cleaned up" pod="default/csi-test-0" podUID="{pod-uid}"
Feb 21 17:59:45  kubelet[1204031]: I0221 17:59:45.963954 1204031 reconciler_common.go:152] "Starting operationExecutor.UnmountVolume for volume \"persistent-data\" (UniqueName: \"kubernetes.io/csi/diskplugin.csi.alibabacloud.com^{disk-id}\") pod \"{pod-uid}\" (UID: \"{pod-uid}\") "
Feb 21 17:59:45  kubelet[1204031]: I0221 17:59:45.963990 1204031 csi_plugin.go:435] kubernetes.io/csi: setting up unmounter for [name={disk-id}, podUID={pod-uid}]
Feb 21 17:59:45  kubelet[1204031]: I0221 17:59:45.964006 1204031 csi_mounter.go:87] kubernetes.io/csi: mounter.GetPath generated [/var/lib/kubelet/pods/{pod-uid}/volumes/kubernetes.io~csi/{disk-id}/mount]
Feb 21 17:59:45  kubelet[1204031]: I0221 17:59:45.964014 1204031 csi_util.go:78] kubernetes.io/csi: loading volume data file [/var/lib/kubelet/pods/{pod-uid}/volumes/kubernetes.io~csi/{disk-id}/vol_data.json]
Feb 21 17:59:45  kubelet[1204031]: I0221 17:59:45.990824 1204031 reader.go:78] Task stats for "/sys/fs/cgroup/cpu,cpuacct/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod0d6c9c59_97e1_43db_ad02_ff274de8755a.slice/cri-containerd-70566c11e47438583e3f92e7979669879912183fa201e49a72b17b3ef6eafa55.scope": {NrSleeping:1 NrRunning:0 NrStopped:0 NrUninterruptible:0 NrIoWait:0}

// 我们删了目录， 如下fd会一直报错，但是是否导致pod删不掉？（最终删掉了）
I0221 19:45:55.099282 1204031 handler.go:293] error while reading "/proc/1204031/fd/9" link: readlink /proc/1204031/fd/9: no such file or directory
```

经过二次试验可知， 上面的fd在mount之后应该马上就被释放了，通过 lsof 命令可知

```
[root@xxxx log]# lsof -p 2113535
COMMAND     PID USER   FD      TYPE             DEVICE SIZE/OFF      NODE NAME
kubelet 2113535 root  cwd       DIR              253,3     4096         2 /
kubelet 2113535 root  rtd       DIR              253,3     4096         2 /
kubelet 2113535 root  txt       REG              253,3 77037960    675118 /usr/bin/kubelet
kubelet 2113535 root  mem       REG              253,3  3299704    657366 /usr/lib64/libc-2.32.so
kubelet 2113535 root  mem       REG              253,3   304440    657380 /usr/lib64/libpthread-2.32.so
kubelet 2113535 root  mem       REG              253,3   130104    657382 /usr/lib64/libresolv-2.32.so
kubelet 2113535 root  mem       REG              253,3   268904    657359 /usr/lib64/ld-2.32.so
kubelet 2113535 root    0r      CHR                1,3      0t0         4 /dev/null
kubelet 2113535 root    1u     unix 0xffff951dca5c97c0      0t0 543438963 type=STREAM
kubelet 2113535 root    2u     unix 0xffff951dca5c97c0      0t0 543438963 type=STREAM
kubelet 2113535 root    3r  a_inode               0,13        0     11596 inotify
kubelet 2113535 root    4u  a_inode               0,13        0     11596 [eventpoll]
kubelet 2113535 root    5r     FIFO               0,12      0t0 543438966 pipe
kubelet 2113535 root    6w     FIFO               0,12      0t0 543438966 pipe
kubelet 2113535 root    7u     unix 0xffff951dca5cb900      0t0 543438974 type=STREAM
kubelet 2113535 root    8u     unix 0xffff951dca5c8e40      0t0 543438975 type=STREAM
kubelet 2113535 root    9u      DIR               0,26      380         1 /sys/fs/cgroup
kubelet 2113535 root   10r      CHR               1,11      0t0        10 /dev/kmsg
kubelet 2113535 root   11u  netlink                         0t0 543439043 GENERIC
kubelet 2113535 root   12r  a_inode               0,13        0     11596 inotify
kubelet 2113535 root   14r  a_inode               0,13        0     11596 inotify
kubelet 2113535 root   15r  a_inode               0,13        0     11596 inotify
kubelet 2113535 root   16u  netlink                         0t0 543438296 GENERIC
kubelet 2113535 root   17u     IPv4          543438254      0t0       TCP localhost:10248 (LISTEN)
kubelet 2113535 root   18u  netlink                         0t0 543529401 GENERIC
kubelet 2113535 root   20u     IPv6          543438248      0t0       TCP *:10250 (LISTEN)
kubelet 2113535 root   21u  netlink                         0t0 543438299 GENERIC
kubelet 2113535 root   23u  netlink                         0t0 543438301 GENERIC
kubelet 2113535 root   24r  a_inode               0,13        0     11596 inotify
kubelet 2113535 root   25u     unix 0xffff951f39d3c980      0t0 543438273 type=STREAM
kubelet 2113535 root   26u  netlink                         0t0 543438300 GENERIC
kubelet 2113535 root   27u  netlink                         0t0 543438302 GENERIC
kubelet 2113535 root   28u  netlink                         0t0 543438303 GENERIC
kubelet 2113535 root   29u  netlink                         0t0 543438304 GENERIC
kubelet 2113535 root   30r  a_inode               0,13        0     11596 inotify
kubelet 2113535 root   31r      CHR               1,11      0t0        10 /dev/kmsg
kubelet 2113535 root   32u  netlink                         0t0 543439042 GENERIC
kubelet 2113535 root   33u  netlink                         0t0 543438305 GENERIC
kubelet 2113535 root   34u  netlink                         0t0 543438306 GENERIC
kubelet 2113535 root   35u  netlink                         0t0 543438307 GENERIC
kubelet 2113535 root   36u  netlink                         0t0 543438308 GENERIC
kubelet 2113535 root   37u  netlink                         0t0 543439046 GENERIC
kubelet 2113535 root   38u  netlink                         0t0 543439044 GENERIC
kubelet 2113535 root   39u  netlink                         0t0 543439045 GENERIC
kubelet 2113535 root   40u  netlink                         0t0 543439047 GENERIC
kubelet 2113535 root   41u  netlink                         0t0 543438309 GENERIC
kubelet 2113535 root   42u  netlink                         0t0 543438310 GENERIC
kubelet 2113535 root   43u  netlink                         0t0 543438311 GENERIC
kubelet 2113535 root   44u  netlink                         0t0 543439048 GENERIC
kubelet 2113535 root   45u  netlink                         0t0 543438312 GENERIC
kubelet 2113535 root   46u  netlink                         0t0 543438313 GENERIC
kubelet 2113535 root   47u  netlink                         0t0 543438314 GENERIC
kubelet 2113535 root   48u  netlink                         0t0 543439049 GENERIC
kubelet 2113535 root   49u  netlink                         0t0 543439050 GENERIC
kubelet 2113535 root   50u  netlink                         0t0 543438315 GENERIC
kubelet 2113535 root   51u  netlink                         0t0 543438316 GENERIC
kubelet 2113535 root   52u  netlink                         0t0 543438317 GENERIC
kubelet 2113535 root   53u  netlink                         0t0 543439051 GENERIC
kubelet 2113535 root   54u  netlink                         0t0 543438318 GENERIC
kubelet 2113535 root   55u  netlink                         0t0 543439052 GENERIC
kubelet 2113535 root   56u  netlink                         0t0 543439053 GENERIC
kubelet 2113535 root   57u  netlink                         0t0 543438334 GENERIC
kubelet 2113535 root   58u  netlink                         0t0 543438335 GENERIC
kubelet 2113535 root   59u  netlink                         0t0 543439054 GENERIC
kubelet 2113535 root   60u  netlink                         0t0 543439055 GENERIC
kubelet 2113535 root   61u  netlink                         0t0 543439056 GENERIC
kubelet 2113535 root   62u  netlink                         0t0 543438336 GENERIC
kubelet 2113535 root   63u  netlink                         0t0 543438337 GENERIC
kubelet 2113535 root   64u  netlink                         0t0 543438338 GENERIC
kubelet 2113535 root   65u  netlink                         0t0 543438339 GENERIC
kubelet 2113535 root   66u  netlink                         0t0 543438340 GENERIC
kubelet 2113535 root   67u  netlink                         0t0 543438341 GENERIC
kubelet 2113535 root   68u  netlink                         0t0 543438342 GENERIC
kubelet 2113535 root   69u  netlink                         0t0 543438343 GENERIC
kubelet 2113535 root   70u  netlink                         0t0 543438344 GENERIC
kubelet 2113535 root   71u  netlink                         0t0 543438345 GENERIC
kubelet 2113535 root   72u  netlink                         0t0 543438346 GENERIC
kubelet 2113535 root   73u  netlink                         0t0 543438347 GENERIC
kubelet 2113535 root   74u  netlink                         0t0 543529812 GENERIC
kubelet 2113535 root   75u  netlink                         0t0 543438349 GENERIC
kubelet 2113535 root   76u  netlink                         0t0 543438350 GENERIC
kubelet 2113535 root   77u  netlink                         0t0 543438351 GENERIC
kubelet 2113535 root   78u  netlink                         0t0 543438352 GENERIC
kubelet 2113535 root   79u  netlink                         0t0 543438353 GENERIC
kubelet 2113535 root   80u  netlink                         0t0 543438354 GENERIC
kubelet 2113535 root   81u  netlink                         0t0 543438355 GENERIC
kubelet 2113535 root   82u     unix 0xffff951ec4589300      0t0 543438365 type=STREAM
kubelet 2113535 root   83u     unix 0xffff951ec65657c0      0t0 543438366 type=STREAM
kubelet 2113535 root   84u     unix 0xffff951ec6564000      0t0 543438368 /var/lib/kubelet/device-plugins/kubelet.sock type=STREAM
kubelet 2113535 root   85r  a_inode               0,13        0     11596 inotify
kubelet 2113535 root   91u  netlink                         0t0 543530046 GENERIC
```

并且在二次实验中，发现上面的找不到fd 的报错其实在 pod running 期间一直存在， 所以并不是 pod 删除不掉的原因.

并且这次 pod 被删掉了..... 就先这样吧， 在这个问题上浪费比较多的时间了 



#### test3

往子目录写入数据
```
[root@xxxx lib]# touch /var/lib/kubelet/pods/{pod-id}/volumes/kubernetes.io~csi/{disk-id}/mount/app-data/test-data
```

尝试umount fd 挂载的路径, 失败
```
[root@xxxxx lib]# umount /var/lib/kubelet/pods/{pod-id}/volumes/kubernetes.io~csi/{disk-id}/mount/app-data
umount: /var/lib/kubelet/pods/{pod-id}/volumes/kubernetes.io~csi/{disk-id}/mount/app-data: not mounted.
```

尝试 fd umount 目标路径, 正常
```
[root@xxxxxx lib]# umount /var/lib/kubelet/pods/{pod-id}/volume-subpaths/{disk-id}/csi-test/0
[root@xxxxxx lib]# 
```

尝试容器写入数据，正常, 看起来fd 被传递到了其他的挂载路径
```
sh-4.4# rm test-data2
sh-4.4# touch /usr/share/redis/test-data2
sh-4.4# ls /usr/share/redis
test-data  test-data2
sh-4.4#
```

如果把所有的挂载路径都umount的话？
```
[root@"" lib]# mount | grep csi
/dev/vdd on /var/lib/kubelet/plugins/kubernetes.io/csi/diskplugin.csi.alibabacloud.com/b8b51917291e7a4dc6b2fc601622fffb0e21d09e71b23bd7115be136edf55fee/globalmount type ext4 (rw,relatime)
/dev/vdd on /var/lib/kubelet/pods/{pod-id}/volumes/kubernetes.io~csi/{disk-id}/mount type ext4 (rw,relatime)
[root@"" lib]# umount /var/lib/kubelet/pods/{pod-id}/volumes/kubernetes.io~csi/{disk-id}/mount
[root@"" lib]# umount /var/lib/kubelet/plugins/kubernetes.io/csi/diskplugin.csi.alibabacloud.com/b8b51917291e7a4dc6b2fc601622fffb0e21d09e71b23bd7115be136edf55fee/globalmount
[root@"" lib]# mount | grep csi

sh-4.4# touch /usr/share/redis/test-data4
sh-4.4# ls /usr/share/redis/
test-data  test-data2  test-data4

[root@"" lib]# mount /dev/vdd /data/test
[root@"" lib]# ls /data/test
aaa  app-data  bbb  lost+found
[root@"" lib]# ls /data/test/app-data/
test-data  test-data2  test-data4
```
依旧可以正常写入， 并且确实写到容器里面了， 证明，相关 subpath 的挂载是和非subpath 是相同的展示形式