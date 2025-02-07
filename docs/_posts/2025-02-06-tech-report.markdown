---
layout: post
title:  "weekly report"
date:   2025-02-06 22:30:08 +0800
categories: weeklyreport
---


# 读书


# 社区
过年在家 & 路上把 CSI AIO 的 KEP 完善了一下，总算是有了第一个版本 https://github.com/kubernetes/enhancements/pull/5153


# 工作

### shared libraries

基本上所有的命令都会有动态链接（*.so）, 标准命令也不例外， 例如 mkfs mount 等， 在 debian系统下，相关依赖可以通过 ```dpkg-query --show``` 命令查看， 在 alpine 的系统下尚未找到相关命令依赖的文件。 apk -L info 之类的命令未能列出所有相关文件


### Exec format error

执行一个脚本命令报了 Exec format error， 遇到这个问题一般会想到是架构不对，例如 arm 架构用了 amd 的制品， 或者 amd 架构用了 arm 的制品。 但是其实还有一种情况， 就是脚本里面有 ```#!/bin/bash``` 或者 ```#!/bin/sh``` 这类脚本， 但是这个脚本里面没有 ```#!/usr/bin/env bash``` 或者 ```#!/usr/bin/env sh``` 这类脚本， 那么就会报错。


### golangci-lint missspelling

这部分是采用关键字过滤的，并且这个关键字也已经很久没有更新了。不做推荐

```
https://github.com/golangci/misspell/blob/master/words.go
```

### 探究 GetDeviceMountRefs

1. 在 globalmount 基础上， 再 bind mount 会触发 still mounted
```
Feb  7 10:17:21 iZrj91pohau7722j68x05wZ kubelet[2166]: E0207 10:17:21.799920    2166 nestedpendingoperations.go:348] Operation for "{volumeName:kubernetes.io/csi/diskplugin.csi.alibabacloud.com^d-xxxxxx podName: nodeName:}" failed. No retries permitted until 2025-02-07 10:17:53.799902973 +0800 CST m=+8812235.211760637 (durationBeforeRetry 32s). Error: GetDeviceMountRefs check failed for volume "d-xxxxxx" (UniqueName: "kubernetes.io/csi/diskplugin.csi.alibabacloud.com^d-xxxxxxx") on node "us-west-1.192.168.0.243" : the device mount path "/var/lib/kubelet/plugins/kubernetes.io/csi/diskplugin.csi.alibabacloud.com/0716a7ba1b4a6c275d8d1c96xxxxxxa1794f6867be0e5e2ac303f69a5/globalmount" is still mounted by other references [/tmp/test]
```

umount 掉 /tmp/test 即可恢复

2. 直接将块设备挂载到新路径

```
sh-4.4# cat /proc/self/mountinfo | grep /dev/vdb
2249 100 253:16 / /var/lib/kubelet/plugins/kubernetes.io/csi/diskplugin.csi.alibabacloud.com/xxxxxxx/globalmount rw,relatime shared:805 - ext4 /dev/vdb rw
2362 100 253:16 / /var/lib/kubelet/pods/xxxxxx/volumes/kubernetes.io~csi/d-xxxx/mount rw,relatime shared:805 - ext4 /dev/vdb rw
2523 100 253:16 / /tmp/test1 rw,relatime shared:855 - ext4 /dev/vdb rw
```

同样会触发

```
sh-4.4# mount | grep /dev/vdb
/dev/vdb on /var/lib/kubelet/plugins/kubernetes.io/csi/diskplugin.csi.alibabacloud.com/0716a7ba1b4a6c275d8d1c96a1794f68674d6bd7debdd18be0e5e2ac303f69a5/globalmount type ext4 (rw,relatime)
/dev/vdb on /tmp/test1 type ext4 (rw,relatime)
```

3. local volume 实践

```
2036 100 253:16 / /tmp/test rw,relatime shared:798 - xfs /dev/vdb rw,attr2,inode64,logbufs=8,logbsize=32k,noquota
2246 100 253:16 / /var/lib/kubelet/pods/0cfa05f5-xxxxxxx/volumes/kubernetes.io~local-volume/cpfs01 rw,relatime shared:798 - xfs /dev/vdb rw,attr2,inode64,logbufs=8,logbsize=32k,noquota
2508 100 253:16 / /tmp/test1 rw,relatime shared:798 - xfs /dev/vdb rw,attr2,inode64,logbufs=8,logbsize=32k,noquota
```

