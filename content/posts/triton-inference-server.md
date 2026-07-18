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

最近，我参与了一个紧急项目，负责将项目中的一个模块利用 TritonServer 框架进行加速，主要通过其 `dynamic_batch` 特性提升单卡吞吐量。经过一段时间的努力，我在今晚值班时整理了相关经验，与大家分享。

## 背景介绍

该模块是 `embedding` 模块，用于将文本表示成浮点数向量，输入为 `List[str]`，输出为 `List[List[float]]`。服务部署了两种模型，通过输入字段决定使用哪个模型推理。目前线上版本使用 FastAPI 封装，提供 HTTP 接口服务。

由于时间紧迫，且服务协议不能改变，还需要对模型输出进行后处理，而我对 `torch` 转 `onnx` 模型不够熟练，因此先直接使用 `python backend`，后续再研究如何转 `tensorrt` 或 `onnx` 进行模型加速。

## 使用方法

### 基本用法

首先明确模型的输入输出，然后输出 `config.pbtxt` 文件。该模型较为简单，配置如下：

```protobuf
name: "model_name"
backend: "python"
input [
  {
    name: "texts"
    data_type: TYPE_STRING
    dims: [ -1 ]
  }
]
output [
  {
    name: "prompt_tokens"
    data_type: TYPE_INT32
    dims: [ 1 ]
  },
  {
    name: "total_tokens"
    data_type: TYPE_INT32
    dims: [ 1 ]
  },
  {
    name: "embedding"
    data_type: TYPE_FP64
    dims: [ -1, 768 ]
  }
]
dynamic_batching {
  preferred_batch_size: [ 4, 8 ]
  max_queue_delay_microseconds: 1000
}
```

需要注意的是，`shape` 默认包含第一维度 `batch_size`。虽然输入 `texts` 的 shape 是 -1，但输入应为二维，例如：

```python
texts = ["hello", "world"]
data = {
    "id": str(uuid.uuid4()),
    "inputs": [
        {
            "name": "texts",
            "shape": [1, len(texts)],
            "datatype": "BYTES",
            "data": [[texts]],
        }
    ],
}
```

实际上，将输入改为 `[batch_size, 1]` 更合理，因为动态尺寸可能导致 Triton 组 batch 时，真实 batch 数失真。但测试发现，使用 `[batch_size, 1]` 会导致吞吐量下降，推测是因为 Triton 更容易组成小 batch，未能充分发挥 `dynamic_batch` 特性。

部署时，建议基于 NVIDIA 官方镜像制作服务运行镜像，基于源码编译安装通常不太顺利。

### 部署参数配置化

由于集群中存在多种 GPU，性能和显存各异，为了充分利用算力资源，需针对不同配置的资源实现不同的部署配置参数。常见的影响算力利用的部署参数包括 `instance_group`、`max_batch_size` 和 `preferred_batch_size`。

例如，在 CPU 机器上使用以下配置：

```protobuf
max_batch_size: 8

instance_group [
  {
    count: 1
    kind: KIND_CPU
  }
]

dynamic_batching {
  preferred_batch_size: [ 4, 8 ]
  max_queue_delay_microseconds: 1000
}
```

在 A10 机器上使用以下配置：

```protobuf
max_batch_size: 8

instance_group [
  {
    count: 2
    kind: KIND_GPU
  }
]

dynamic_batching {
  preferred_batch_size: [ 4, 8 ]
  max_queue_delay_microseconds: 5000
}
```

这些值需通过压测评估吞吐量和耗时，以确定较优配置。NVIDIA 提供了 `perf_analyzer` 工具，可通过镜像 `nvcr.io/nvidia/tritonserver:<tag>-sdk` 获得。

部署时，通过 `--model-config-name` 指定配置文件名，具体可参考[模型配置文档](https://github.com/triton-inference-server/server/blob/main/docs/user_guide/model_configuration.md#custom-model-configuration)。

但在使用过程中，我遇到了 TritonServer 错误：

```
{"error":"[request id: xxx] expected 0 inputs but got 1 inputs for model 'thenlper_gte-base-zh'. Got input(s) ['texts'], but missing required input(s) ]. Please provide all required input(s)."}
```

查询模型元数据发现，模型的输入输出为空，配置文件未生效。仔细对比文档后发现，配置文件路径有要求：

- 每个模型目录下必须有默认的 `config.pbtxt`。
- 其他配置需放在 `./configs` 目录下。

调整配置文件目录后，问题解决。

### 模型预热

通常，模型需要预热后才能正式上线，否则前几个请求推理较慢。以往我们通过代码预热，但在 TritonServer 中不太方便，且使用 Triton Client 预热会导致线上遥测和监控出现脏数据。TritonServer 自带的模型预热特性解决了这一问题。

具体使用案例可参考[预热测试脚本](https://github.com/triton-inference-server/server/blob/main/qa/L0_warmup/test.sh)，示例如下：

```protobuf
model_warmup [
  {
    name: "warmup_zero_data"
    batch_size: 1
    inputs: {
      key: "texts"
      value: {
        data_type: TYPE_STRING
        dims: [256]
        zero_data: true
      }
    }
  }
]
```

与使用 Triton Client 预热相比，模型内部预热更快，且当 `instance_group: kind > 1` 时，能确保预热到所有实例，避免脏数据。

### 模型缓存

TritonServer 可根据输入计算哈希值，作为唯一标识缓存响应。以 `local` 缓存为例，启动时通过 `--cache-config local,size=104857600` 指定缓存介质，然后在 `config.pbtxt` 中声明启用缓存：

```protobuf
response_cache {
  enable: true
}
```

但由于这里使用的是语言模型，缓存命中率较低，因此仅进行了简单调研，未实际应用。

### 监控

TritonServer 提供了方便的内置监控接口，数据结构可直接接入 Prometheus，详细指南可参考[监控文档](https://github.com/triton-inference-server/server/blob/6afaed7c4f0dbd83d59627a7f546baaf527fb856/docs/user_guide/metrics.md#L4)。

## 其他问题

改造完成后，在独立云主机上测试发现，8 并发下吞吐量提升了 3 倍。但线上部署后，提升仅为 40%。通过控制变量法排除了 CPU 型号和核心数的影响后，经同事建议，尝试比较集群和云主机上的基本矩阵运算性能，发现集群上的矩阵运算速度比云主机慢了 4 倍。

推测可能是 NVIDIA 驱动与 CUDA 不兼容导致的，高版本 CUDA 在低版本驱动上会使用兼容模式。于是，我尝试降低 TritonServer 基础镜像版本至 `22.01-py3`，将 CUDA 版本降至 `11.6`，但矩阵计算性能仍较低。随后，我从 SRE 同学那里获取了一台高 NVIDIA 驱动版本的机器，使用 `24.06-py3` TritonServer 基础镜像，测试发现矩阵计算性能提高了 4 倍，问题得以解决。后续需协调算力平台人员升级驱动版本以提升 GPU 利用率。

## 总结

TritonServer 非常适合用于提升线上服务质量，其 `dynamic_batch` 特性可提升吞吐量，多模型实例可提升算力利用率，模型预热可提升服务稳定性，内置监控则便于开发人员运维。
