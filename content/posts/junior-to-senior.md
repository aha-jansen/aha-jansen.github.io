---
title: "从初级到高级工程师的成长之路"
date: 2023-09-02
draft: false
tags:
  - 职场
  - 成长
categories:
  - 职场
---

最近，我们的服务需要对外开放。在此之前，我们需要进行压测。由于这是一个跨部门的项目，我们没有足够的上游资源，因此必须自行模拟（Mock）服务进行压测。通过使用 `go build -race` 编译，我们可以在压力测试中发现数据竞争问题。

# 问题

## 隐藏的数据竞争

首先，我们进行了100左右的并发压测，并修复了一个数据竞争的问题。这个问题可能非常普遍，因为它很容易犯。其原因可能是 Golang 不建议采用此种方式实现并发。

```go
func foo() chan interface{} {
    out := make(chan interface{})
    in := make(chan interface{})
    go func() {
        defer close(out)
        go func() {
            for v := range in {
                out <- do(v)
            }    
        }()
        if err := produce(in); err != nil {
            out <- err
        } // stuck work
        close(in)
    }()
    return out 
}
```

这里的 `produce` 接收一个 `chan<- interfaceP{}` 并返回 `error` ，是阻塞的，但是 `foo` 是异步返回 `chan interface{}` ，因此需要起一个协程去作为 `out` 的生产者，还需要起一个协程作为`in` 的消费者。

这段代码有无问题？是有的，这里面存在一个数据竞争的 bug ，不知道大伙能不能发现。其实只要关注读写两种操作就行了， 这里面的 `in` 读是通过 `range` 实现的，并且在生产者 `produce(in) ` 返回后使用了 `close(in)` 正确关闭了，因此 `in` 的读写操作是正确的。再来看 `out` 读写操作，`out` 的读写操作出现三个协程里面，一是 `foo` 返回后的协程，二是外面写入 `out <- err` 的协程，三是写入 `out <- do(v)` 的协程，这里约定外面的协程只有`for range` 读取操作，主要考虑 `foo` 里面的两个协程。两个写入操作：

- 第12行的 `out <- err` ，这里的 `out` 一定不是 `nil` 并且一定不是 `closed` ，因此可以正常写入。

- 第8行的 `out <- do(v)` ，这里的 `out` 有可能是 `closed` ，当 `close(in)` 后，`out <- do(v)` 还会继续消费在 `in` 队列中最后一个元素，因此我们并不能保证 `defer close(out)` 发生在这个消费动作之后。所以，在这种情况下，尝试向已关闭的 `out` 写入数据会导致 PANIC。



## 服务端未正常处理空闲连接

在100并发压测正常后，将并发数提升至4000，这时候出现了问题，服务报错：

```
accept tcp 0.0.0.0:8001: accept4: too many open files; retrying in 10ms
```

看起来是文件句柄数过多，使用 `watch -n 1 ss -s` 监测系统的 `socket` 数一直高居不下，并且到2000多就开始出现上述错误，后面通过 `ulimit -n 4096` 将最大文件数提高，但依旧很快到达上限。

这里初步怀疑是客户端在连接时没有正常关闭，也就是 `resp.Body.Close()` 没有正常执行，但是未发现问题，后面将Mock的下游服务和客户端分离部署，分别查看 `socket` 连接数，结果显示 Mock 服务端结果依旧异常，客户端的 `socket` 连接数正常。因此怀疑是 Mock 服务端在结束请求后，连接并没有回收，而是在躺在连接池中，经过与 ChatGPT的对话后，疑问解开了，对话内容总结如下：

> Q: 在客户端使用 resp.Body.Close() 后，golang http 服务端会正确断开连接，释放资源吗？
>
> 
>
> GPT: 在Golang的`http`客户端中，当你调用`resp.Body.Close()`后，客户端会正确关闭与服务端的连接，释放连接占用的资源。但是在某些情况下，HTTP协议的特性可能会导致连接保持，即使客户端已经关闭了该连接。
>
> 
>
> Q: 但是系统显示建立的连接还是存在的，及时客户端已经使用了resp.Body.Close()
>
> GPT: 即使客户端正确关闭了响应体，TCP 连接可能确实会在服务端短暂地保持打开状态。这种情况下，可能是服务端在使用 Keep-alive 策略。
>

ChatGPT YYDS！也就是空闲连接占用，但是客户端没有复用连接，或者说服务端来不及回收空闲连接，`net/http` 的默认参数为：

```go
var DefaultTransport RoundTripper = &Transport{
        Proxy: ProxyFromEnvironment,
        DialContext: (&net.Dialer{
                Timeout:   30 * time.Second,
                KeepAlive: 30 * time.Second,
                DualStack: true,
        }).DialContext,
        ForceAttemptHTTP2:     true,
        MaxIdleConns:          100,
        IdleConnTimeout:       90 * time.Second,
        TLSHandshakeTimeout:   10 * time.Second,
        ExpectContinueTimeout: 1 * time.Second,
}
```

默认的 `IdleConnTimeout` 为 `90s` ，在压测程序过程中根本来不及回收，因此需要将服务端的 `IdleConnTimeout` 设置得短一些。

```go
mux := http.NewServeMux()
mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
	// do something
})
server := &http.Server{
    Addr:        *host,
    Handler:     mux,
    IdleTimeout: 10 * time.Second,
}
fmt.Println(server.ListenAndServe())
```

然后通过 `ss -s` 监测压测过程中的连接数情况，嗯，一切正常！

```sh
ss -s
// 初始状态
Total: 1165 (kernel 11337)
TCP:   806 (estab 20, closed 772, orphaned 0, synrecv 0, timewait 18/0), ports 4260

Transport Total     IP        IPv6
*         11337     -         -
RAW       2         1         1
UDP       5         3         2
TCP       34        30        4
INET      41        34        7
FRAG      0         0         0

// 测试中
Total: 1188 (kernel 11337)
TCP:   816 (estab 29, closed 773, orphaned 0, synrecv 0, timewait 19/0), ports 4260

Transport Total     IP        IPv6
*         11337     -         -
RAW       2         1         1
UDP       5         3         2
TCP       43        39        4
INET      50        43        7
FRAG      0         0         0

// 测试后 测试客户端进程结束
Total: 1185 (kernel 11337)
TCP:   828 (estab 41, closed 774, orphaned 0, synrecv 0, timewait 20/0), ports 4260

Transport Total     IP        IPv6
*         11337     -         -
RAW       2         1         1
UDP       5         3         2
TCP       54        50        4
INET      61        54        7
FRAG      0         0         0
```

结束服务端的程序后，`estab` 状态的 `socket` 数量回归正常。

# 结论

总之，在程序压测时，使用 `go build -race` 能够很好的帮助我们发现数据竞争问题。而在处理模拟的 HTTP 服务时，需要注意客户端关闭请求体并不意味着服务端立即释放资源。服务端要正确处理请求后，才能释放相关资源。为了解决这个问题，我们需要适当地设置连接空闲时间 `IdleConnTimeout`。
