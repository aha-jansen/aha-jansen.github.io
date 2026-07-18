---
title: "Go 项目中比较好的实践方案"
date: 2024-11-22
draft: false
tags:
  - Go
  - 后端
categories:
  - 技术
---

工作两年来，我并未遇到太大的挑战，也没有特别值得夸耀的项目。尽管如此，在日常的杂项工作中，我积累了不少心得，许多实践方法也在思考中逐渐得到优化。因此，我在这里记录下这些心得。

## 一、转发与封装

这个需求相当常见，包括封装上游的请求、接收下游的响应，并将它们封装后发送给上游。当需要扩展其他额外需求时，这里的设计就显得尤为重要。

### 1.1 借助第三方代理封装

如果仅需要支持HTTP协议，我们可以直接使用 `httputil.ReverseProxy`：

```go
director := func(req *http.Request) {
    body, _ := io.ReadAll(req.Body)
    req.Body = io.NopCloser(bytes.NewBuffer(body))
    req.Header.Set("KEY", "Value")
    req.Header.Del("HEADER")
    originURL := req.URL.String()
    req.Host = remote.Host
    req.URL.Scheme = remote.Scheme
    req.URL.Host = remote.Host
    req.URL.Path = path.Join("redirected", req.URL.Path)
    req.URL.RawPath = req.URL.EscapedPath()
    query := req.URL.Query()
    query.Add("ExtraHeader", "Value")
    req.URL.RawQuery = query.Encode()
}

modifyResponse := func(resp *http.Response) error {
    if resp.StatusCode < 200 || resp.StatusCode >= 300 {
        // log
    }
    resp.Body = &ReadWrapper{
        reader: resp.Body,
    }
    return nil
}

p := &httputil.ReverseProxy{
    Director:       director,
    ModifyResponse: modifyResponse,
}
```

这里只需专注于实现业务代码，不需关心如何发送和接收数据包，这也符合Go语言基于接口编程的思想。

### 1.2 自己如何实现

#### 初期方案：使用协程和管道

```go
errCh := make(chan error)
respCh := make(chan []byte)
go RequestServer(ctx, reqBody, respCh, errCh)
loop:
for {
    select {
    case resp, ok := <-respCh:
        if !ok {
            break loop
        }
        // 封装响应并发送
    case err, ok := <-errCh:
        // 封装错误并发送
    }
}
```

这里引入了一个协程和两个管道，导致程序的复杂度大大提高。

#### 改进方案：使用缓冲区

```go
var buf Buffer
go RequestServer(ctx, reqBody, &buf)
for {
    n, err := buf.Read(chunk)
    if err != nil {
        if err != io.EOF {
            // 封装错误并发送
        }
        return
    }
    if n > 0 {
        // 封装响应并发送
    }
}
```

#### 最优方案：基于io.Reader接口

根据Practical Go的建议：*"如果主逻辑要从另一个 goroutine 获得结果才能取得进展，那么主逻辑自己完成工作通常比委托他人更简单。"*

```go
type WrappedReader struct {
    rawReader io.ReadCloser
}

func (r *WrappedReader) Read(p []byte) (int, error) {
    raw := make([]byte, cap(p))
    n, err := r.rawReader.Read(raw)
    // 封装响应
}
func (r *WrappedReader) Close() error {
    return r.rawReader.Close()
}

wrappedReader, err := ConnectServer(ctx, reqBody)
if err != nil {
    return err
}
defer wrappedReader.Close()
for {
    n, err := wrappedReader.Read(chunk)
    if err != nil {
        if err != io.EOF {
            return err
        }
        return nil
    }
    if n > 0 {
        // 发送响应
    }
}
```

全部改成了同步逻辑，不存在异步通信！并且在扩展类似计费、限流、鉴权等功能时，不会污染转发的主逻辑。

**关键提示**：在定义接口时尽量多考虑更抽象更底层的行为，也就是go中已有的接口定义，通过这些接口组合得到最终的接口，这样可能往往是较好的设计。

## 二、配置文件

### 2.1 配置文件的读取

使用 `viper` 支持在多个路径下查找文件：

```go
package config

import (
	"fmt"
	"path"

	"github.com/spf13/viper"
)

var (
	G         *viper.Viper
	Workspace string
)

func init() {
	G = viper.New()
	G.SetConfigName("config")
	G.SetConfigType("yaml")
	G.AddConfigPath("../")
	G.AddConfigPath(".")
	err := G.ReadInConfig()
	if err != nil {
		panic(fmt.Errorf("fatal error config file: %w", err))
	}
	Workspace = path.Dir(G.ConfigFileUsed())
}
```

### 2.2 配置文件格式

`viper` 允许无结构体定义直接读取配置项：

```go
var G *viper.Viper
region := config.G.GetString("host.region")
namespace := config.G.GetString("namespace")
```

## 三、JSON序列化

推荐 `json-iterator/go` 第三方库：

```go
import jsoniter "github.com/json-iterator/go"

var json = jsoniter.Config{
	IndentionStep:          2,
	EscapeHTML:             true,
	SortMapKeys:            true,
	ValidateJsonRawMessage: true,
}.Froze()
```

通过 `Config` 设置缩进以及是否转义，通过注入 `Extension` 来避免零值在序列化时被忽略。使用的语法和标准的 `encoding/json` 包是一致的，可以无缝替代历史代码。

## 未完待续
