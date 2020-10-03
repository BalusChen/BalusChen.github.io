---
title: 一个诡异的map并发读写问题
date: 2020-09-19 23:42:31
tags:
    - golang
    - bug
---

# 一个诡异的 map 并发读写问题

## 背景

这两天线上的一个服务时不时地就报**进程退出**报警，服务主要做的事情就是打包广告相关的数据，即以 rpc 的方式调用各个下游打包数据并组装成一个大的结构返回上游；由于需要调用多个下游，所以同时起了很多 goroutine 去获取数据。

然后照常登陆实例看日志，直接`less xxx.log`然跳到最后一行往上翻，日志很多，大部分都是：

```txt

```

很像 goroutine 泄露的迹象，所以一开始就朝着这个方向去排查，但是找了很久没有什么思路（主要是想用 pprof 的`http://ip:port/pprof/goroutine?debug=2`的方式去查看，结果公司的性能分析平台不太支持`debug=2`的方式，而且线下难以模拟线上环境，所以一团乱麻）。

## continue

后面陆陆续续又报了一些问题，这次把日志从下往上一直翻到错误日志的第一行，结果发现错误日志是：

```txt
fatal error: concurrent map read and map write

goroutine 1 [running]:
runtime.throw(0x10db16c, 0x21)
...
```

即 map 并发读写导致的问题。问题原因确定了，但是错误日志中显示的错误发生的代码位置很诡异。

大致的代码逻辑是这样的：

```go
futures := make([]*async.Future, 0, 3)
if (len(ads1) > 0) {
  futures = append(futures, async.NewFutureWithTimeout(ctx, 300*time.Millisecond, func(context.Context) interface{} {
    return loadMaterial(ads1)
  })
  ...
}

for _, future := range futures {
  res, ok := future.Get()
  if ok {
    buildMaterial(res)
  }
}
```

每种广告都使用一个`future`去 load 数据，这块是起多个 goroutine 并发地调用下游获取数据，然后 load 完毕之后 build 成一个大结构体（build 并没有使用多个 goroutine）。问题出现在`buildMaterial`函数中的一条对map的读语句中，但是很奇怪的是`buildMaterial`中只有对该 map 的读取操作，所有的写入操作都在`loadMaterial`中，那么按理来说是不会发生并发读写的问题的。

后面发现原来问题出现在 future 的实现上面，这里使用的是`NewFutureWithTimeout`，那么要是其中的操作超出了指定的超时值（这里是 300ms），那么就直接返回 nil，但是此时 load 数据的 goroutine 并没有结束；而

## 为什么 panic 没有被捕获？

有一点比较奇怪的是，报警报的是【进程退出】而不是【服务 panic】；一般而言，当 goroutine 因为 panic 而退出，但是 panic 被外层（或是业务代码本身，或是框架代码）的`recover`所捕获时，会报【服务 panic】错误，而当整个服务进程退出时则会报【进程退出】（当然进程退出后，会由 k8s 再尝试拉起）。

那么为什么 map 并发读写没有被`future`包中的以及rpc 框架本身的 recover 所捕获呢？

## 总结

## 参考
