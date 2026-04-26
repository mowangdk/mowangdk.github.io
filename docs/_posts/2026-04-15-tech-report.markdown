---
layout: post
title:  "weekly report"
date:   2026-04-15 22:30:08 +0800
categories: weeklyreport
---

# 读书

## Controller-Runtime 技术细节整理

### Watch Stream 静默中断 (High Risk)
- **问题**: Reflector watch stream可能在运行数小时/数天后静默停止投递事件
- **影响**: Controller完全停止接收CREATE/UPDATE/DELETE事件
- **恢复**: 必须重启controller pod
- **相关Issue**: [kubernetes-sigs/controller-runtime#3428](https://github.com/kubernetes-sigs/controller-runtime/issues/3428)
- **缓解方案**: 配置SyncPeriod强制定期relist
  ```go
  Cache: cache.Options{
      SyncPeriod: ptr.To(10 * time.Minute),
  }
  ```

### Cache Staleness / 409 Conflict
- **问题**: 快速并发更新导致HTTP 409 Conflict，cache持续读取过期数据
- **影响**: Controller基于过期版本操作，持续409错误
- **相关Issue**: [kubernetes-sigs/controller-runtime#1938](https://github.com/kubernetes-sigs/controller-runtime/issues/1938)
- **缓解方案**: 使用`mgr.GetAPIReader()`绕过cache直接读API Server


### 当前配置
```go
RateLimiter: workqueue.NewTypedMaxOfRateLimiter(
    workqueue.NewTypedItemExponentialFailureRateLimiter[reconcile.Request](
        5*time.Millisecond,   // 基础延迟
        1000*time.Second,     // 最大延迟 ~16.7分钟
    ),
    &workqueue.TypedBucketRateLimiter[reconcile.Request]{
        Limiter: rate.NewLimiter(rate.Limit(10), 100), // 10 QPS, 100 burst
    },
)
```

这里可以尝试调整为了一个固定时长的 interval，并且在5min之内固定， 可以缓解因为状态不对导致的重试间隔过长的问题

### Rate Limiter 类型对比

| 类型 | 行为 | 适用场景 |
|------|------|---------|
| **ItemExponentialFailureRateLimiter** | 5ms → 10ms → 20ms → 40ms... → 1000s | 错误重试，指数退避 |
| **ItemFastSlowRateLimiter** | 前N次5s，之后120s | 固定间隔，避免指数增长 |
| **BucketRateLimiter** | 恒定QPS限制 | 全局速率控制 |
| **MaxOfRateLimiter** | 取多个limiter最大值 | 组合策略（当前使用） |

### Requeue vs RequeueAfter

**关键区别**：
```go
// ❌ 触发 rate limiter
return ctrl.Result{Requeue: true}, nil

// ✅ 绕过 rate limiter，精确控制
return ctrl.Result{RequeueAfter: 5 * time.Second}, nil
```

**使用场景**：
- `Requeue: true` - 错误重试（受rate limiter限制）
- `RequeueAfter: X` - 定时轮询（绕过rate limiter）


### Expectation 机制
```go
// 设置期望（在Patch后）
r.expectations.SetExpectationFunc(pod.UID, func(pod *corev1.Pod) bool {
    return controllerutil.ContainsFinalizer(pod, FinalizerVsc)
})

// 检查期望（在reconcile开始时）
if !r.expectations.SatisfyExpectations(&pod) {
    return ctrl.Result{}, nil // 不满足，等待下次事件
}
```

#### 超时逻辑
```go
func (e expectationWithDeadline) Satisfy(pod *corev1.Pod) bool {
    // 超时 或 满足条件，都返回true
    return time.Now().After(e.deadline) || e.PodExpectation.Satisfy(pod)
}
```
- **默认超时**: 3秒
- **作用**: 防止因informer延迟导致永久阻塞


### 在 controller-rumtime 里面记录 HTTP 请求日志

#### 实现
```go
type loggingTransport struct {
    rt http.RoundTripper
}

func (t *loggingTransport) RoundTrip(req *http.Request) (*http.Response, error) {
    start := time.Now()
    resp, err := t.rt.RoundTrip(req)
    duration := time.Since(start)
    
    // 只记录K8s API请求
    isK8sAPI := strings.Contains(req.URL.Path, "/api/") || 
                strings.Contains(req.URL.Path, "/apis/")
    
    if isK8sAPI && duration > time.Second {
        ctrl.Log.Info("slow Kubernetes API request",
            "method", req.Method,
            "url", req.URL.Path,
            "duration", duration,
            "statusCode", resp.StatusCode,
        )
    }
    return resp, err
}

// 应用配置
cfg.WrapTransport = func(rt http.RoundTripper) http.RoundTripper {
    return &loggingTransport{rt: rt}
}
```


## 7. Predicate 过滤

### 7.1 当前实现
```go
predicate.NewPredicateFuncs(func(object client.Object) bool {
    // 1. 忽略特定namespace
    if ignoreNsSet.Has(object.GetNamespace()) {
        return false
    }
    
    // 2. 只处理有PVC volumes的pod
    pod := object.(*corev1.Pod)
    for _, volume := range pod.Spec.Volumes {
        if volume.VolumeSource.PersistentVolumeClaim != nil {
            return true
        }
    }
    return false
})
```

### 7.2 潜在问题
- **Pod创建时无PVC，后来通过webhook添加**: 永远不会被reconcile
- **静态过滤**: 只在watch时评估，不比较old/new对象

# 工作

nc -vz -w 3 127.0.1.1 12049

-v: verbose mode 展示详细信息
-z: zero-I/O mode，不对端口发送任何data
-w 3: 设置超时时间 3s
