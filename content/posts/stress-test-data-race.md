---
title: "服务压测的心路历程——解决数据竞争与空闲连接问题"
date: 2023-06-22
draft: false
tags:
  - Go
  - 后端
  - HTTP
categories:
  - 技术
---

## 背景

最近，我们的服务需要对外开放。在此之前，我们需要进行压测。由于这是一个跨部门的项目，我们没有足够的上游资源，因此必须自行模拟（Mock）服务进行压测。通过使用 `go build -race` 编译，我们可以在压力测试中发现数据竞争问题。

## 隐藏的数据竞争

首先，我们进行了100左右的并发压测，并修复了一个数据竞争的问题。这个问题可能非常普遍，因为它很容易犯。

**代码示例：**

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
        }
        close(in)
    }()
    return out 
}
```

这里的 `produce` 接收一个 `chan<- interface{}` 并返回 `error`，是阻塞的，但是 `foo` 是异步返回 `chan interface{}`。

**问题分析：**

这段代码存在数据竞争的bug。关键在于读写两种操作：

- **`in` 的读写操作**：`in` 的读是通过 `range` 实现的，并且在生产者 `produce(in)` 返回后使用了 `close(in)` 正确关闭了，因此 `in` 的读写操作是正确的。

- **`out` 的读写操作**：`out` 的读写操作出现在三个协程里面：
  1. `foo` 返回后的协程
  2. 外面写入 `out <- err` 的协程
  3. 写入 `out <- do(v)` 的协程

当 `close(in)` 后，`out <- do(v)` 还会继续消费在 `in` 队列中最后一个元素，因此我们并不能保证 `defer close(out)` 发生在这个消费动作之后。在这种情况下，尝试向已关闭的 `out` 写入数据会导致 PANIC。

## 服务端未正常处理空闲连接

在100并发压测正常后，将并发数提升至4000，这时候出现了问题：

```
accept tcp 0.0.0.0:8001: accept4: too many open files; retrying in 10ms
```

看起来是文件句柄数过多，使用 `watch -n 1 ss -s` 监测系统的 `socket` 数一直高居不下。通过 `ulimit -n 4096` 将最大文件数提高，但依旧很快到达上限。

**与ChatGPT的对话总结：**

> 即使客户端正确关闭了响应体，TCP 连接可能确实会在服务端短暂地保持打开状态。这种情况下，可能是服务端在使用 Keep-alive 策略。

**根本原因：**

`net/http` 的默认 `IdleConnTimeout` 为 `90s`，在压测程序过程中根本来不及回收：

```go
var DefaultTransport RoundTripper = &Transport{
    MaxIdleConns:          100,
    IdleConnTimeout:       90 * time.Second,
    // ...
}
```

**解决方案：**

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

通过 `ss -s` 监测压测过程中的连接数情况，结果正常。

## 结论

- 在程序压测时，使用 `go build -race` 能够很好的帮助我们发现数据竞争问题
- 在处理模拟的 HTTP 服务时，需要注意客户端关闭请求体并不意味着服务端立即释放资源
- 服务端要正确处理请求后，才能释放相关资源
- 需要适当地设置连接空闲时间 `IdleConnTimeout`
