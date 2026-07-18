---
title: "fastapi 如何控制并发——其三"
date: 2024-07-26
draft: false
tags:
  - Python
  - FastAPI
  - 并发
categories:
  - 技术
---

## 背景

我也没想到这个系列能出到第三期，最近又双叒叕接了另外一个模型的服务化需求。相比上几个需求，这个模型比较轻，可以在单个pod上创建多个实例。因此单pod可以支持多个并发，自然而然想到的是，直接使用 uvicorn 框架本身支持多进程的能力，例如：

```python
import uvicorn
from threading import Lock
from fastapi import FastAPI, HTTPException

app = FastAPI()

lock = Lock()
import time


@app.get("/hello")
def predict():
    acquired = lock.acquire(blocking=False)
    if acquired:
        try:
            time.sleep(0.1)
        finally:
            lock.release()
        return "world"
    else:
         raise HTTPException(
            status_code=503,
            detail="service busy",
        )


if __name__ == "__main__":
    uvicorn.run(
        app="main:app",
        host="0.0.0.0",
        port=8080,
        log_level="debug",
        workers=4,
    )
```

## 进程调度的局限性

上面的代码启动了4个worker进程，按道理来说，如果存在4个客户端对其进行压测，成功率应该是100%，但是实际上却不是。这里使用的开源压测软件 [go-stress-testing](https://github.com/link1st/go-stress-testing) 进行测试：

```shell
./go-stress-testing-linux -c 4 -n 100 -u http://localhost:8080/hello
```

测试报告为：

```
处理协程数量: 4
请求总数（并发数*请求数 -c * -n）: 400 总请求时间: 10.154 秒 successNum: 159 failureNum: 241
tp90: 101.000
tp95: 101.000
tp99: 107.000
```

成功率连一半都不到，这显然不符合需求的预期。一开始我以为是因为调度不均匀导致导致的错误，但是通过日志排查，发现4个worker都是有机会被调度到的。正如之前所讨论的，uvicorn 使用的是 ASGI 协议，多个请求是可以重入单个worker的，uvicorn 并没有办法了解进程是否繁忙。因此需要额外的调度策略以及跨进程通信来读写worker的状态，这样会复杂许多。[这里](https://www.uvicorn.org/deployment/) 提供了一些开源的进程管理器，但是我没有深入调研是否存在可实现我们需求的框架，有了解的同学可以在评论区留言。

## 队列管理线程实例

既然进程通信比较复杂，那么我在worker进程内进行线程通信应该简单吧！因此我们转而使用队列管理模型实例：

```python
import uvicorn
import queue
import time
from threading import Lock
from contextlib import asynccontextmanager
from fastapi import FastAPI, HTTPException

lock = Lock()
pool = queue.Queue()
max_workers = 4


class Worker:
    def do(self):
        time.sleep(0.1)
        pass


@asynccontextmanager
async def lifespan(app: FastAPI):
    for i in range(max_workers):
        pool.put(Worker())
    logging.info(f"initialize model done, qsize {pool.qsize()}")
    yield
    logging.info("woker exist, quit now")


app = FastAPI(lifespan=lifespan)


@app.get("/hello")
def hello():
    with lock:
        try:
            item = pool.get(block=False)
        except queue.Empty:
            raise HTTPException(
                status_code=403,
                detail="service busy",
            )
    if item is not None:
        try:
            response = item.do()
        except Exception as e:
            print(f"catch unexpected error: {e}")
            raise HTTPException(
                status_code=500,
                detail="internal error",
            )
        finally:
            with lock:
                pool.put(item)
                pool.task_done()
        print(f"send 200")
        return response
    else:
        raise HTTPException(
            status_code=500,
            detail="internal error",
        )


if __name__ == "__main__":

    uvicorn.run(
        app="main_pool:app",
        host="0.0.0.0",
        port=8080,
        workers=1,
        log_level="debug",
    )
```

注意，我们在 `lifespan` 中注册加载当前的模型实例队列，为什么不是在 `main` 中呢？因为worker进程是主进程的子进程，两者并不是同一个进程，因此不能保证端口起来之前（主进程ready），模型是否加载完毕（子进程ready），这里可以从多worker的 uvicorn 的启动日志中看出来：

```shell
INFO:     Uvicorn running on http://0.0.0.0:8080 (Press CTRL+C to quit)
INFO:     Started parent process [626757]
INFO:     Started server process [626760]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Started server process [626762]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Started server process [626761]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Started server process [626763]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
```

**父进程是先于子进程启动的**。ok，那这里的压测情况是否符合预期呢？

```shell
处理协程数量: 4
请求总数（并发数*请求数 -c * -n）: 400 总请求时间: 10.394 秒 successNum: 400 failureNum: 0
tp90: 104.000
tp95: 105.000
tp99: 105.000
```

在并发为4的情况下，模型实例数也为4的情况下成功率100%，完全符合预期，也能如预期的拒绝6并发的情况下过载的请求：

```shell
处理协程数量: 6
请求总数（并发数*请求数 -c * -n）: 600 总请求时间: 4.672 秒 successNum: 163 failureNum: 437
tp90: 104.000
tp95: 104.000
tp99: 106.000
```

## 性能分析

上面我们使用的是多线程编程，正如我们之前所说的，对于计算密集型服务，python的多进程就是个笑话，这是由于 [GIL](https://xantygc.medium.com/python-and-the-never-ending-history-about-multithreading-15b625d561cb) 的存在。因此我们有必要测试在多核机器上，多线程方案的表现。

| 客户端并发数 | 服务端实例数 | QPS   | 平均耗时 |
| ------------ | ------------ | ----- | -------- |
| 1            | 8            | 16.76 | 59.67    |
| 4            | 8            | 9.73  | 411      |
| 8            | 8            | 15.8  | 506      |
| 4            | 4            | 13.84 | 289.11   |
| 2            | 2            | 20.45 | 57.09    |
| 1            | 1            | 20.82 | 48.04    |

这里的测试机器时16核，可以看到在并发打满情况下，实例数越多平均耗时越大，而且是显著增加。起初，我们这里猜想的是，客户端并发数越多，越容易触发线程调度，引入GIL次数越多，平均耗时增加越明显。

随后我们改用多进程的方式去测试，在客户端并发数为4，服务端进程数为8的情况下，平均耗时也有347ms，并且成功率只有22%，这里的347s存在水分，很多失败的请求只有1～2ms，存在摊还的情况。那可能真的是在并发数上来的情况下确实是耗时增大了，我们猜测是并发多了后，CPU成为了瓶颈，我们在使用多进程方案时，观察CPU利用率，发现确实已经占满了。再次切换到多线程方案，测试发现CPU利用率也是满的。这不是很奇怪吗？由于GIL的存在，每个线程只能在一个CPU核心上面跑，为什么16个核心都占满了呢？这里从GPT获得了最可能的答案：

> 外部调用：如果您的计算线程涉及到外部调用，例如调用C/C++扩展或其他并行库，这些外部调用可能会在多个CPU核心上并行执行，而不受全局解释器锁（GIL）的限制。这可能导致您看到所有CPU核心都被充分利用。

说明这里的模型实例本身就可以充分利用多个CPU核心，因此GIL的限制也就不存在了！因此**对于本身就能充分利用多核进行计算的模型，多线程方案部署并不比多进程方案性能表现差**。

## 结论

uvicorn多进程部署方案的调度策略存在局限性，主进程不能直接获取worker进程的状态，需要额外的通信手段才能实现预期的调度。使用多线程可以解决这一问题，尤其当模型本身就能充分利用多核计算，不受GIL限制，多线程的方案性能并不比多进程差。