正常卸载，kubelet 存在报错
```
sh-4.4# cat /proc/self/mountinfo | grep /dev/vdb
2036 100 253:16 / /tmp/test rw,relatime shared:798 - xfs /dev/vdb rw,attr2,inode64,logbufs=8,logbsize=32k,noquota
2508 100 253:16 / /tmp/test1 rw,relatime shared:798 - xfs /dev/vdb rw,attr2,inode64,logbufs=8,logbsize=32k,noquota
```

```
Feb  8 15:17:19 iZrj91pohau7722j68x05wZ kubelet[2166]: E0208 15:17:19.569426    2166 nestedpendingoperations.go:348] Operation for "{volumeName:kubernetes.io/local-volume/cpfs01 podName: nodeName:}" failed. No retries permitted until 2025-02-08 15:19:21.569407553 +0800 CST m=+8916722.981265217 (durationBeforeRetry 2m2s). Error: GetDeviceMountRefs check failed for volume "cpfs01" (UniqueName: "kubernetes.io/local-volume/cpfs01") on node "us-west-1.192.168.0.243" : the device mount path "/tmp/test" is still mounted by other references [/data/test]

```
这个报错意味着即使没有umountDevice（globalmount）但是，相关的流程还是会走的， 并且还基本走完了。但是这个报错完全不影响挂载&卸载


首先，local volume 没有 globalmount, 只有每个 pod 的 mountpoint， 如上所示, 而关于 HasMountRefs 的代码， 则统一都用一个

```
// HasMountRefs checks if the given mountPath has mountRefs.
// TODO: this is a workaround for the unmount device issue caused by gci mounter.
// In GCI cluster, if gci mounter is used for mounting, the container started by mounter
// script will cause additional mounts created in the container. Since these mounts are
// irrelevant to the original mounts, they should be not considered when checking the
// mount references. The current solution is to filter out those mount paths that contain
// the k8s plugin suffix of original mount path.
func HasMountRefs(mountPath string, mountRefs []string) bool {
	// A mountPath typically is like
	//   /var/lib/kubelet/plugins/kubernetes.io/some-plugin/mounts/volume-XXXX
	// Mount refs can look like
	//   /home/somewhere/var/lib/kubelet/plugins/kubernetes.io/some-plugin/...
	// but if /var/lib/kubelet is mounted to a different device a ref might be like
	//   /mnt/some-other-place/kubelet/plugins/kubernetes.io/some-plugin/...
	// Neither of the above should be counted as a mount ref as those are handled
	// by the kubelet. What we're concerned about is a path like
	//   /data/local/some/manual/mount
	// As unmounting could interrupt usage from that mountpoint.
	//
	// So instead of looking for the entire /var/lib/... path, the plugins/kubernetes.io/
	// suffix is trimmed off and searched for.
	//
	// If there isn't a /plugins/... path, the whole mountPath is used instead.
	pathToFind := mountPath
	if i := strings.Index(mountPath, kubernetesPluginPathPrefix); i > -1 {
		pathToFind = mountPath[i:]
	}
	for _, ref := range mountRefs {
		if !strings.Contains(ref, pathToFind) {
			return true
		}
	}
	return false
}
```


相关卸载日志

```
Feb  8 13:47:01 iZrj91pohau7722j68x05wZ kubelet[2166]: I0208 13:47:01.812244    2166 reconciler_common.go:159] "operationExecutor.UnmountVolume started for volume \"config\" (UniqueName: \"kubernetes.io/local-volume/cpfs01\") pod \"xxxxx\" (UID: \"xxxx\") "

Feb  8 13:47:01 iZrj91pohau7722j68x05wZ kubelet[2166]: I0208 13:47:01.821393    2166 operation_generator.go:803] UnmountVolume.TearDown succeeded for volume "kubernetes.io/local-volume/cpfs01" (OuterVolumeSpecName: "config") pod "xxxxx" (UID: "xxxxxx"). InnerVolumeSpecName "cpfs01". PluginName "kubernetes.io/local-volume", VolumeGidValue ""

```
local 相关卸载主流程
```
// CleanupMountPoint unmounts the given path and deletes the remaining directory
// if successful. If extensiveMountPointCheck is true IsNotMountPoint will be
// called instead of IsLikelyNotMountPoint. IsNotMountPoint is more expensive
// but properly handles bind mounts within the same fs.
func CleanupMountPoint(mountPath string, mounter Interface, extensiveMountPointCheck bool) error {
	pathExists, pathErr := PathExists(mountPath)
	if !pathExists && pathErr == nil {
		klog.Warningf("Warning: mount cleanup skipped because path does not exist: %v", mountPath)
		return nil
	}
	corruptedMnt := IsCorruptedMnt(pathErr)
	if pathErr != nil && !corruptedMnt {
		return fmt.Errorf("Error checking path: %v", pathErr)
	}
	return doCleanupMountPoint(mountPath, mounter, extensiveMountPointCheck, corruptedMnt)
}
```

