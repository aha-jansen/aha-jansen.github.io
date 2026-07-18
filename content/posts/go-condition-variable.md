---
title: "更符合 gopher 的条件变量"
date: 2024-01-15
draft: false
tags:
  - Go
  - 并发
categories:
  - 技术
---

## 背景

有一日业务方反馈线上偶发 `io timeout` 错误，观察线上流量，发现时间点恰好位于流量尖峰处。组内周会上把这个问题抛出来，有同学提出是 TCP 的监听数不足导致的，当时的QPM在 7w 左右，占据流量请求大部分都是流式请求，因此极有可能出现连接数不足导致的等待连接超时。

我们需要实现 4k 请求齐发。如果只是单纯地启动 4k 个 worker 协程，然后在从消息队列中取，这种方式存在一定程度上的延迟，不能保证真正意义上的并发。我们需要的是 4k 个 worker 协程都预备好，只等一声令下，所有worker都开始发出请求。

## C++ Style 实现

使用条件变量实现：

```go
package main

import (
	"log"
	"sync"
	"time"
)

func main() {
	cond := sync.NewCond(&sync.Mutex{})
	ok := false
	nWorker := 5
	var wg sync.WaitGroup
	for i := 0; i < nWorker; i++ {
		wg.Add(1)
		go func() {
			cond.L.Lock()
			for !ok {
				cond.Wait()
			}
			cond.L.Unlock()
			time.Sleep(time.Second)
			log.Println(time.Now(), "work done")
			wg.Done()
		}()
	}
	cond.L.Lock()
	ok = true
	cond.Broadcast()
	cond.L.Unlock()
	wg.Wait()
}
```

嗯，看起来非常符合预期，但是整个 `main` 函数的代码长达 20 多行，我们仅仅只是想实现类似 "开闸" 的效果。这种实现方式非常地像 C++ 选手写出来的代码。

## Go Style 实现

go 最核心的观点是：

> 使用通信来共享内存，而不是使用共享内存来通信

刚开始能想到的是通过生产消费模式实现条件变量的效果：

```go
func main() {
	ok := make(chan struct{})
	nWorker := 5
	var wg sync.WaitGroup
	for i := 0; i < nWorker; i++ {
		wg.Add(1)
		go func() {
			<-ok
			time.Sleep(time.Second)
			log.Println(time.Now(), "work done")
			wg.Done()
		}()
	}
	for i := 0; i < nWorker; i++ {
		ok <- struct{}{}
	}
	close(ok)
	wg.Wait()
}
```

但上面的 chan 发送也是需要时间的，这会导致"并发度"变弱。

## 性能测试对比

| 实现方式 | 最大时间差 |
|---------|----------|
| `sync.Cond` | 0.63us |
| `chan with cap 0` | 1.73us |
| `chan with cap nWorker` | 0.7us |

## 最优方案：Closed Channel

核心理念：**receive from closed chan will never block**

```go
func parallelByClosedChan() time.Duration {
	nWorker := 5
	ok := make(chan struct{})
	var wg sync.WaitGroup
	recvT := make(chan time.Time, nWorker)
	for i := 0; i < nWorker; i++ {
		wg.Add(1)
		go func() {
			<-ok
			recvT <- time.Now()
			wg.Done()
		}()
	}
	close(ok)
	wg.Wait()
	// ... 统计时间差
}
```

不再发送消息，而是直接关闭管道！测试结果显示：**0.64us**，这一结果已经与条件变量的实现方式几乎一样了，并且代码更加简洁！

## 总结

在 go 中我们不必仿照 c++ 使用基于互斥量实现的条件变量，我们可以选择更加符合 go 哲学的方式，即通过**通信来共享内存**，尤其是在实现广播操作时，`closed chan` 能够让你的代码更加简洁。

前面的问题的解决方案：通过 `sysctl -a | grep net.core.somaxconn` 查看 POD 的监听队列长度，为 128，随后更改为 4096，再次压测未出现这类错误。
