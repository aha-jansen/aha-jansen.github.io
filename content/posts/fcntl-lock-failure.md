---
title: "解决 fcntl 偶发的加锁失败问题"
date: 2024-08-03
draft: false
tags:
  - Python
  - Linux
  - 后端
categories:
  - 技术
---

## 问题背景

服务需要加载数据库的配置，但是配置的变更很少。服务是基于 `flask` 框架的多进程服务，因此使用了本地文件去做缓存，使用定时任务去更新本地缓存，为了避免读写冲突，这里使用了 `fcntl` 锁定文件，刚开始读写的操作是这样的：

```python
def read():
    with open(cache_file, 'r') as f:
        fcntl.flock(f.fileno(), fcntl.LOCK_EX)
        data = json.load(f)
        return data

def write():
    with open(cache_file, 'w') as f:
        fcntl.flock(f.fileno(), fcntl.LOCK_EX)
        json.dump(data)
```

## 问题现象

后来某一日在 `json.load()` 出现了问题：

```
json.decoder.JSONDecodeError: Expecting value: line 1 column 1 (char 0)
```

通过查看日志出现的时间点，发现存在一定的规律，出现的时间间隔恰好是五分钟的倍数，秒级别是相等的，毫秒级别是几乎一样的，因此锁定到了定时任务导致读写冲突的问题。

## 问题复现

写了一个demo来复现：

**readfile.py**
```python
import fcntl
import time
import random
import json

filename = "demo.json"
for i in range(1000):
    time.sleep(random.randint(0, 100) * 0.001)
    with open(filename, "r") as f:
        fcntl.flock(f.fileno(), fcntl.LOCK_EX)
        data = json.load(f)
        print("read ok")
```

**writefile.py**
```python
import fcntl
import time
import random
import json

data = {"x": "x", "y": "y"}
filename = "demo.json"
for i in range(1000):
    time.sleep(random.randint(0, 100) * 0.001)
    with open(filename, "w") as f:
        fcntl.flock(f, fcntl.LOCK_EX)
        json.dump(data, f)
        print("write ok")
```

果然，在 `readfile.py` 的程序中出现了异常。

## 问题分析

这里面的 `char 0` 值得关注，会不会是读操作拿到的文件句柄是空的？如果写操作使用的是截断模式，是有可能直接清空文件的。上面的代码，不管是读还是写，都是先获取句柄，然后再使用 `fcntl` 加锁，这样可能会出现下面的时间序：

```
1. writefile: with open(filename, 'w') as f
2. readfile: with open(filename, 'r') as f
3. readfile: fcntl.flock(f, fcntl.LOCK_EX)
4. readfile: data = json.load(f)
```

这个时候 `json.load(f)` 读取的文件句柄就一定是空的，因此会出现上面的问题。

## 解决方案

引入额外的一个文件 `lockfile`，这个文件只用来加锁：

**readfile.py（改进版）**
```python
import fcntl
import time
import random
import json

filename = "demo.json"
for i in range(1000):
    time.sleep(random.randint(0, 100) * 0.001)
    with open("lockfile", "r") as lockf:
        fcntl.flock(lockf, fcntl.LOCK_EX)
        with open(filename, "r") as f:
            data = json.load(f)
            print("read ok")
```

**writefile.py（改进版）**
```python
import fcntl
import time
import random
import json

data = {"x": "x", "y": "y"}
filename = "demo.json"
for i in range(1000):
    time.sleep(random.randint(0, 100) * 0.001)
    with open("lockfile", "w") as lockf:
        fcntl.flock(lockf, fcntl.LOCK_EX)
        with open(filename, "w") as f:
            json.dump(data, f)
            print("write ok")
```

注意，这里使用阻塞模式下的锁，不然没有获取到锁会出现 `IOError`。如果句柄释放的话，持有该句柄的锁也会释放，因此可以使用 `with` 语法糖来实现释放锁。

## 总结

在使用 `fcntl` 加锁时，必须考虑获取句柄的时间顺序是否会产生非预期的结果。虽然 `fcntl` 能够建立后续操作的保护区间，但是获取句柄的时间顺序是无法保证的，因此使用 `fcntl` 加锁的句柄一定不能是操作目标，而应该引入额外的文件。
