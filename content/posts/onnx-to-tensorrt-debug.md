---
title: "记录一次复杂的 ONNX 到 TensorRT 动态 Shape 转换排错过程"
date: 2026-01-09
draft: false
tags:
  - AI
  - TensorRT
  - ONNX
categories:
  - AI Infra
---

在将 encoder 的 ONNX 模型转换成 TensorRT 格式时遇到了错误："shape tensor must have build-time extent"。从报错信息看，ONNX 的 Range 算子在转换时被视为 shape tensor，而 TensorRT 要求 shape tensor 在 build 时维度必须是已知常量。

## 问题诊断

通过 Netron.app 可视化发现，Range 算子的 limit 参数依赖于上一个 Cast 算子的输出。为了进一步排查，向 ONNX 图中添加了额外的输出 tensor，打印 Range 算子上下游的输入和输出。

**诊断代码示例：**
```python
def inspect_all_tensors(onnx_path, target_tensor_name, input_feed):
    model = onnx.load(onnx_path)
    inferred_model = onnx.shape_inference.infer_shapes(model)
    all_tensors = (list(inferred_model.graph.value_info) + 
                   list(inferred_model.graph.input) + 
                   list(inferred_model.graph.output))
    for t in all_tensors:
        if t.name in target_tensor_name:
            model.graph.output.append(t)
    
    modified_model_bytes = model.SerializeToString()
    sess = onnxruntime.InferenceSession(modified_model_bytes)
    output_names = [output.name for output in sess.get_outputs()]
    outputs = sess.run(output_names, input_feed)

    results = {name: value for name, value in zip(output_names, outputs)}
    
    for name, value in results.items():
        print(f"{name}: {value.shape} {value.dtype} {value}")
```

**打印结果：**
```
onnx::ReduceSum_851: (1,) int64 [1]
onnx::Unsqueeze_953: (1,) int64 [0]
/ReduceMax_output_0: () int64 13
/Range_output_0: (13,) int64 [ 0  1  2  3  4  5  6  7  8  9 10 11 12]
/Unsqueeze_output_0: (1, 13) int64 [[ 0  1  2  3  4  5  6  7  8  9 10 11 12]]
/Where_output_0: (2,) int64 [ 1 13]
/Expand_output_0: (1, 13) int64 [[ 0  1  2  3  4  5  6  7  8  9 10 11 12]]
```

## 根本原因分析

Range 输出长度达 13，考虑到后续图中不太可能存在如此高维的 tensor，错误日志的含义是：

> 这里试图让一个数据张量（Range 的输出）的长度（extent）依赖于一个形状张量，而 TensorRT 在构建时无法将这个依赖关系静态化。

**核心问题：** TensorRT 可以处理动态形状，但有一个前提——所有用来决定张量形状的计算，必须只依赖于输入的【形状】，而不能依赖于输入的【内容/值】。

## 第一次尝试：使用 TopK 替换 ReduceMax

尝试用 TopK 算子代替 ReduceMax，使其生成一个 shape tensor 而非 data tensor，但仍然出现错误：

```
Error[9]: [graph.cpp::computeInputExecutionUses::553] Error Code 9: Internal Error 
(/TopK_for_/ReduceMax: ITopKLayer cannot be used to compute a shape tensor)
```

TopK 的输出依然不能作为 shape tensor。

## 第二次尝试：替换动态输入

根源在于输入设置不合理。输入 `x_lens` 是一个一维动态 tensor，表示这个 batch 的输入长度。直接将输入改为该 mask tensor，可以避免上述错误：

```python
def solve_replace_dynamic_range_with_mask_input(onnx_path):
    model = onnx.load(onnx_path)
    ir_version = model.ir_version
    graph = gs.import_onnx(model)
    x_len_mask = gs.Variable(name=f"x_len_mask", dtype=np.bool_, shape=['N', 'L'])
    for node in graph.nodes:
        for i, input_t in enumerate(node.inputs):
            if input_t.name == "/GreaterOrEqual_output_0":
                print(f"found node : {node}")
                node.inputs[i] = x_len_mask
    inputs_to_delete_set = ["x_lens"]
    outputs_to_delete_set = ["encoder_out_lens"]
    graph.inputs = [inp for inp in graph.inputs if inp.name not in inputs_to_delete_set]
    graph.outputs = [out for out in graph.outputs if out.name not in outputs_to_delete_set]
    graph.inputs.append(x_len_mask)
    graph.cleanup()
    graph.toposort()
    onnx.save(gs.export_onnx(graph, ir_version=ir_version), "./onnx/encoder_mask_input_solved.onnx")
```