发现相关流程不在卸载流程里面， 反而是在挂载流程里面有reference 的检查， 但是这个 references 并不影响挂载，只跟 volumeOwnership 有关

```golang
// SetUpAt bind mounts the directory to the volume path and sets up volume ownership
func (m *localVolumeMounter) SetUpAt(dir string, mounterArgs volume.MounterArgs) error {
	refs, err := m.mounter.GetMountRefs(m.globalPath)
	if mounterArgs.FsGroup != nil {
		if err != nil {
			klog.Errorf("cannot collect mounting information: %s %v", m.globalPath, err)
			return err
		}

		// Only count mounts from other pods
		refs = m.filterPodMounts(refs)
		if len(refs) > 0 {
			fsGroupNew := int64(*mounterArgs.FsGroup)
			_, fsGroupOld, err := m.hostUtil.GetOwner(m.globalPath)
			if err != nil {
				return fmt.Errorf("failed to check fsGroup for %s (%v)", m.globalPath, err)
			}
			if fsGroupNew != fsGroupOld {
				m.plugin.recorder.Eventf(m.pod, v1.EventTypeWarning, events.WarnAlreadyMountedVolume, "The requested fsGroup is %d, but the volume %s has GID %d. The volume may not be shareable.", fsGroupNew, m.volName, fsGroupOld)
			}
		}

	}
    if !m.readOnly {
		// Volume owner will be written only once on the first volume mount
		if len(refs) == 0 {
			return volume.SetVolumeOwnership(m, dir, mounterArgs.FsGroup, mounterArgs.FSGroupChangePolicy, util.FSGroupCompleteHook(m.plugin, nil))
		}
	}
	return nil
```


### mount 生效， nest 挂载点

创建 /tmp/test， /data/test 目录，首先将 /tmp/test 挂载到 /data/test, 这时相当于把系统盘挂载到了 /data/test 下面

```shell
sh-4.4# mount | grep /data/test
/dev/vda3 on /data/test type ext4 (rw,relatime)
```

这时将 /dev/vdb 挂载到 /tmp/test

```shell
mount /dev/vdb /tmp/test
```

发现 /tmp/test + /data/test 都挂载到了 /dev/vdb 上

```shell
sh-4.4# cat /proc/self/mountinfo | grep test
2036 100 253:3 /tmp/test /data/test rw,relatime shared:1 - ext4 /dev/vda3 rw
2246 100 253:16 / /tmp/test rw,relatime shared:804 - xfs /dev/vdb rw,attr2,inode64,logbufs=8,logbsize=32k,noquota
2247 2036 253:16 / /data/test rw,relatime shared:804 - xfs /dev/vdb rw,attr2,inode64,logbufs=8,logbsize=32k,noquota
```

umount /data/test， 这时会发现 /dev/vdb 的挂载也同样会被卸载掉, 只剩下下面这个

```
2036 100 253:3 /tmp/test /data/test rw,relatime shared:1 - ext4 /dev/vda3 rw
```

第一列 mountID， a unique ID for the mount
第二列 parentID, the ID of the parent mount (or of self for the root of this mount namespace's mount tree)

如果将新挂载堆叠在先前已有挂载的顶部（使其隐藏现有挂载的路径名 P 上堆叠新挂载（从而隐藏现有挂载）则新挂载的父挂载是该位置上的前一个挂载。 因此在查看堆叠在特定位置的所有挂载时、最上面的挂载（对用户显示的）是不是任何其他 mountpoint 的父挂载的mountpoint

如果父挂载位于进程的根目录之外（chroot）则在mountinfo里面不会看到父挂载的信息

第七列 
- shared:x, 表示这个挂载在 group X 中是共享的， 每一个 group 都有一个独立的由内核生成的 ID. 这些id 从 1开始自增，并且有可能被回收如果一个 group 里面没有任何 member

- master:x, 表示这个挂载是一个 对于shared：x 的 slave 挂载

- propagate_from:x, 表示这个挂载是一个从 shared peer group x 接受传播的一个 slave 挂载, 这个标签总是与 master:x 标签一起出现， 这里 X 是一个在进程root目录下的  the closest dominant peer group， 如果在同一个 进程root 下没有 closest dominant peer group, 或者 这个挂载点是一个 immediate master， 那就不会有这个propagate_from 这个字段

- unbindable: 这是一个不可以bind 的挂载点

- 如果什么都没有显示， 则说明这个是一个 private 的挂载点

details： https://man7.org/linux/man-pages/man7/mount_namespaces.7.html 
这里甚至还有例子