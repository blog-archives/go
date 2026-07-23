---
title: Go maps 的实际应用
order: 20130206
tags: ['map']
---

> [Go maps in action](https://go.dev/blog/maps)

```go
map[KeyType]ValueType
```

KeyType 可以是任何可比较的类型，而 ValueType 则可以是任何类型，包括另一个映射对象！

## 利用零值进行剥削

当键不存在时，映射检索返回零值，这一特性会带来便利。

例如，布尔值映射可作为类集合数据结构使用（请注意，布尔类型的零值为 false）。以下示例会遍历由节点（Node） 构成的链表并打印其值，该示例通过Node 指针构成的映射来检测链表中的环。

```go {8, 12}
type Node struct {
    Next  *Node
    Value interface{}
}
var first *Node

visited := make(map[*Node]bool)
for n := first; n != nil; n = n.Next {
    if visited[n] {
        fmt.Println("cycle detected")
        break
    }
    visited[n] = true
    fmt.Println(n.Value)
}
```

如果已经访问过 n ，那么 `visited[n]` 就等于 true ；如果 n 不存在，那么 `visited[n]` 就等于 false 。无需使用双值形式来检测映射中是否存在 n ，因为默认情况下，零值就代表了该元素的存在。

另一个有用的零值用法是处理切片映射。将值添加到空切片中时，只会分配一个新的切片；因此，只需一步操作即可将值添加到切片映射中，无需检查该键是否存在。在下面的例子中，people 切片被填充了 Person 值。每个 Person 都包含一个 Name ，并且还有一个“喜欢”切片。这个例子创建了一个映射，将每个“喜欢”与对应的 people 切片关联起来。

```go {8}
type Person struct {
    Name  string
    Likes []string
}
var people []*Person

likes := make(map[string][]*Person)
for _, p := range people {
    for _, l := range p.Likes {
        likes[l] = append(likes[l], p)
    }
}
```

## Key 主要类型

映射键可以是任何可比较的类型。

语言规范对此有明确的定义，简而言之，可比较的类型包括布尔值、数字、字符串、指针、通道、接口类型，以及仅包含这些类型的结构体或数组。值得注意的是，切片、映射和函数不在这些类型的列表中；这些类型无法使用 == 进行比较，因此也不应被用作映射键。

## 并发性

映射在并发使用时并不安全：没有明确的规定在同时读写映射时会发生什么情况。如果需要在并发执行的 goroutine 之间读写映射，那么必须通过某种同步机制来协调这些访问操作。一种常见的保护映射的方法是使用 `sync.RWMutex`。

## 迭代顺序

在通过循环遍历映射时，迭代的顺序是不确定的，无法保证每次迭代的顺序与之前相同。如果你需要固定的迭代顺序，就必须维护一个单独的数据结构来记录这个顺序。