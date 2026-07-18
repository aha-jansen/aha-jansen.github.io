---
title: "Golang 网络程序性能优化"
date: 2023-09-02
draft: false
tags:
  - 技术
  - Go
categories:
  - 技术
---

golang 在接收流式数据时有多个方式，本文以 HTTP 协议下如何处理 `Content-Type: json` 下的响应为例，介绍几种流式协议处理方法。一般而言，我们主要通过 `resp.Body` 来接收响应的消息：

# 方法

## bufio.Reader

```go
func Read(resp *http.Response, out chan <- interface{}) error {
	r := resp.Body
    defer r.Close()
    br := bufio.NewReader(r)

    for {
        m := &Response{} // Response is the struct of response in this request
        buf := make([]byte, 1024)
        l, err := br.Read(buf)
        if err != nil && err != io.EOF {
            return err
        }
        if pe := json.Unmarsha(buf[:l], m); pe != nil {
            return pe
        }
        out <- m
        if err == io.EOF {
            return nil
        }
    }   
}
```

先通过 `bufio.NewReader` 建立缓冲队列，然后再从缓冲队列中消费，这种方式比较灵活，对 `json/plain-text` 都适用。但是会存在消息堆积的隐患，导致两个粘在一起的 json 包解析失败，适用于读取 plain-text ，且协议中响应的内容没有明确的分隔符。



## bufio.NewScanner

```go
func Read(resp *http.Response, out chan <- interface{}) error {
	r := resp.Body
    defer r.Close()
    sc := bufio.NewScanner(r)

    for {
        m := &Response{} // Response is the struct of response in this request
        if !sc.Scan() {
            return sc.Err()
        }
        s := sc.Text()
        if pe := json.Unmarsha(buf[:l], m); pe != nil {
            return pe
        }
        out <- m
    }
}
```

这个方法通过换行符对来到的消息进行分割解析（这里也可以自定义分割函数），这种方法就可以避免出现上述方法中因消费赶不上生产导致粘包出现的解析失败问题，但不太适合 plain-text，这样每次只会流出一行。



## json.Decoder

```go
func Read(resp *http.Response, out chan <- interface{}) error {
	r := resp.Body
    defer r.Close()
    dec := json.NewDecoder(r)
    for {
        m := &Response{} // Response is the struct of response in this request
        if err := dec.Decode(&m); err != nil {
            if err != io.EOF {
                return err
            }
            out <- m
            return nil
        }
        out <- m
    }  
}
```

直接使用 `json.Decoder` 对接收端的 `io.ReadCloser` 进行解码，需要明确知道 Response 的协议是什么，如果 Json 格式不确定，例如 [OpenAI 中的 Chat 接口](https://platform.openai.com/docs/api-reference/chat/create)，则会导致不在预期内的 Json 解析失败。但优点在于简单明了，代码简单。



## httputil.NewChunkedReader

```go
func Read(resp *http.Response, out chan <- interface{}) error {
	r := resp.Body
    defer r.Close()
    cr := httputil.NewChunkedReader(r)
    for {
        m := &Response{} // Response is the struct of response in this request
        buf := make([]byte, 1024)
        l, err := cr.Read(buf)
        if err != nil && err != io.EOF {
            return err
        }
        if pe := json.Unmarsha(buf[:l], m); pe != nil {
            return pe
        }
        out <- m
        if err == io.EOF {
            return nil
        }
    }  
}
```

适用于标准的 HTTP-Chunk 协议，这要求响应的返回的分隔符为  `\r\n` ，然后依次给出消息的大小以及消息的响应体，适用条件比较苛刻。



# 总结

|                           | Freedom | Disadvantage                       | Content-Type    |
| ------------------------- | ------- | ---------------------------------- | --------------- |
| bufio.Reader              | 🌟🌟🌟     | 存在粘包风险                       | plain-text      |
| bufio.NewScanner          | 🌟🌟      | 消息按照分割符分开                 | Plain-text/json |
| json.Decoder              | 🌟       | Body格式需要严格按照给定 json 返回 | json            |
| httputil.NewChunkedReader | 🌟       | 协议需要严格遵循 HTTP-Chunked 协议 | json            |
