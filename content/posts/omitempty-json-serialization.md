---
title: "带有 omitempty json tag 的结构体序列化"
date: 2024-07-26
draft: false
tags:
  - Go
  - JSON
categories:
  - 技术
---

## 问题背景

在使用Go实现RPC服务时，由protobuf文件生成的桩代码中会自动附带 `omitempty` json tag。这导致在序列化结构体时，如果某个字段的值是零值，序列化后的字符串中就不会存在这个字段。当下游服务对反序列化要求比较严格时，就会出现字段缺失的问题。

例如在Python中直接访问这个字段：

```python
response = requests.post(url, headers=headers, data=json.dumps(data))
result = response.json()
value = result["key"]
```

会出现异常：

```
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
KeyError: 'key'
```

## 问题演示

**Go代码示例：**

```go
package main

import (
	"encoding/json"
	"fmt"
)

type data struct {
	X string `json:"x,omitempty"`
	Y string `json:"y"`
}

func main() {
	d0 := &data{
		X: "x",
		Y: "y",
	}
	d1 := &data{}
	sd0, _ := json.Marshal(d0)
	sd1, _ := json.Marshal(d1)
	fmt.Println(string(sd0), string(sd1))
}
```

**输出结果：**

```
{"x":"x","y":"y"} {"y":""}
```

由于 `data.Y` 使用了 `omitempty` json tag，导致当 `data.Y` 为空时，序列化后的字符串不会含有字段 `Y`。

## 解决方案

引入第三方SDK `jsoniter` 来解决这个问题：

```go
package main

import (
	"fmt"
	"unsafe"

	jsoniter "github.com/json-iterator/go"
	"github.com/modern-go/reflect2"
)

var json = jsoniter.ConfigCompatibleWithStandardLibrary

type data struct {
	X string `json:"x,omitempty"`
	Y string `json:"y"`
}

func main() {
	d0 := &data{
		X: "x",
		Y: "y",
	}
	d1 := &data{}
	sd0, _ := json.Marshal(d0)
	sd1, _ := json.Marshal(d1)
	fmt.Println(string(sd0), string(sd1))
}

type NotOmitemptyValEncoder struct {
	encoder jsoniter.ValEncoder
}

func (codec *NotOmitemptyValEncoder) Encode(ptr unsafe.Pointer, stream *jsoniter.Stream) {
	codec.encoder.Encode(ptr, stream)
}

func (codec *NotOmitemptyValEncoder) IsEmpty(ptr unsafe.Pointer) bool {
	return false
}

type NotOmitemptyEncoderExtension struct {
	jsoniter.DummyExtension
}

func (extension *NotOmitemptyEncoderExtension) DecorateEncoder(typ reflect2.Type, encoder jsoniter.ValEncoder) jsoniter.ValEncoder {
	return &NotOmitemptyValEncoder{encoder: encoder}
}

func init() {
	jsoniter.RegisterExtension(new(NotOmitemptyEncoderExtension))
}
```

**输出结果：**

```
{"x":"x","y":"y"} {"x":"","y":""}
```

可以看到符合预期，所有的字段都序列化出来了。
