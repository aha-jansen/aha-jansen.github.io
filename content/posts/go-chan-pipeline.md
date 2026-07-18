---
title: "一次模块优化——基于 Go chan 的流水线模式"
date: 2023-03-04
draft: false
tags:
  - Go
  - 并发
  - 设计模式
categories:
  - 技术
---

## 背景

最近有一个文本切分的模块，主要作用是将一段文本切分成单句。仿照重构之前的版本使用正则表达式去匹配标点符号，然后按照「单句文本长度最大值」作为阈值，将整段文本切分为多个句子。

```go
func splitWithPeriod(text string) []string {
    reg := regexp.MustCompile(`([\n\t])([^""'])`)
	// use reg to match newline and tab space
}
func splitWithComma(text string) []string {
	reg := regexp.MustCompile(`([,，])([^""'])`)
	// use reg to match comma
}

func Split(text string) []string {
    var results []string
    for _, v := range splitWithPeriod(text) {
        if len(v) < maxLen {
            results = append(results, v)
            continue 
        }
        for _, vv := range splitWithComma(v) {
            if len(vv) < maxLen {
                results = append(results, vv)
                continue
            }
            for _, vvv := range splitForce(vv) {
                results = append(results, vvv)
            } 
        }
    }
    return results;
}
```

上述代码运行逻辑完全是OK的，但后来线上环境遇到一个英文文本的请求，有必要加上按空格进行划分。现在 `Split` 函数圈复杂度就太高了！

## 优化方案

简单分析一下，这些分割函数 `splitWithXXX` 都是按照优先级进行流水线式的处理方式（`Period > Comma > Space`）。

### 责任链模式（不够优雅）

使用设计模式中的「责任链」模式设计，但平白无故多了许多接口，引入的额外代码体量几乎赶上来业务代码。

### Pipeline 模式（最终方案）

想起《Go语言精进之路》书上的 `chan` pipeline 模式：

```go
type splitter func(string) []string
func spawn(do splitter, in <-chan string) <- chan string {
    out := make(chan string)
    go func() {
    	for v := range in {
            out <- do(v)
        }    
        close(out)
    }()
    return out
}

// usage
in := make(chan string)
go func() {
    for _, v := range texts {
    	in <- v   
    }
    close(in)
}()
out := spawn(splitWithSpace, spawn(splitWithComma, spawn(splitWithPeriod, in)))
var results []string
for v := range out {
    results = append(results, v)
}
```

这样看起来就简洁明了了，而且后面再引入其他的分割函数，也能很轻易地加入进来。同步处理也转为了异步处理，理论上是提升了该程序的性能。

此外，考虑到分割函数中很多都是按照正则表达式进行匹配的，将其提取出来作为全局变量，以免每次调用都需要生成一个 `Regexp` 对象。从 `go.doc` 也能得知 `Regexp` 的操作基本是线程安全的：

> A Regexp is safe for concurrent use by multiple goroutines, except for configuration methods.

## 性能对比

| 指标 | 重构前 | 重构后 |
|------|--------|--------|
| 内存分配 | 3325 allocs/op | 975 allocs/op |
| B/op | ~531,000 | ~287,000 |
| ns/op | ~1,250,000 | ~1,300,000 |

内存分配减少约 **71%**！

无论是代码整洁程度还是性能来看，这次重构都是令人振奋的！

## 参考

- 《Go语言精进之路》第33.2条 Go常见的并发模式
- [Go regexp](https://pkg.go.dev/regexp#Regexp)
