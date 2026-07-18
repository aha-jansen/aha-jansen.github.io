---
title: "Go Channel 深度分析及优化"
date: 2023-09-02
draft: false
tags:
  - 技术
  - Go
categories:
  - 技术
---

## 背景

最近有一个文本切分的模块，主要作用是将一段文本切分成单句。仿照重构之前的版本使用正则表达式去匹配标点符号，然后按照「单句文本长度最大值」作为阈值，将整段文本切分为多个句子。大致的代码如下：

```go
// 按句号分割
func splitWithPeriod(text string) []string {
    reg := regexp.MustCompile(`([\n\t])([^”"’])`)
	// use reg to match newline and tab space
}
// 按逗号分割
func splitWithComma(text string) []string {
	reg := regexp.MustCompile(`([,，])([^”"’])`)
	// use reg to match comma
}

func splitForce(text string) []string {
    // ....
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

上述代码运行逻辑完全是OK的，但后来线上环境遇到一个英文文本的请求，在做切分的时候会遇到很长一段英文，中间没有标点符号。导致有单词被切割开来，例如 `hello -> hel, lo`，这显然是不能接受的。因此有必要加上按空格进行划分，因此 `Split` 函数变为：

```go
func Split(text string) []string {
    var results []string
    ...
    		
            for _, vvv := range splitWithSpace(vv) {
            	if len(vvv) < maxLen {
            		results = append(results, vvv)	
            	}
                for _, vvvv := range splitForce(vvv) {
                	results = append(results, vvv)
                }
            } 
        }
    }
    return results;
}
```

好吧，现在就有点接受不了了，`Split` 函数圈复杂度太高了！如果后续还要接入其他的分割函数，维护起来不是很难受？这不能忍，不想自己写的这份代码日后慢慢变成“屎山”，立即重构！

## 优化方案

简单分析一下 `splitWithComma, splitWithPeriod, splitWithSpace` 这些函数的定义形式是一致的，有没有办法将这些函数合并缩减到一到两个。仔细思考一下发现是不行的，因为在文本分割里面，标点符号是存在优先级的 `Period > Comma > Space` ，也就是如果句号可以将整段文字分开就不必使用逗号分割。

这些分割函数 `splitWithXXX` 都是按照优先级进行流水线式的处理方式，这样就可以按照设计模式中的「[责任链](https://refactoringguru.cn/design-patterns/chain-of-responsibility)」模式设计：

```go
type Splitter interface {
    Execute(string) []string
    SetNext(Splitter)
}

type basicSplitter struct {
    next Splitter
}

func (this *BasicSplitter) SetNext(s Splitter) {
    this.next = s
}

type splitterWithComma struct {
    basicSplitter
}

func (this *splitterWithComma) Execute(text string) []string {
	// use reg to match newline and tab space
	var results []string
    for _, v := range splitWithComma(text) {
        if len(v) < maxLen {
            results = append(results, vv)
        }
        results = append(results, next.execute(v)...)
    }
    return results;
}

......

func Split(text string) []string {
    periodSplitter := &splitterWithperiod{}
    commaSplitter := &splitterWithComma{}
    spaceSplitter := &splitterWithSpace{}
    ......
    // set next
    periodSplitter.SetNext(commaSplitter)
    commaSplitter.SetNext(periodSplitter)
    ......
    
    return periodSplitter.Execute(text)
}
```



嗯，圈复杂度是下降了，但是平白无故多了许多接口，而且引入了责任链设计模式，引入的额外代码体量几乎赶上来业务代码，还是不够优雅！

想起来之前在《Go语言精进之路》书上看过使用 `chan` 实现的 `pipline` 模式，查阅一下，结合业务逻辑，大致的代码如下：

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
out := spaw(splitWithPeriod, spaw(splitWithComma, spaw(splitWithSpace, in)))
var results []string
for v := range out {
    results = append(results, v)
}
```

嗯，这样看起来就简洁明了了，而且后面再引入其他的分割函数，也能很轻易地加入进来，相比之前的代码，业务逻辑直观了许多，并且同步处理也转为了异步处理，理论上是提升了该程序的性能。

此外，考虑到分割函数中很多都是按照正则表达式进行匹配的，尝试将其提取出来作为全局变量，以免每次调用都需要生成一个 `Regexp` 对象，而且从 `go.doc` 也能得知 `Regexp` 的操作基本是线程安全的：

> A Regexp is safe for concurrent use by multiple goroutines, except for configuration methods.

通过 `go test -bench` 可以得知重构前后的性能对比结果：

```shell
# 重构后
[1]
1032	   1301266 ns/op	  286533 B/op	     974 allocs/op
PASS
ok  		1.470s
[2]
886	   1267392 ns/op	  284943 B/op	     975 allocs/op
PASS
ok  		1.269s
[3]
1017	   1215578 ns/op	  287777 B/op	     975 allocs/op
PASS
ok  		1.364s

重构前
[1]
BenchmarkSplit-8   	     914	   1250922 ns/op	  530060 B/op	    3325 allocs/op
PASS
[2]
ok  		1.284s
BenchmarkSplit-8   	     908	   1464555 ns/op	  531797 B/op	    3325 allocs/op
PASS
ok  		2.487s
[3]
BenchmarkSplit-8   	     981	   1605954 ns/op	  531603 B/op	    3325 allocs/op
PASS
ok  		2.648s
```

无论是代码整洁程度还是性能来看，这次重构都是令人振奋的！（叉会腰～）

## 参考

- 《Go语言精进之路》第33.2条 Go常见的并发模式
-   [Go regexp](https://pkg.go.dev/regexp#Regexp)
