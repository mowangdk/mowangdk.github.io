---
layout: post
title:  "weekly report"
date:   2025-01-05 22:30:08 +0800
categories: weeklyreport
---


# 读书

### ebpf learning

这两周又陆陆续续的看了一些， 主要是讲，不使用标准框架的情况下，ebpf 是如何运行的， 这里用了strace 跟踪了系统调用，有一个点还是很好的，之前一直没有注意过， 比如多核的情况下， 我们怎么确定程序运行到那个 vcpu， 万一要从调度了应该怎么办
他们的解决方法相对也比较简单， 就是每一个vcpu 都注册一个点， 这样无论程序运行到了那个 vcpu上， 都不会落下跟踪信息。 接着还有收集信息的问题， 每一个 vcpu 都会单独输出一个buffer(pperf buffer), 除了这种之外还可以用ring buffer， 
这样，所有的信息都会收集到一个ringbuffer 中， 无论整体的效率还是对客的效率都会高很多


# 工作

### VT-x

Intel Virtualization Technology (VT). Formerly known as Vanderpool, this technology enables a CPU to act as if you have several independent computers, 
in order to enable several operating systems to run at the same time on the same machine. In this tutorial we will explain everything you need to know about this technology. 
Intel’s virtualization technology is available in two versions: VT-x, for x86 processors; and VT-i, for Itanium (i.e., IA-64) processors. 
In this tutorial we will be covering the details of the VT-x technology.

VT-x is enabled in the bios if the bios and motherboard support it

No it will not hurt performance if it is enabled, most PC's have it disabled in the bios by default.


### ca-certificates

最近我们在镜像里面去掉了这个 ca-certificates 的 rpm 包， 好像会导致自签名的证书失效， 例如

```
F0101 16:08:41.214772
35878 oss-go:71l Get"https://xxx/api/v1/namespaces/kube-system/configmaps/xxxx": tls: failed to verify certificate: x509: certificate signed by unknown authority
```
因为 kubernetes 里面所有的 apiserver 的请求都是tls加密的，所以会报这个错误。 详细解释下这个错误就是通过 tls 发出的证书没有被访问的特定 server 侧正确的解读。 可能有以下几个原因

- 自签名的证书， server 侧 可能使用自签名的证书， 但是这个自签名的证书默认没有被信任， 我们需要在客户端显示的信任这个证书
- 缺少中间证书： 服务器可能没有提供正确的证书链。中间证书是服务器证书与受信任根证书颁发机构之间的桥梁。确保服务器证书链完整。
- 不信任的证书颁发机构： 证书可能由未列入系统信任存储的证书颁发机构签署。检查该 CA 是否为系统或应用程序所熟知和信任。
- 证书过期或者被吊销
- 错误的地址， 证书可能对您试图访问的域无效。请仔细检查证书的主题或主题备选名称 (SAN) 字段。



ca-certificates 这个包里面包含了顶级公开的根证书机构。按理说如果没有这个包的话curl 域名之类的会失败，但是在我们的环境里面好像没有复现这个问题，需要持续跟踪



### volumeattachment 在 pod 被删除之前删除

#### 背景

##### node yaml 状态

```yaml
volumesAttached:
- devicePath: ""
  name: kubernetes.io/csi/diskplugin.csi.alibabacloud.com^d-2ze0lxxxx
volumesInUse:
- kubernetes.io/csi/diskplugin.csi.alibabacloud.com^d-2ze0lxxxx
```
同样 node 上也没有 outofservice & notready 标记

##### va 状态

```
kubectl get volumeattachments csi-fxxxxxxx -oyaml

apiVersion: storage.k8s.io/v1
kind: VolumeAttachment
metadata:
  annotations:
    csi.alpha.kubernetes.io/node-id: i-xxxxxxxxx
  creationTimestamp: "2024-12-23T04:38:58Z"
  deletionGracePeriodSeconds: 0
  deletionTimestamp: "2024-12-23T04:50:38Z"
  finalizers:
  - external-attacher/diskplugin-csi-alibabacloud-com
  name: csi-f9c08d139b384f12b8ebb070a256fcb2c49b6aa8e40374e8243a65c0ca21f3cd
  resourceVersion: "204304470"
  uid: 28c6cee9-5f40-4f08-b6cb-d0da885563c7
spec:
  attacher: diskplugin.csi.alibabacloud.com
  nodeName: i-xxxxxxxxx
  source:
    persistentVolumeName: d-******
status:
  attached: true
  detachError:
    message: 'rpc error: code = DeadlineExceeded desc = context deadline exceeded'
    time: "2024-12-25T07:19:19Z"
```
##### pod 状态

