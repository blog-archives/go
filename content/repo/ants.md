---
title: ants
order: 20260728
tags: ['goroutine', 'pool', '并发']
---

> [panjf2000/ants](https://github.com/panjf2000/ants) · [pkg.go.dev](https://pkg.go.dev/github.com/panjf2000/ants/v2)

Go 里开 goroutine 很便宜，但无上限地 `go func()` 仍可能把内存和调度拖垮。`ants` 提供一个固定容量的 goroutine 池：任务提交进池，由有限个 worker 复用执行，从而把并发度卡在你设定的上限内。

适合批量请求、爬虫、队列消费这类「任务很多、但并发不能无限涨」的场景。需要时还可以动态调整池容量、统一回收资源，并在 worker 里兜住 panic，避免单个任务把进程打崩。手写 channel + worker 也能做到类似效果，但 `ants` 更省事，性能也够用。之后遇到要控并发的地方，可以优先考虑它。

## 最小用法

```go
pool, _ := ants.NewPool(100)
defer pool.Release()

_ = pool.Submit(func() {
    // 任务逻辑
})
```
