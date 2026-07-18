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

> ⚠️ 本文完整内容在 CSDN 平台需付费解锁，以下为可见部分摘要。

## 背景

这是该系列的第三期。作者接到了新的模型服务化需求，新模型相对轻量，可在单个pod上创建多个实例。考虑使用 uvicorn 框架本身支持的多进程能力。

## 核心代码

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
        workers=10,
    )
```

---

> 📌 完整内容请访问 [CSDN 原文](https://blog.csdn.net/QSTARTmachine/article/details/140711952)
