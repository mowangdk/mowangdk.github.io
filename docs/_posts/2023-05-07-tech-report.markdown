---
layout: post
title:  "weekly report"
date:   2023-05-07 22:30:08 +0800
categories: weeklyreport
---

# 读书

这一周主要读了其他的书, 下周开始恢复正轨

# 工作

### pod 同时挂载两个pvc, 但是只有一个 pvc 挂载上了

尽管这周只上了三天的班, 但是还是遇到了很有意思的问题, 一个客户想要在一个 pod 里面同时使用两个 pvc, 这本来是一件非常正常的事情, 但是神奇的是, 只有一个 pvc(pvc2) 挂载上了, 并且另一个 pvc(pvc1) 始终改在不上, 两个 pvc 分别挂在的话却还是可以挂上. 只有两个 pvc 使用在一个 pod 上就不行.像是被 kubelet 完全忽略了一样. 定位流程如下

- 首先查看 csi 请求, 发现请求根本没有到 csi-plugin 里面
- 然后开始查看 kubelet 日志, 发现 pvc1 有相关的处理逻辑, pvc2 的处理逻辑像是中断了一样
- 于是判断是 kubelet 出了问题, 尝试在本地集群复现(并且复现成功了)
- 修改 kubelet 日志 level 查看具体日志
- 发现 pvc2 的逻辑到了这步骤之后就中断了, 发现 kubelet 确实将 pv-1 加到了desired state 里面, 没有被忽略掉
```
 Added volume "disk-pvc1" (volSpec="pv-1") for pod "xxxx" to desired state.
```
- 那就需要定位 kubelet 相关部分代码开始查问题,

```golang
		// Add volume to desired state of world
		_, err = dswp.desiredStateOfWorld.AddPodToVolume(
			uniquePodName, pod, volumeSpec, podVolume.Name, volumeGidValue)
		if err != nil {
			klog.Errorf(
				"Failed to add volume %s (specName: %s) for pod %q to desiredStateOfWorld: %v",
				podVolume.Name,
				volumeSpec.Name(),
				uniquePodName,
				err)
			dswp.desiredStateOfWorld.AddErrorToPod(uniquePodName, err.Error())
			allVolumesAdded = false
		} else {
			klog.V(4).Infof(
				"Added volume %q (volSpec=%q) for pod %q to desired state.",
				podVolume.Name,
				volumeSpec.Name(),
				uniquePodName)
		}
```
- 查看 AddPodToVolume 方法, 看看它是怎么被加进去的

```golang
	// Create new podToMount object. If it already exists, it is refreshed with
	// updated values (this is required for volumes that require remounting on
	// pod update, like Downward API volumes).
	dsw.volumesToMount[volumeName].podsToMount[podName] = podToMount{
		podName:             podName,
		pod:                 pod,
		volumeSpec:          volumeSpec,
		outerVolumeSpecName: outerVolumeSpecName,
	}
```
- 可以看到他是将 volumeName 作为 key 存储到 map 结构里面, 那么 volumeName 又是什么? 由于我们使用的 csi , 所以直接跳转看 csi 实现

```golang
func GetUniqueVolumeNameFromSpec(
	volumePlugin volume.VolumePlugin,
	volumeSpec *volume.Spec) (v1.UniqueVolumeName, error) {
	if volumePlugin == nil {
		return "", fmt.Errorf(
			"volumePlugin should not be nil. volumeSpec.Name=%q",
			volumeSpec.Name())
	}

	volumeName, err := volumePlugin.GetVolumeName(volumeSpec)
	if err != nil || volumeName == "" {
		return "", fmt.Errorf(
			"failed to GetVolumeName from volumePlugin for volumeSpec %q err=%v",
			volumeSpec.Name(),
			err)
	}

	return GetUniqueVolumeName(
			volumePlugin.GetPluginName(),
			volumeName),
		nil
}

func (p *csiPlugin) GetVolumeName(spec *volume.Spec) (string, error) {
	csi, err := getPVSourceFromSpec(spec)
	if err != nil {
		return "", errors.New(log("plugin.GetVolumeName failed to extract volume source from spec: %v", err))
	}

	// return driverName<separator>volumeHandle
	return fmt.Sprintf("%s%s%s", csi.Driver, volNameSep, csi.VolumeHandle), nil
}
```
- 可以看到这里是直接拿 csi.VolumeHandle 拼接的, 那么如果两个 pv 声明的 VolumeHandle 一样, 那么很有可能一个 pv 就会被另一个 pv 覆盖
- 查看用户 pvyaml 发现确实如此, 问题解决

小结: 其实这个问题实际上非常简单, 但是他引起的现象又特别奇特, 集群中既没有明显报错,也没有相关可以排查的信息,初次见到就需要花费些时间排查, 一旦了解这个机制之后, 之后的问题就都好办了


### 内核报警

用户集群突然报警 ```EXT4-fs error (device vde): ext4_find_entry reading directory lblock 0``` 登录到节点上看确实有这个 dmesg, 
初期看十分的迷惑, 但是尝试看 device vde 这个设备的时候就豁然开朗了, 发现节点上这个设备还不存在, 但是这个设备还有对应的挂载点, 就是所谓的挂载点残留了, 之前都没注意过居然会报这个错, 需要简单记录下