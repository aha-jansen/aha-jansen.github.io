---
title: "slice共享底层引起的量子纠缠"
date: 2025-07-27
draft: false
tags:
  - Go
  - 后端
categories:
  - 技术
---

需要处理语音大语言模型的输入，输入格式为按固定格式组成的 `[]int` 数组，包含特殊token标识符。为了简化处理，在输入生成端加入了特殊标识符 `-1` 来分隔常量和变量。

## 原始代码实现

```go
func splitSliceByKey(s []int, key int) [][]int {
    var results [][]int
    var start int
    for i, v := range s {
        if v == key {
            if i > start {
                results = append(results, s[start:i])
            }
            start = i + 1
        }
    }
    if start < len(s) {
        results = append(results, s[start:])
    }
    return results
}

func equal(a, b []int) bool {
    if len(a) != len(b) {
        return false
    }
    for i := range a {
        if a[i] != b[i] {
            return false
        }
    }
    return true
}

func Merge(tt [][]int) ([]int, []int) {
    n := len(tt)
    var fixMask, content []int
    for i := 0; i < n; i++ {
        if i%2 == 0 {
            fixMask = append(fixMask, tt[i]...)
        } else {
            content = append(content, tt[i]...)
        }
    }
    // ...
}
```

## 遇到的问题

测试用例运行时出现 `panic: inconsistent mask tokens` 错误，看似不合理，因为常量应该是一致的。

## 问题根源

通过调试发现，当执行以下操作时：

```go
content[i>>1] = append(content[i>>1], tt[i]...)
```

`fixMask` 变量会意外改变。这是因为 `fixMask` 和 `content` 共享了原始slice的底层数组。当向 `content` 追加元素时，会修改共享的底层数组，导致 `fixMask` 中的数据也被改变。

## 解决方案

通过在首次填充时进行深拷贝来避免底层数组共享：

```go
fixMask = append(fixMask, append([]int(nil), tt[i]...))
content = append(content, append([]int(nil), tt[i]...))
```

这种方法通过构造空slice然后append的方式，确保每个元素都有独立的底层数组。

## 关键要点

- **问题本质**：`fixMask` 与 `content` 直接引用了原切片的子切片，导致底层数组共享
- **触发条件**：后续 `append` 操作修改了原数据
- **解决方法**：首次填充时使用 `append([]int(nil), x...)` 进行拷贝
- **性能影响**：虽然会触发多次slice扩容，但仅在处理第一个token时执行，影响不大
