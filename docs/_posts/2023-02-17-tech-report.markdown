---
layout: post
title:  "weekly report"
date:   2023-02-17 22:30:08 +0800
categories: weeklyreport
---

# 读书

### 30天自制操作系统

这几天看了下内存分配, 看下来还是收货颇丰的, 正好与之前读到的文章对应上, 互为补充.
书中主要介绍了两种 memory management  的模式
1. 使用标记位来将页内存是否空闲进行标记
其中每一个bit 代表 内存中的某一页(4KB) 是否被占用, 这种方式的好处就是简单明确, 并且统计和释放也非常简单. 直接将对应的标记页置为 0/1 即可. 但是同样缺点也很明显, 就是需要占用很多内存, 并且内存管理表大小是跟着内存线性增长的

2. 使用描述符进行内存管理

这种方式描述起来有点麻烦, 简单搞一些代码大概也就明白了

```c
#define MEMMAN_FREES   4096
struct FREEINFO {
    unsigned int addr, size;
}; 
struct MEMMAN {
    int frees, maxfrees, lostsize, losts;
    struct FREEINFO free[MEMMAN_FREES];
};
void memman_init(struct MEMMAN *man) 
{
    man->frees = 0;
    man->maxfrees = 0;
    man->lostsize = 0;
    man->losts = 0;
    return;
}
unsigned int memman_total(struct MEMMAN *man) {
    unsigned int i, t = 0;
    for (i = 0; i < man->frees; i++) {
        t += man->free[i].size
    }
    return t;
}

unsigned int memman_alloc(struct MEMMAN *man, unsigned int size) {
    unsigned int i, a;
    for (i = 0; i < man->frees; i++) {
        // 找到了足够大的内存
        if (man->free[i].size >= size) {
            a = man->free[i].addr;
            man->free[i].addr += size;
            man->free[i].size -= size;
            if (man->free[i].size == 0){
                man->frees--;
                for (;i < man->frees; i++){
                    man->free[i] = man->free[i+1];
                }
            }
            return a;
        }
    }
    // 没有可用空间
    return 0;

}

```

简单来说上面这个内存管理方法可以有效的减少内存管理所使用的内存. 但是同样也增加了内存管理的复杂度, 相关内存的分配和释放变复杂了


# 工作

### klog

之前遇到了集群中大量 pvc/pv 排队挂载的问题, 虽然说 kcm 的性能问题是一直以来的大坑, 不过之前说白了也只是做了一些隔靴搔痒的优化.其实并不实际解决问题. 这次又出来了. 于是尝试用 pprof 看了下性能指标, 发现 klog 这块占用了非常大的一块性能, 需要看下具体的性能瓶颈在哪里? 看了半天也没有找到,最后请教了公司的大佬, 说是 klog 是没有办法关闭日志所在文件/行数的. 而获取这两个信息需要调用 ```runtime.Caller(3 + depth)``` 获取堆栈信息. 这步骤是十分消耗资源的. 想了下也确实是这样... 不愧是大佬

