---
layout: post
title:  "weekly report"
date:   2023-03-04 22:30:08 +0800
categories: weeklyreport
---

# 读书

### Extension Framework for File Systems in User space

https://www.usenix.org/system/files/atc19-bijlani.pdf



这篇论文看完了, 其实fuse相关的论文之前也读过一些, 所以整体下来这个既视感还是非常重的. 比如
https://www.usenix.org/conference/osdi22/presentation/zhong
https://www.usenix.org/system/files/conference/fast17/fast17-vangoor.pdf

尤其是第二篇, 这篇论文就像是根据第二篇论文写出来的一个具体实践, 本文实现了一种叫 ExtFuse 的 framework, 它扩展了 default fuse driver, 并且加上了 ebpf 来减少 context switch. 这个是整体的结构图 .

![iommu](/assets/img/extfuse.jpg)


下面简要说一下 extfuse 的工作流程

- 当开始挂载用户文件系统的时候, 需要判断用户 os 是否支持 EXTFUSE framework.
  - fuse driver 发送 FUSE_INT 请求到 用户空间的 user daemon
  - user daemon 判断如果支持, 请求 libExtFUSE init api
  - libExtFUSE 开始 load 含有特殊 handler 的 ebpf 程序 到内核, 并且向注册 EXTFUSE driver. 使用 bpf_load_prog.
  - bpf_load_prog 开始使用 ebpf verifiers 去检查extension 是否合规
  - 如果不支持或者校验出错, 进程可以退出或者使用给默认的 fuse 功能
  - 如果校验成功, extension 会被安装到 bpf_prog_type. 本质上是一个跳转表, fuse
   驱动只需要带上 fuse 的操作代码(比如 FUSE_OPEN)执行一个 bpf_tail_call. 就可以进入到 extension 的逻辑
  - user daemon 校验安装成功之后就会发消息给 ExtFuse 通知关于内核扩展的信息
  - ExtFuse driver 一旦接到通知, 就可以在 ebpf 环境安全的加载并且在运行时执行这些扩展请求, 比如
    - 在 userspace 和内核空间共享数据
    - 将上层请求的数据直接转发给本地文件系统(如果 fuse 是基于本地文件系统做的)
    - 直接传递给 fuse daemon

整个 workflow 还算是比较清晰


# 工作

### hostpath

之前以为所有的外置存储都会在 kubelet 下面有一个路径映射的, 不过最近遇到的一个问题是 hostpath 相关, 使用 hostpath 的外置存储在 kubelet 上的确是没有相关映射路径的. 看了下相关代码, 发现确实是直接将 hostPath 的路径传入到了容器内部

```golang
func (plugin *hostPathPlugin) NewMounter(spec *volume.Spec, pod *v1.Pod, opts volume.VolumeOptions) (volume.Mounter, error) {
	hostPathVolumeSource, readOnly, err := getVolumeSource(spec)
	if err != nil {
		return nil, err
	}

	path := hostPathVolumeSource.Path
	pathType := new(v1.HostPathType)
	if hostPathVolumeSource.Type == nil {
		*pathType = v1.HostPathUnset
	} else {
		pathType = hostPathVolumeSource.Type
	}
	kvh, ok := plugin.host.(volume.KubeletVolumeHost)
	if !ok {
		return nil, fmt.Errorf("plugin volume host does not implement KubeletVolumeHost interface")
	}
	return &hostPathMounter{
		hostPath:      &hostPath{path: path, pathType: pathType},
		readOnly:      readOnly,
		mounter:       plugin.host.GetMounter(plugin.GetPluginName()),
		hu:            kvh.GetHostUtil(),
		noTypeChecker: plugin.noTypeChecker,
	}, nil
}

func (b *hostPathMounter) GetPath() string {
	return b.path
}

// SetUp does nothing.
func (b *hostPathMounter) SetUp(mounterArgs volume.MounterArgs) error {
	err := validation.ValidatePathNoBacksteps(b.GetPath())
	if err != nil {
		return fmt.Errorf("invalid HostPath `%s`: %v", b.GetPath(), err)
	}

	if *b.pathType == v1.HostPathUnset {
		return nil
	}
	if b.noTypeChecker {
		return nil
	} else {
		return checkType(b.GetPath(), b.pathType, b.hu)
	}
}
```