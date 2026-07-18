---
title: "fastapi 如何控制并发——其二"
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

书接上文《gunicorn+fastapi 如何控制并发——其一》，后续接到了模型服务化的需求，需要搭建一个demo服务支持业务验证和对比实验。

作者认为之前使用 `Nginx+gunicorn+FastAPI` 的方案过于复杂，寻求更简单的解决方案。经过调研，发现可以使用 `uvicorn`，这是一种 ASGI 服务器框架，主要用于处理异步请求。

**关键区别**：
- FastAPI 使用 ASGI 标准
- 之前使用的 gunicorn 是用于处理同步请求的 WSGI 标准（如 Flask、Django）

## 简单启动方式

可以直接在 Python 代码中启动服务，使用 `python app.py` 启动 HTTP 服务：

```python
import uvicorn
from fastapi import FastAPI

app = FastAPI()


@app.get("/hello")
def predict():
    return "world"


if __name__ == "__main__":
    uvicorn.run(
        app="main:app",
        host="0.0.0.0",
        port=8000,
        log_level="debug",
    )
```

## 并发控制实现

因为 `uvicorn` 基于 ASGI 协议，接收请求时不必等待 worker 空闲，默认 worker 能够异步处理多个请求。

**实现每个 worker 支持 1 个并发的方案**：使用线程锁判断当前 worker 是否繁忙。

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
            time.sleep(1)
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
        port=8000,
        log_level="debug",
        workers=10,
    )
```

## 实现原理

- 通过**非阻塞方式**获取线程锁
- 判断是否能够拿到锁
- 如果能拿到锁，执行业务逻辑，最后释放锁
- 如果不能拿到锁，立刻返回 503 错误（服务繁忙）

## 优势

相比之前 `Nginx+gunicorn+FastAPI` 的方案，这种方法具有以下优势：

1. **更简单** - 不需要复杂的配置
2. **更轻量** - 减少了中间层
3. **易于部署** - 直接运行 Python 脚本即可
4. **支持异步流式** - 如果模型支持异步流式，甚至可以控制 worker 的并发度
