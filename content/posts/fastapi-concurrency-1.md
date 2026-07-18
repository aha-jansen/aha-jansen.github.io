---
title: "fastapi 如何控制并发——其一"
date: 2024-07-17
draft: false
tags:
  - Python
  - FastAPI
  - 并发
categories:
  - 技术
---

## 业务背景

**问题定义：** 单独靠 `gunicorn+fastapi` 很难实现真正的并发控制。这里的并发控制指的是：

- 当并发设置为8时，如果已有8个请求正在处理，应立即拒绝其他请求
- 不进行排队，而是直接拒绝

**业务场景：**
- 单机启动一个进程（`gunicorn:worker=1`）
- 同时只能处理一个请求
- 其他请求全部拒绝

## 尝试过的解决方案

### 1. 排队超时方案（失败）

- Gunicorn 本身没有排队机制
- 只会将请求的 socket 挂起，等待空闲的 worker
- 不存在可设置的排队超时参数
- `timeout` 和 `max_request` 是为了避免后端服务阻塞或内存泄露而重启 worker 的参数

### 2. 套接字限制方案（失败）

- 尝试使用 `backlog` 参数设置 gunicorn 能挂起的最大连接数
- 理论上设置 `backlog=1` 可以限制连接
- 实际效果不理想，操作系统会平衡 backlog 长度和丢弃请求频率
- TCP 底层有重试机制，客户端需要 200ms 才能收到 `dial tcp timeout` 错误

### 3. 业务层实现并发控制（部分成功）

通过中间件和线程锁实现单 worker 并发控制为 1：

```python
import asyncio
import logging
from fastapi.responses import JSONResponse
from fastapi.middleware.cors import CORSMiddleware
from fastapi import FastAPI

app = FastAPI()
app.state.running = False

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"],
)

@app.middleware("http")
async def limit_requests_middleware(request, call_next):
    if app.state.running:
        print("reject request, cause app is running")
        return JSONResponse(status_code=503, content={"message": "Service unavailable"})
    app.state.running = True
    try:
        response = await call_next(request)
        return response
    finally:
        app.state.running = False

@app.get("/hello")
async def echo_feature():
    await asyncio.sleep(6)
    print("hello")
```

**局限性：** 对于计算密集型服务，Python 的异步是"笑话"，因为异步只能用在 IO 阻塞时

### 4. Nginx 反向代理方案（最终方案）

最终采用 Nginx 作为反向代理实现并发控制。

**关键配置：**

```nginx
events {
    use epoll;
    worker_connections  4;
}

http {
    limit_conn_zone all zone=conn_limit:10m;

    server{
        listen 80;
        location / {
            limit_conn conn_limit 1;
            proxy_pass http://0.0.0.0:8080;
        }
    }
}
```

**重要发现：**
- `worker_connections` 指的是单个工作进程能同时打开的连接数
- 包括与代理服务器、后端服务以及客户端的所有连接
- 需要设置为 4 的原因：
  - 设置为 2：任意请求出现 `dial tcp fail`
  - 设置为 3：客户端收到 500 错误，nginx 日志显示 "worker_connections are not enough while connecting to upstream"
  - 设置为 4：正常运行

**结果：** 超过并发数的连接，nginx 会主动关闭，客户端收到 `Post "http://0.0.0.0:80": EOF` 错误
