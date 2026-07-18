---
title: "Triton Inference Server 框架实践"
date: 2025-03-21
draft: false
tags:
  - AI
  - Triton
categories:
  - AI Infra
---

> ⚠️ 本文完整内容在 CSDN 平台需付费解锁，以下为可见部分摘要。

## 项目背景

参与紧急项目，负责使用 TritonServer 框架对项目模块进行加速，主要通过 `dynamic_batch` 特性提升单卡吞吐量。

## 模块说明

- **模块类型**：`embedding` 模块
- **功能**：将文本表示成浮点数向量
- **输入**：`List[str]`
- **输出**：`List[List[float]]`
- **部署方式**：两种模型，通过输入字段决定使用哪个模型
- **当前线上版本**：FastAPI 封装，提供 HTTP 接口服务

## 技术选择

由于时间紧迫和服务协议限制，先使用 `python backend`，后续计划研究转换为 `tensorrt` 或 `onnx` 进行模型加速。

## config.pbtxt 配置示例

```protobuf
name: "embedding_model"
backend: "python"

input [
  {
    name: "texts"
    data_type: TYPE_STRING
    dims: [-1]
  }
]

output [
  {
    name: "prompt_tokens"
    data_type: TYPE_INT32
    dims: [1]
  },
  {
    name: "total_tokens"
    data_type: TYPE_INT32
    dims: [1]
  },
  {
    name: "embedding"
    data_type: TYPE_FP64
    dims: [-1, 768]
  }
]

dynamic_batching {
  preferred_batch_size: [4, 8]
  max_queue_delay_microseconds: 1000
}
```

## 重要提示

shape 默认包含第一维度 batch_size。虽然输入 texts 的 shape 是 -1，但实际输入应为二维数组。

---

> 📌 完整内容请访问 [CSDN 原文](https://blog.csdn.net/QSTARTmachine/article/details/146424801)
