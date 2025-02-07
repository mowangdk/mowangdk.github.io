---
layout: post
title:  "weekly report"
date:   2025-01-10 22:30:08 +0800
categories: weeklyreport
---


# 读书


# 工作

### lvs major minor 返回 -1

```shell
lvs --reportformat json --units b -o lv_kernel_major,lv_kernel_minor --nosuffix -a
```

上面这个命令遇到了 major & minor 返回-1 的情况, 经调查，发现如果lv关联的pv 不存在的话，就会导致这个问题。




### golang flag parse

```
// Parse parses flag definitions from the argument list, which should not
// include the command name. Must be called after all flags in the [FlagSet]
// are defined and before flags are accessed by the program.
// The return value will be [ErrHelp] if -help or -h were set but not defined.
```
根据 Parse 方法的定义， 原则上所有对 flag 变量的调用都要放到 flag parse 之后. 但是经过测试，以下解析却可以正常解析(没有报错)， 
但是 attacher-reconcile-sync 则没有生效

```golang
var reconcileSync time.Duration
flag.DurationVar(&reconcileSync, "attacher-reconcile-sync", -1*time.Minute, "Resync interval of the VolumeAttachment reconciler.")
flag.DurationVar(&ReconcileSync, "reconcile-sync", 1*time.Minute, "Resync interval of the VolumeAttachment reconciler.")
if reconcileSync != -1*time.Minute {
	ReconcileSync = reconcileSync
}
```

详细看了下源码

首先上述代码会将 reconcileSync 指针默认设置为 -1*time.Minute, 放到 Flag 变量里面， 并把这个Flag 变量放到 FlagSet.formal map 里面
```
func newFloat64Value(val float64, p *float64) *float64Value {
	*p = val
	return (*float64Value)(p)
}

// Remember the default value as a string; it won't change.
flag := &Flag{name, usage, value, value.String()}
f.formal[name] = flag
```

所以上面的 if 条件判断的都是默认值，永远也没有办法实际生效，实际生效必须得放到 Parse() 方法之内，那么 parse 又具体做了什么

```golang
	flag, ok := f.formal[name]
	if fv, ok := flag.Value.(boolFlag); ok && fv.IsBoolFlag() { // special case: doesn't need an arg
		if hasValue {
			if err := fv.Set(value); err != nil {
				return false, f.failf("invalid boolean value %q for -%s: %v", value, name, err)
			}
		} else {
			if err := fv.Set("true"); err != nil {
				return false, f.failf("invalid boolean flag %s: %v", name, err)
			}
		}
	} else {
		// It must have a value, which might be the next argument.
		if !hasValue && len(f.args) > 0 {
			// value is the next arg
			hasValue = true
			value, f.args = f.args[0], f.args[1:]
		}
		if !hasValue {
			return false, f.failf("flag needs an argument: -%s", name)
		}
		if err := flag.Value.Set(value); err != nil {
			return false, f.failf("invalid value %q for flag -%s: %v", value, name, err)
		}
	}
	if f.actual == nil {
		f.actual = make(map[string]*Flag)
	}
	f.actual[name] = flag
	return true, nil

```
实际上就是解析命令行参数，然后就是通过flag.SetValue 重新将 volume 设置为参数
