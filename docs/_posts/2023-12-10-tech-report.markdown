---
layout: post
title:  "weekly report"
date:   2023-12-10 22:30:08 +0800
categories: weeklyreport
---


# 读书

### 大话存储

接下来是快照底层架构的详细介绍. 这部分确实要好好看下

### rust

最近看了下引用相关的语法, 感觉上大体就是地址(指针)的概念, 可能跟其他语音有点不同的地方就是 rust 是以变量为主体的, 任何引用的生命周期都不能超过指针的生命周期. 在 rust 里面也有胖指针的的概念, 也就是包含某个值的地址以及与使用该值相关的必要信息的一个两个字的值, 对切片的引用就是一个胖指针. 包含切片地址和其长度信息. 本章主要讲的是如何在方法中正确使用引用. 下面列一下一个简单的给全局变量赋值的几个代码演进版本

```rust

// ver1
static mut STASH: &i32
fn f(p: &i32) { STASH = p;}


// ver2 
// 1. STASH 必须初始化
// 2. STASH 必须在 unsafe block 内被修改, 尽管可能是单线程的应用

static mut STASH: &i32 = &128;
fn f(p: &i32) {
    unsafe {
        STASH = p;
    }
}

// completed
fn f<'a>(p: &'a i32) {...}

// ver2 会报错, lifetime of reference outlives lifetime of borrowed content...

// ver3

static mut STASH: &i32 = &128;
fn f(p: &'static i32) {
    unsafe {
        STASH = p;
    }
}
// this works

```

# 社区

### statefulset pvc resize

上周听社区会议上的人说 statefulset 的东西没有人维护了. 便在社区留言说可以帮忙做一下, 但是至今也没有人回. 也暂时没办法嘞

### scheduler support csinodes driver

提了一个 pr, 本来以为很简单的, 但是没想到居然涉及到了 csi-host-driver 的 csinode 相关的问题. 也是没想到. 这两周统一处理一下吧

https://github.com/kubernetes/kubernetes/pull/122109


# 工作

最近工作上碰上了不少的工单, 还都是非常神奇的问题.

### errors.As

这里传递的 error 要传递二次指针.... 还没有详细看为啥


### mount err

```
E1201 11:50:21.323600    5362 utils.go:101] GRPC error: mounting failed: exit status 32 cmd: 'mount -o bind /home/t4/kubernetes/lib/kubelet/plugins/kubernetes.io/csi/volumeDevices/staging/d-xxxx/d-xxxx /home/t4/kubernetes/lib/kubelet/plugins/kubernetes.io/csi/volumeDevices/publish/d-xxxx/aeb03e38-af77-4b90-a994-95a66511710b' output: "mount: /home/t4/kubernetes/lib/kubelet/plugins/kubernetes.io/csi/volumeDevices/publish/d-xxxx/aeb03e38-af77-4b90-a994-95a66511710b: mount(2) system call failed: No such file or directory.\n"
```
非常奇怪, 这里 mount 报错某个文件不存在, 但是登录到机器上可以看到这个文件, dmesg 里面有很多的异常信息. 目前还在和内核同学一起定位. 需要持续跟进

### device is busy

```
time="2023-12-05T14:47:38+08:00" level=info msg="AttachDisk: Disk d-xxxxxx is already attached to self Instance i-j6c3p7aglcx91hmer3pa, and device is: /dev/vdd"
time="2023-12-05T14:47:38+08:00" level=info msg="NodeStageVolume: Volume Successful Attached: d-xxxxxx, to Node: i-j6c3p7aglcx91hmer3pa, Device: /dev/vdd"
I1205 14:47:38.185685   13524 mount_linux.go:353] `fsck` error fsck from util-linux 2.32.1
/dev/vdd is in use.
e2fsck: Cannot continue, aborting.


E1205 14:47:38.188197   13524 mount_linux.go:179] Mount failed: exit status 32
Mounting command: mount
Mounting arguments: -t ext4 -o shared,defaults /dev/vdd /var/lib/kubelet/plugins/kubernetes.io/csi/pv/d-xxxxxx/globalmount
Output: mount: /var/lib/kubelet/plugins/kubernetes.io/csi/pv/d-xxxxxx/globalmount: /dev/vdd already mounted or mount point busy.

time="2023-12-05T14:47:38+08:00" level=error msg="NodeStageVolume: Volume: d-xxxxxx, Device: /dev/vdd, FormatAndMount error: mount failed: exit status 32\nMounting command: mount\nMounting arguments: -t ext4 -o shared,defaults /dev/vdd /var/lib/kubelet/plugins/kubernetes.io/csi/pv/d-xxxxxx/globalmount\nOutput: mount: /var/lib/kubelet/plugins/kubernetes.io/csi/pv/d-xxxxxx/globalmount: /dev/vdd already mounted or mount point busy.\n"
E1205 14:47:38.188264   13524 utils.go:101] GRPC error: rpc error: code = Internal desc = mount failed: exit status 32
Mounting command: mount
Mounting arguments: -t ext4 -o shared,defaults /dev/vdd /var/lib/kubelet/plugins/kubernetes.io/csi/pv/d-xxxxxx/globalmount
Output: mount: /var/lib/kubelet/plugins/kubernetes.io/csi/pv/d-xxxxxx/globalmount: /dev/vdd already mounted or mount point busy.
```
发现/dev/vdd 一直被占用, lsof 查 /var/lib/kubelet/plugins/kubernetes.io/csi/pv/d-xxxxxx/globalmount 没有异常信息. fuser 查 /dev/vdd 发现一直是 fsck 占用设备, 用 strace 跟了一下, 发现一直卡在 futex. 也不知道一直在等什么, 进一步查询估计也很困难了


### rpm 反向查询

```
rpm -qf /xxx
```

可以根据二进制命令反向查询相关的包