但随后出现新的问题：

```
Error Code 2: Internal Error (/encoder/Slice_2 requires bool I/O but node can not be handled by Myelin. 
Dynamic shapes are not equal for slice params.)
```

## 第三次尝试：使用 Range + Gather 替换 Slice

需要把 Slice 节点也替换掉。使用 `Range + Gather` 来替换 Slice 节点，但 Range 节点的 limit 参数不支持 INF，只能通过 Shape 以及 Gather 算子获取。

继续转换后出现新的错误：

```
[12/13/2025-17:14:45] [E] Error[2]: [shapeContext.cpp::checkVolume::2379] Error Code 2: Internal Error 
(Assertion bound >= 0 failed.)
```

## 版本升级尝试

怀疑是 TensorRT 的版本太低，导致 dynamic shape 支持得不好。升级了 CUDA 驱动至 12.4，使用基于 TensorRT 10.0 的基础镜像。

**Dockerfile 配置：**
```dockerfile
# nVidia TensorRT Base Image
# tensorrt 10.x support more onnx ops with dynamic shape 
ARG TRT_CONTAINER_VERSION=24.05
FROM nvcr.io/nvidia/tensorrt:${TRT_CONTAINER_VERSION}-py3

ARG ONNXRUNTIME_REPO=https://github.com/Microsoft/onnxruntime 
ARG ONNXRUNTIME_BRANCH=main
ARG CMAKE_CUDA_ARCHITECTURES=75

RUN apt-get update &&\
    apt-get install -y sudo git bash unattended-upgrades
RUN unattended-upgrade

WORKDIR /code
ENV PATH=/usr/local/nvidia/bin:/usr/local/cuda/bin:/code/cmake-3.31.5-linux-x86_64/bin:${PATH}

# Prepare onnxruntime repository & build onnxruntime with TensorRT
RUN git clone --single-branch --branch ${ONNXRUNTIME_BRANCH} --recursive ${ONNXRUNTIME_REPO} onnxruntime &&\
    /bin/sh onnxruntime/dockerfiles/scripts/install_common_deps.sh &&\
    trt_version=${TRT_VERSION:0:4} &&\
    cd onnxruntime &&\
    /bin/sh build.sh --allow_running_as_root --parallel --build_shared_lib --cuda_home /usr/local/cuda --cudnn_home /usr/lib/x86_64-linux-gnu/ --use_tensorrt --tensorrt_home /usr/lib/x86_64-linux-gnu/ --config Release --build_wheel --skip_tests --skip_submodule_sync --cmake_extra_defines '"CMAKE_CUDA_ARCHITECTURES='${CMAKE_CUDA_ARCHITECTURES}'"' &&\
    pip install /code/onnxruntime/build/Linux/Release/dist/*.whl &&\
    cd ..
```

## 后续问题

使用最原始的 ONNX 模型进行转换，出现新的问题：

```
[01/06/2026-12:05:53] [E] Error[4]: [graphShapeAnalyzer.cpp::analyzeShapes::2084] Error Code 4: Miscellaneous 
(IConditionalOutputLayer /encoder/1/encoder/encoder_pos/If_OutputLayer: dimensions not compatible for if-conditional outputs)
```

通过 Netron.app 发现这个节点的 then 分支输出是空的，估计是之前用 onnxoptimizer 给优化掉了。

## 最终问题

继续转换后出现错误：

```
[01/06/2026-12:19:30] [E] [TRT] ModelImporter.cpp:836: ERROR: ModelImporter.cpp:194 In function parseNode:
[6] Invalid Node - /encoder/1/downsample/Softmax
```

原因在于 TensorRT 要求 Softmax 算子的输入 Tensor 必须是二维及以上，第一维表示 batch_size。这里的输入其实是常量 bias，在 ONNX 中是 initializer。

## 核心难点

ONNX 模型中存在一个 If 算子，它的 else_branch 和 then_branch 子图输出 shape 不一致，导致问题。其中 then_branch 输出的是固定 size 的 tensor，但 else_branch 输出的是 dynamic size 的 tensor。这个问题比较棘手，因为 else_branch 的输出 size 有可能大于 then_branch 的输出 size，无法通过 padding 到固定 size 解决。

## 总结

本次尝试暂时告一段落。排错过程中遇到的主要问题包括：

- Shape tensor 与 data tensor 的混淆
- 动态形状依赖于数据内容而非形状信息
- TensorRT 版本对动态形状支持的限制
- If 条件分支输出 shape 不一致的兼容性问题
