---
title: "接口与结构体指针之间的转换引发的一次思考"
date: 2023-09-02
draft: false
tags:
  - 技术
  - Go
categories:
  - 技术
---

## 接口与结构体指针之间的转换引发的一次思考

### 背景

在最近的一次项目中，需要实现几个具有相似功能的 `worker` 。那当然想到的谁是通过实现一个 `BasicWorker` ，然后通过组合来具体实现其他的 `worker`，于是代码就写成了下面的形式：

```go
type Worker interface {
    Do()
}

type BasicWorker struct{}

func (this *BasicWorker) Do() {
	// do something
}

type WorkerA struct {
	BasicWorker
}
type WorkerB struct {
	BasicWorker
}
```

上述代码没啥其他问题，但是业务在创建 `worker` 的时候需要进行参数检验、鉴权之类的操作。因此，这里通过使用 `New` 函数的方式来构造比较合适。

```
func NewWorkerA() *WorkerA {
	return &WorkerA{}
}

func NewWorkerB() *WorkerB {
	if !Check() {
        return nil
    }
	return &WorkerB{}
}
```

好，现在问题来了，如果按照上述实现，请问以下代码是否存在隐患：

```go
var workers []Worker
workers = append(workers, NewWorkerA())
workers = append(workers, NewWorkerB())
for _, worker := range workers {
    if worker != nil {
        worker.Do()
    }
}
```

当然是存在问题的，在运行时，如果 `Check()` 失败，`NewWorkerA, NewWorkerB` 返回 `nil` ，就会出现运行时错误：`panic: runtime error: invalid memory address or nil pointer dereference`。

你可能会想，这，这不对吧？不是已经判断是不是 `nil` 了吗？好，先说结论：

**判断 `interface == nil` 的充要条件为类型和值都为 `nil`**

### 问题修复

之前在《Go语言精进之路》一书上好想看过类似的结论，但由于当时工期比较赶，所以通过 Google 检索换了一种方法来临时修复了改问题，即通过直接判断值是否为空来跳过类型检查：

```go
if reflect.ValueOf(v).IsNil() {
	worker.Do()
}
```

### 进一步验证

上述代码之所以出现了问题，我想应该是发生了从结构体指针到接口的类型转换 `*WorkerA -> Worker` ，使得转换后的变量出现 `type:*WorkerA,value:nil` 的情况。因此做了一下实验，打印变量的一些信息：

```
fmt.Printf("%T, %q", worker)
Got:
*main.WorkerA, %!q(*main.WorkerA=<nil>)
```

上述的结果验证了我的想法。

### 总结

当时看起来也没什么问题，但其实还是不够优雅的，这里正确的做法应该是要符合依赖性倒置原则**「程序要依赖于抽象接口，不要依赖于具体实现」**。对外需要屏蔽具体实现，并返回接口：

```go
type basicWorker struct {}
type workerA struct {
	basicWorker
}
func NewWorkerA() Worker {
    // ......
	return &workerA{}
}
```

并且在 effective go 中也有类似的建议：

> In such cases, the constructor should return an interface value rather than the implementing type. 

> Exporting just the interface makes it clear that it's the behavior that matters, not the implementation, and that other implementations with different properties can mirror the behavior of the original type. It also avoids the need to repeat the documentation on every instance of a common method

进一步，我们在 `context` 源码中也可以类似的风格：

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
	c := withCancel(parent)
	return c, func() { c.cancel(true, Canceled, nil) }
}

func withCancel(parent Context) *cancelCtx {
	......
	return c
}
```

这里 `WithCancel` 返回的是可导出的 `Context` 接口，并且实现类型 `cancelCtx` 是不可导出的。因此，应该遵循这种编码风格：导出接口，隐藏具体实现，这样自然而然就可以避免出现上述问题。

### 参考

- [effective go](https://go.dev/doc/effective_go)
- [Why Golang Nil Is Not Always Nil? Nil Explained](https://codefibershq.com/blog/golang-why-nil-is-not-always-nil)
- [context.Context](https://github.com/golang/go/blob/master/src/context/context.go)
