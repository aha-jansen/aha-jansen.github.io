---
title: "如何使用 cgo 封装推理模型"
date: 2024-08-02
draft: false
tags:
  - Go
  - CGO
  - C++
categories:
  - 技术
---

## 基本原理

CGO的基本使用示例：

```go
package main

//int sum(int a, int b) { return a+b; }
import "C"

func main() {
	println(C.sum(1, 1))
}
```

生成的桩代码用于实现Go语言到C语言函数跨界调用的流程。

## 使用方法

假设现在有一个C++的库 `libeng.so`，以下展示如何使用CGO调用其中的函数：

```cpp
namespace eng {
void callEngine() {}
void init(string& protofile) {}   
}
```

### 第一步：构建不含C++特性的静态库 `libapi.a`

由于CGO不能识别C++特性，需要将涉及C++特性的地方用C语言转化。

**创建 wrapper.h：**
```c
#include "eng.h"
void init(char* protofile) {
    eng::init(string(protofile));
}
void callEngine() {
    eng::callEngine();
}
```

**构建源文件 apic.cc：**
```cpp
#include <errno.h>
extern "C" {
    #include "wrapper.h"
}

void api_init() {
    init();
}

void api_call_engine() {
    callEngine();
}
```

**创建头文件 apic.h：**
```c
void api_init();
void api_call_engine();
```

**使用gcc编译静态库：**
```bash
gcc -c -o api.o apic.cc -I ./
ar rcs libapi.a api.o
```

### 第二步：在Go代码中创建engine包

**engine.go（CGO层）：**
```go
package engine

/*
#cgo CXXFLAGS: -std=c++11
#cgo CFLAGS: -Ipath_to_apic.h
#cgo LDFLAGS: -L${SRCDIR}/relpath_to_api.a/ -lapi -lstdc++
#cgo LDFLAGS: -L${SRCDIR}/relpath_to_libeng.so/ -leng

#include "apic.h"
#include <errno.h>
*/
import "C"

func cgo_init() error {
	return C.api_init()
}

func cgo_call_engine(s string) error {
	return C.api_call_engine(C.CString(s))
}
```

**api.go（封装层）：**
```go
package engine

import "fmt"

func Init() error {
	return cgo_init()
}

func CallEngine(protofile string) error {
    return cgo_call_engine(protofile)
}
```

## 性能损耗分析

### 主要开销来源

1. **C语言函数调用产生的额外开销**：建议将C语言函数调用次数压缩，尽可能少开放接口
2. **C语言函数阻塞导致线程阻塞**：Go无法取得对该线程的管理，因此该线程上无法开辟新的goroutine，线程数可能会暴涨
3. **其他开销**：C语言仍需要手动GC，Go工具无法获取C语言代码的相关堆栈信息，导致调试困难

### 性能对比数据

**CGO调用耗时：32,861 ms | C++直接调用耗时：32,804 ms**

这里的函数为 `void foo() {...}`，调用没有涉及到Go和C变量之间的类型转换。

## 参考资源与最佳实践

**关键要点：**
- Go是强类型语言，CGO中传递的参数类型必须与声明的类型完全一致
- 传递前必须用"C"中的转化函数转换成对应的C类型，不能直接传入Go中类型的变量

**包间类型兼容性问题：** 不同Go包中引入的虚拟C包的类型是不同的，这导致从它们延伸出来的Go类型也是不同的类型。

**最佳实践：**
- 对C模块的封装包，导出函数的输入输出应该都要是Go语言类型
- `#cgo`语句主要影响CFLAGS、CPPFLAGS、CXXFLAGS、FFLAGS和LDFLAGS几个编译器环境变量
- 更严谨的做法是为C语言函数接口定义严格的头文件，然后基于稳定的头文件实现代码

**调试命令：**
```bash
ldd ./libs/libapi.so      # 查看链接依赖
objdump -p api.a          # 查看函数签名
```
