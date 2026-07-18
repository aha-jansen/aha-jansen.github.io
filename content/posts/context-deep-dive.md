---
title: "Context 深度解析"
date: 2023-09-02
draft: false
tags:
  - 技术
  - Go
categories:
  - 技术
---

本文将对Golang中的context.go源码进行深入剖析，从实际项目应用到整体设计和源码细节，力求让读者对于context有更加清晰全面的认识。首先会介绍在实际开发中，`context` 如何起到链路控制的作用，可以先大概看下，然后带着问题往下读。其次，简单介绍一下 `Context` 接口，然后围绕三个问题来阐述 `Context` 的生命周期。最后再引入一系列源码中使用到的技巧，来深入学习源码。这一篇主要介绍链路控制，因此以 `WithCancel, cancelCtx` 为例，展开相关内容。

#   引言

引用来自[官方](https://go.dev/blog/context) 的一句话：

> At Google, we developed a `context` package that makes it easy to pass request-scoped values, cancellation signals, and deadlines across API boundaries to all the goroutines involved in handling a request. 

简单而言，就是针对RPC场景下，用于传递请求的值、取消信号以及过期时间，最重要的是 `across API boundaries` ，能够跨 `API` 。因此在很多场景下，我们都能看到：

`func foo(ctx context.Context, ...)` 的身影，而且一般在函数的参数列表的第一个位置，这与`Context` 使用规范是一致的。通过 `WithCancel, WithTimeout, WithDeadline` 可以在父 `Context` 中衍生出子 `Context` 以及用于取消子 `Context` 的函数 `CancelFunc`。和我们预期的一致，取消父 `Context` ，同时也会取消子 `Context` ，调用 `CancelFunc` 会取消子的 `Context` 。

## 应用场景

当我们遇到异常情况，需要及时断开连接释放下游资源。以一个简单的生产者和消费者为例，介绍消费者如何通过 `ctx.Done`  通知生产者停止发送消息：

```go
package main

import (
	"context"
	"fmt"
	"time"
)

func producer(ctx context.Context, out chan<- string) error {
	defer fmt.Println("producer done")
	var pkgIdx int
	for {
		time.Sleep(10 * time.Millisecond)
		select {
		case <-ctx.Done():
			close(out)
			return ctx.Err()
		case out <- fmt.Sprintf("pkg %d", pkgIdx):
			pkgIdx++
		}
	}
}

func main() {
	out := make(chan string)
	ctx, cancelFunc := context.WithCancel(context.TODO())
	go producer(ctx, out)

	fmt.Println("got: ", <-out)
	fmt.Println("got: ", <-out)
    time.Sleep(time.Second)
	cancelFunc()
    time.Sleep(time.Second)
	fmt.Println("consumer done")
}



/* output
got:  pkg 0
got:  pkg 1
producer done
consumer done
*/
```

这里的 `producer` 通过 `chan out` 向消费者发送消息，并且使用 `ctx.Done()` 监测 `ctx` 是否取消。在使用 `Context` 进行链路控制的时候，需要把握住一点的是：**`ctx.Done()` 和读写 chan 必须在同一个 `select-case` 中，避免因为chan读写阻塞导致 `ctx.Done()` 通知失败，引起协程泄漏。**

如果把上述例子中的 `slelect-case` 语句变为：

```go
select {
case <-ctx.Done():
    close(out)
    return ctx.Err()
default:
    out <- fmt.Sprintf("pkg %d", pkgIdx)
    pkgIdx++
}

/* output
got:  pkg 0
got:  pkg 1
consumer done
*/
```

你会发现，直到程序退出前 `producer` 都没有退出，因此生产者协程可以确定是泄漏了。这是因为 `out` 此时已经没有消费动作了，加上 `out` 是无缓存的，因此阻塞在 `out <- ` 动作上了。



# 整体设计

在正式介绍 `Context` 接口详情之前，先仔细品读一下有关于 `context` 使用规范。

1. 不要将 `Context` 存入到结构体中，而是作为函数的第一个参数。至于为什么不建议将 `Context` 存入结构体中，ChatGPT给出以下几点原因：不利于生命周期控制，这会使得 `Context` 的生命周期和结构体的生命周期一致；不利于代码阅读者了解请求的依赖关系，同样也会使得静态分析工具失效；
2. 不要使用 `nil` 作为 `Context` ，毕竟不确定函数内部是否会使用到该 `Context` ，应该使用 `context.TODO()` 或者 `context.Background()` 。
3. `Context` 是可以被多个协程安全并发调用的，这点很关键，因此我们可以将同一个 `Context` 作为多个API的输入参数。

## context接口

```go
type Context interface {
    // (ddl时间，是否设置了ddl)
	Deadline() (deadline time.Time, ok bool)
    // 如果Context被取消了，那么返回一个已关闭的chan，从已关闭的chan中接收到的永远是(nil, false)，不会阻塞。否则返回一个正常的chan，但永远不会接收到任何值，直到被关闭。
    // 如果Context是不可取消的，那么返回nil，这时候接收操作将永远阻塞。
	Done() <-chan struct{}
    // 如果Context没有关闭，那么返回nil，否则返回一个error。
	Err() error
	Value(key any) any
}
```

想想上述接口中 `Done()` 返回的为什么会是仅接收取消信号的chan，为什么除了 `Value(key any) any` 之外  `Context` 似乎没有暴露写入取消信号的方法？因为通常接收一个取消信号的函数只是一个生产者，只负责吐出消息，并不care什么情况下会中断生产。

> the function receiving a cancellation signal is usually not the one that sends the signal.  --- go.dev

##  生命周期

其实在使用 `context` 去做链路控制的时候，一般会思考几个问题：一是如何管理子 `Context`，当父 `Context` 取消时，如何将这个信号广播到子 `Context`；二是子 `Context`取消时， 父 `Context` 是否需要感知到，并将其剔除；三是当调用 `CancelFunc` 时，取消信号是如何通知 `Context` 关闭通道的。`Context` 接口的具体实现有好几种，这里以 `cancelCtx` 为例，解答上述问题：

```go
type cancelCtx struct {
	Context

	mu       sync.Mutex            // protects following fields
	done     atomic.Value          // of chan struct{}, created lazily, closed by first cancel call
	children map[canceler]struct{} // set to nil by the first cancel call
	err      error                 // set to non-nil by the first cancel call
	cause    error                 // set to non-nil by the first cancel call
}

type canceler interface {
	cancel(removeFromParent bool, err, cause error)
	Done() <-chan struct{}
}
```



#### 管理子Context以及信号广播



`cancelCtx.children` 用于存储和管理子 `Context` 关联的 `canceler` ，`canceler` 接口定义了 `cancel()` 函数和 `Done()` 。至于为什么 `children` 存储的不是 `Context` ，这是因为上面提到的，`Context` 没有暴露主动发送取消信号的方法。



但是，这里有两个疑问：

1. `cancel()` 函数具体是如何通知子 `Context` 关闭？

    正如预期的一样，`children` 中所有的 `cancler` 都会递归执行 `cancel()`，并且关闭自身的 `done` 中存储的 `chan struct{}`。也就是 `cancel()` 做了两件事：取消 `children` 中所有的 `canceler` ，关闭自己的 `done` 。并且注意到 `removeFromParent` 该参数仅在执行`CancelFunc` 的时候为 True ，其他情况均为 False。`CancelFunc` 其实就是 Wrapped 的 `cancel()` ，相当于把不可导出的接口通过函数返回让接口可用。

    

2. `canceler` 里面的 `Done() <-chan struct{}` 看起来是多余的？

    在 `context/context.go` 摸索了一圈才知道，在使用 `WithXxx` 创建子 `Context` 时，会执行一次 `propagateCancel(parent Context, child canceler)` ：如果 `parent` 是 `*cancelCtx` 则将其放入到 `parent.children` 中，否则起一个协程进行监听：

    ```go
    go func() {
        select {
        case <-parent.Done():
            child.cancel(false, parent.Err())
        case <-child.Done():
        }
    }()
    ```

    毕竟如果都使用协程监听的方式，对内存消耗太大。`context` 包 内部实现 `Context` 的具体类型都是基于 `cancelCtx` ，除了 `Background, TODO` 这两个不能取消的 `Context` 之外。因此 `canceler` 接口定义了 `Done()` 方法主要是为了能够监测到Gopher自行实现的 `Context` 取消信号。或者换个说法，`canceler` 接口增加 `cancel(removeFromParent bool, err, cause error)` 方法，其实就是为了在使用 `cancelCtx` 时，避免起的协程过多。



总结一下，如果父 `Context` 是不可导出的 `cancelCtx` ，那么取消信号会通过父 `Context`的 `children` 中的 `cancel()` 方法传递到 ；否则，在使用 `WithXxx` 创建子 `Context`时，起一个协程，监听 `<- P.Done()` ，然后调用 `C.cancel()` 传递到子 `Context`。

#### 感知子Context的取消信号

这段代码真正的展示了组合的魅力所在，实际上父 `Context` 本身不用去关心子 `Context` 的取消信号，而是子 `Context ` 通过`c.Context`来得到父 `Context` ，然后将子 `Context` 移除。非常的Amazing！

```go
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
	// ......
	if removeFromParent {
		removeChild(c.Context, c)
	}
}
```

#### 取消信号如何关闭Chan

前面已经提到，`Context` 的取消主要通过 `c.cancel()` 实现，那么我们可能会想到的是，在创建 `Context` 接口时，存储一个 `chan struct{}`，然后在 `cancel` 中关闭，使得从  `Done()`  返回的 `chan strcut{}` 中接收信息永远是非阻塞的：

> | Operation          | A Nil Channel  | A Closed Channel    | A Not-Closed Non-Nil Channel    |
> | ------------------ | -------------- | ------------------- | ------------------------------- |
> | Close              | panic          | panic               | succeed to close (C)            |
> | Send Value To      | block for ever | panic               | block or succeed to send (B)    |
> | Receive Value From | block for ever | **never block (D)** | block or succeed to receive (A) |
>
> reference [Go101](https://go101.org/article/channel.html)

```go
func newCancelCtx(parent Context) cancelCtx {
    ctx := cancelCtx{Context: parent}
    ctx.done.Store(make(chan struct{}))
    return ctx
}

func (c *cancelCtx) cancel(removeFromParent bool, err error) {
    // ...
    d, _ := c.done.Load().(chan struct{})
    close(d)
    // ...
}
```

但是，`cancelCtx` 在这里使用了懒加载的方法，首先在创建 `cancelCtx` 时：

```go
func newCancelCtx(parent Context) cancelCtx {
	return cancelCtx{Context: parent}
}
```

这里并没有显式地调用 `c.done.Store(make(chan struct{}))` 。因此 `cancelCtx.done` 存的是一个 `nil` ，并在 `Done()` 中进行懒创建：

```go
func (c *cancelCtx) Done() <-chan struct{} {
	d := c.done.Load()
	if d != nil {
		return d.(chan struct{})
	}
	c.mu.Lock()
	defer c.mu.Unlock()
	d = c.done.Load()
	if d == nil {
		d = make(chan struct{})
		c.done.Store(d)
	}
	return d.(chan struct{})
}
// check and check and set
```

这里稍微想想，其实是存在些隐患的。如果 `c.Done()` 的调用发生在 `c.cancel()` 之前，那么 `c.Done()` 此时一定不能再去创建一个 `chan` ，因为 `cancel` 已经发生了，那么从 `Done()` 返回的管道中接收信息一定不是阻塞，也就是 `Done()` 返回的一定是一个已经关闭的管道。因此 `c.cancel()` 必须要要对 `done` 这个管道负责：

```go
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
    // ...
    d, _ := c.done.Load().(chan struct{})
	if d == nil {
		c.done.Store(closedchan)
	} else {
		close(d)
	}
    // ...
}
```

还有个有意思且值得借鉴的地方：`c.done.Store(closedchan)` 这里的 `closedchan` 是一个全局可复用的变量，可以说是把内存省出天际了：

```go
// closedchan is a reusable closed channel.
var closedchan = make(chan struct{})

func init() {
	close(closedchan)
}
```

# 源码剖析

1. 使用 **WrappedFunc** 作为返回变量，用于返回不可导出的接口或方法

    ```go
    func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
        // ...
    	return &c, func() { c.cancel(true, Canceled) }
    }
    ```

    例如，在实际项目中，我们拿到 `http.resp.Body` 是一个 `io.ReadCloser` ，但是有时候我们不是很想导出 `io.ReadCloser` ，只想返回 `io.Reader` 让函数更抽象，这个时候我们可以这样处理：

    ```go
    type CloseFunc func() error
    
    func loadRespBody() (io.Reader, CloseFunc) {
    	// ...
    	body := resp.Body
    	return body, func() error {
    		return body.Close()
    	}
    }
    ```

    这种写法更可靠，这样比较容易让使用者写出这种代码：

    ```go
    r, closeFunc := loadRespBody()
    defer closeFunc()
    ```

    相比直接返回 `io.ReadCloser` ，使用者很容易忘记 `r.Close()` 关闭连接。

2. 如果某个结构体实现一个接口，但是结构体本身没有成员变量时，使用 `int` 类型而非 `struct{}`，保证同一类型的不同变量地址是一定不同的

    ```go
    // An emptyCtx is never canceled, has no values, and has no deadline. It is not
    // struct{}, since vars of this type must have distinct addresses.
    type emptyCtx int
    ```

    下面这份代码打印的是 `true` 还是 `false` ？

    ```go
    package main
    
    import (
    	"fmt"
    )
    
    type Fooer interface {
    	foo()
    }
    
    type F struct{}
    
    func (*F) foo() {
    	return
    }
    
    func newFooer() Fooer {
    	return new(F)
    }
    
    var (
    	x = newFooer()
    	y = newFooer()
    )
    
    func main() {
    	fmt.Println(x == y)
    }
    ```

    如果你回答的是 `false` ，那么你对 `interface` 一定很熟悉，判断两个 `interface` 是否一致的充要条件是 `data` 和 `type` 两个指针指向的内容都是相等的。如果是 `struct{}` 类型的话，那么 `x, y` 的 `data` 都是空的，因此两者是相等的。如果是 `int` 类型的话，则 `data` 指向两个整数变量，因此不相等。可以通过 `dlv` 工具来观察 `eface` 里面的 `data, _type` 两者是否正如预期的那样：

    ```shell
    # type F struct{}
    (dlv) p *((*runtime.eface)(uintptr(&x)))
    runtime.eface {
            _type: *runtime._type {size: 4856064, ptrdata: 4848672, hash: 815858679, tflag: 0, align: 0, fieldAlign: 0, kind: 0, equal: (unreadable could not find function for 0x8348207610663b49), gcdata: *16, str: 4890816, ptrToThis: 0},
            data: unsafe.Pointer(0x559170),}
    (dlv) p *((*runtime.eface)(uintptr(&y)))
    runtime.eface {
            _type: *runtime._type {size: 4856064, ptrdata: 4848672, hash: 815858679, tflag: 0, align: 0, fieldAlign: 0, kind: 0, equal: (unreadable could not find function for 0x8348207610663b49), gcdata: *16, str: 4890816, ptrToThis: 0},
            data: unsafe.Pointer(0x559170),}
            
            
    # type F int
    (dlv) p *((*runtime.eface)(uintptr(&x)))
    runtime.eface {
            _type: *runtime._type {size: 4856032, ptrdata: 4848736, hash: 815858679, tflag: 0, align: 0, fieldAlign: 0, kind: 0, equal: (unreadable could not find function for 0x8348207610663b49), gcdata: *16, str: 4890784, ptrToThis: 0},
            data: unsafe.Pointer(0xc000138000),}
    (dlv) p *((*runtime.eface)(uintptr(&y)))
    runtime.eface {
            _type: *runtime._type {size: 4856032, ptrdata: 4848736, hash: 815858679, tflag: 0, align: 0, fieldAlign: 0, kind: 0, equal: (unreadable could not find function for 0x8348207610663b49), gcdata: *16, str: 4890784, ptrToThis: 0},
            data: unsafe.Pointer(0xc000138008),}
    ```

    可以看到，当 `F` 是 `struct{}` 类型时，`x, y` 是一样的，而当 `F` 是 `int` 类型时，`_type ` 是一致的，但是 `data` 所指向的地址是不一样。

    因此，使用 `int` 替代 `struct{}` ，可以避免出现明明是不同的变量，但是是相等的这种现象出现。当然，我们平常在使用的时候，判断两个接口变量是否相等的情况也比较少见。但不保证不会出现，代码是否容易踩坑不也是评价代码写得好与不好的一个标准吗🤪？
