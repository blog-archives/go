---
title: 遍历函数类型
order: 20240820
tags: ['range', 'iterator']
---

> [Range Over Function Types](https://go.dev/blog/range-functions)

## 为什么需要迭代器

此前 Go 允许对 `array`、`slice`、`map`、`string`、`channel` 使用 `for/range` 进行遍历。

```go
for k, v := range collection {

}
```

很好用，但问题是：如果自定义了一个数据结构怎么办？例如这个 `Set`，就不能使用 `for/range`，因为 Go 并不知道要怎么遍历它。

```go
type Set[T comparable] struct {
    data map[T]struct{}
}
```

于是 Go 官方提出了一个问题：有泛型容器了，但是没有统一的遍历方式。

## 在没有迭代器之前怎么解决这种问题

### 1. 暴露内部结构

```go
type Set[T comparable] struct {
    Data map[T]struct{}
}

for k := range set.Data {

}
```

问题：
- 暴露实现细节
- 用户依赖内部结构
- 以后不好修改

### 2. 提供回调函数

```go
func (s *Set[T]) Each(fn func(T)) {
    for v := range s.data {
        fn(v)
    }
}

set.Each(func(v int) {
    fmt.Println(v)
})
```

问题：可读性比 `for/range` 稍差。

### 3. 返回 slice

```go
func (s *Set[T]) Values() []T

for _, v := range set.Values() {

}
```

问题：需要一次性分配内存，如果数据量很大，可能会浪费内存、增加延迟。

## 迭代器是什么

> Iterator 是一种 **按需产生数据** 的机制，在 Go 中通过函数实现。

以前是：我给你一个集合，你自己遍历。有了迭代器以后：我负责产生下一个元素，你负责单个处理。

### Go 1.23 的迭代器形式

Go 没有引入下面这种传统设计：

```go
Iterator interface {
    Next()
}
```

而是让 `for/range` 能够适配特殊格式的函数：

```go
func(yield func(T) bool)

func(yield func(K, V) bool)
```

官方称为 push iterator，因为迭代器主动调用 `yield` 来推送数据。

### 为什么 yield 返回 bool

```go
if !yield(i) {
    return
}
```

增加一个 `bool` 返回值是为了支持提前退出。

```go
for n := range Numbers {
    if n == 3 {
        break
    }
}
```

### 简单示例

这里的 `yield` 并不会直接出现在使用方代码中，而是由 `for range` 隐式创建并传递给 iterator。

例如：

```go
func Numbers(yield func(int) bool) {
    for i := 1; i <= 5; i++ {
        if !yield(i) {
            return
        }
    }
}
```

调用：

```go
for n := range Numbers {
    fmt.Println(n)
}
```

实际类似于：

```go
Numbers(func(n int) bool {
    fmt.Println(n)
    return true
})
```

执行流程：

1. `Numbers` 调用 `yield(1)`
2. `yield` 执行循环体：

    ```go
    fmt.Println(n)
    ```

3. 返回 `true`，告诉 iterator 继续生成下一个值
4. 直到 iterator 结束，或者使用方 `break`

例如：

```go
for n := range Numbers {
    fmt.Println(n)

    if n == 3 {
        break
    }
}
```

此时 Go 生成的内部回调会返回 `false`，通知 iterator 停止继续生成数据：

```go
if !yield(i) {
    return
}
```

于是停止继续生成数据。

所以可以理解为：

```
iterator
    |
    | yield(value)
    v
for range 循环体
    |
    | true  -> 继续
    | false -> 停止
```

`yield` 就是 iterator 和使用方之间的控制通道：iterator 通过它发送数据，使用方通过返回值控制是否继续。

Go 选择函数作为 iterator 的实现形式，是为了让自定义数据结构能够复用熟悉的 `for/range` 语法，从而通过一致性提升可读性。

### 代入自定义 Set

```go
import "iter"

type Set[T comparable] struct {
    data map[T]struct{}
}

func (s *Set[T]) All() iter.Seq[T] {
    return func(yield func(T) bool) {
        for v := range s.data {
            if !yield(v) {
                return
            }
        }
    }
}
```

什么是 `iter.Seq`？

> Go 1.23 新增标准库 `iter` 包，用于定义 iterator 类型：

```go
type Seq[V any] func(yield func(V) bool)

type Seq2[K, V any] func(yield func(K, V) bool)
```

需要注意的是：`range over function` 本身是语言特性，并不依赖 `iter` 包；`iter` 只是提供了统一的类型别名，方便在标准库和业务代码中复用。

## Push iterator vs Pull iterator

Push iterator 和 Pull iterator 的主要区别在于「谁控制数据流动」。

* **Push iterator**：iterator 主动调用 `yield` 提供数据 → 适合遍历和流式处理
* **Pull iterator**：消费者主动调用 `Next()` 获取数据 → 适合精细控制读取过程

```go
// Push
func(yield func(int) bool) {
    yield(1)
    yield(2)
    yield(3)
}

// Pull
value, ok := iterator.Next()
```

Go 1.23 的 `range over function` 选择了 push iterator，因为它能让自定义数据结构拥有类似 slice、map 的自然遍历体验。若需要 pull 语义，可通过标准库的 `iter.Pull` / `iter.Pull2` 将 push iterator 转换出来。

## 开发中的实际用途

### 数据库批量读取

处理大量数据库数据时，可以通过 `yield` 逐条返回结果，避免一次性加载所有数据。例如导出千万级用户数据时，查询一条处理一条。

```go
func Users(yield func(User) bool) {
    for rows.Next() {
        user := scanUser(rows)

        if !yield(user) {
            return
        }
    }
}

for user := range Users {
    export(user)
}
```

### 日志文件扫描

读取大文件时，可以通过 `yield` 持续产生每一行内容，调用方按需处理，不需要把整个文件读入内存。

```go
func Lines(path string) iter.Seq[string] {
    return func(yield func(string) bool) {
        file, err := os.Open(path)
        if err != nil {
            return
        }
        defer file.Close()

        scanner := bufio.NewScanner(file)
        for scanner.Scan() {
            if !yield(scanner.Text()) {
                return
            }
        }
    }
}

for line := range Lines("app.log") {
    if strings.Contains(line, "ERROR") {
        alert(line)
    }
}
```

### 自定义数据结构遍历（Push Iterator）

对于树、图等复杂结构，可以实现 iterator，让使用者直接通过 `range` 遍历内部数据。

```go
func (t *Tree) Nodes() iter.Seq[*Tree] {
    return func(yield func(*Tree) bool) {
        if t == nil {
            return
        }

        if !yield(t) {
            return
        }

        if t.Left != nil {
            for node := range t.Left.Nodes() {
                if !yield(node) {
                    return
                }
            }
        }

        if t.Right != nil {
            for node := range t.Right.Nodes() {
                if !yield(node) {
                    return
                }
            }
        }
    }
}

for node := range tree.Nodes() {
    fmt.Println(node.Value)
}
```

### 分页接口读取（Pull Iterator）

调用分页接口时，可以通过 `Next()` 主动获取下一批数据，由业务决定什么时候继续读取。

```go
for {
    users, ok := iterator.Next()
    if !ok {
        break
    }

    process(users)
}
```