```
NAME                            READY   STATUS        RESTARTS   AGE    IP           NODE                     NOMINATED NODE   READINESS GATES
sts-ebs-csi-xxxxx-0   0/3     Terminating   0          2d2h   10.3.3.139   i-xxxxxxxxx   <none>           <none>
```

##### kcm log

```
attach 成功：
{"content":"I1223 12:39:01.774065       1 operation_generator.go:410] AttachVolume.Attach succeeded for volume \"d-******\" (UniqueName: \"kubernetes.io/csi/diskplugin.csi.alibabacloud.com^d-******\") from node \"i-xxxxxxxxx\" ","_time_":"2024-12-23T12:39:01.774167361+08:00","_source_":"stderr","_image_name_":"registry-vpc.cn-beijing.aliyuncs.com/ackee/ak8s-ee-kube-controller-manager:v1.24.6.109-xxxxx","_container_name_":"kube-controller-manager","_pod_name_":"controlplane-kcm-8968fc4b9-9gj4w","_namespace_":"ced3930f76b3e47e09f83ddda776a5391","_pod_uid_":"b9c48b94-4c99-461e-8cea-929c273b3839","_container_ip_":"7.10.51.168","__pack_meta__":"28|MTczNDkyNDA5NDY1MDY5MTU5NA==|7|0","__topic__":"","__source__":"7.0.15.166","__tag__:__pack_id__":"2778FDD82A367F4C-11A2A9","__tag__:ALIYUN_LOGTAIL_USER_DEFINED_ID":"k8s-group-asi-cluster-beijing-7-0-2-135","__tag__:__hostname__":"iZ2zegg56xyy8g0dgc481wZ","__tag__:_node_name_":"cn-beijing.7.0.15.166","__tag__:_node_ip_":"7.0.15.166","__tag__:__receive_time__":"1734928745","__time__":"1734928742"}



detach 日志：
{"content":"E1223 12:50:55.193041       1 csi_attacher.go:795] kubernetes.io/csi: detachment for VolumeAttachment for volume [d-******] failed: rpc error: code = Aborted desc = DetachDisk: canceling waiting for disk d-****** detach: context deadline exceeded","_time_":"2024-12-23T12:50:55.193137545+08:00","_source_":"stderr","_image_name_":"registry-vpc.cn-beijing.aliyuncs.com/ackee/ak8s-ee-kube-controller-manager:v1.24.6-alibaba.109-ge7a5aa003","_container_name_":"kube-controller-manager","_pod_name_":"controlplane-kcm-8968fc4b9-9gj4w","_namespace_":"ced3930f76b3e47e09f83ddda776a5391","_pod_uid_":"b9c48b94-4c99-461e-8cea-929c273b3839","_container_ip_":"7.10.51.168","__pack_meta__":"26|MTczNDkyODMwOTI2ODYxNzgyOA==|60|0","__topic__":"","__source__":"7.0.15.166","__tag__:__pack_id__":"2778FDD82A367F4C-11A365","__tag__:_node_name_":"cn-beijing.7.0.15.166","__tag__:_node_ip_":"7.0.15.166","__tag__:ALIYUN_LOGTAIL_USER_DEFINED_ID":"k8s-group-asi-cluster-beijing-7-0-2-135","__tag__:__hostname__":"iZ2zegg56xyy8g0dgc481wZ","__tag__:__receive_time__":"1734929457","__time__":"1734929455"}
```

可以看到审计里面是有 detach 的， 但是神奇的是 detach 的时间跟 va 上的 deletionTimestamp 不一致，猜想可以是关键字过滤掉了中间的日志， 于是开始倒追， 发现 deletionTimestamp 前后没有 kcm 的日志，只能认为是日志等级开启的不够（默认level3）


##### 结论

找了各种资源， 发现所有的资源状态都是正常的。但是va 确莫名奇妙的被删了，于是开始看删除逻辑，找了对应的集群版本的kcm 代码，发现最终居然是代码的问题


```golang
// Check whether timeout has reached the maximum waiting time
timeout := elapsedTime > rc.maxWaitForUnmountDuration

// Trigger detach volume which requires verifying safe to detach step
// If timeout is true, skip verifySafeToDetach check
// If the node has node.kubernetes.io/out-of-service taint with NoExecute effect, skip verifySafeToDetach check
klog.V(5).InfoS("Starting attacherDetacher.DetachVolume", "volume", attachedVolume)
if hasOutOfServiceTaint {
	klog.V(4).Infof("node %q has out-of-service taint", attachedVolume.NodeName)
}

```
基于上面的逻辑，导致一旦 pod删除超时，就会触发detach。 而不是最新版本代码中的，必须pod被完全删掉，并且node 被打标 notready 或者是 outofservice 才会触发 detach
