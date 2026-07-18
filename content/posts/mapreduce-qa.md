---
title: "MapReduce Q&A"
date: 2023-09-02
draft: false
tags:
  - 技术
  - 分布式
categories:
  - 技术
---

1. Map 和 Reduce 可以相互转化吗？

    Worker做何种任务应该是按照 Coordinator分配到的type去区分

2. Coodinator 如何判断整个 Job 已经结束了？

    Reduce 任务已经完成

> 1. the coordinator, as an RPC server, will be concurrent
> 2. reduces can't start until the last map has finished
> 3. the coordinator can't reliably distinguish workers
> 4. using a temporary file and atomically renaming it once it is completely written, avoid to get a partial written file when crash



- Mapper 应该将中间结果分发个 N 个 Reducer

    Mapper 如何区分 Reducer？

    Mapper 不用区分 Reducer，只需要将中间结果写成文件，等待 Reducer 取文件

    取文件这个动作是 Coordinator做，然后通过rpc发送给Reducer，还是Reducer 直接读文件？

    怎么知道中间过程文件已经写完了？

    使用`ioutil.TempFile` 创建临时文件，然后在完成之后再重命名该文件

- job完成，worker就需要退出，那么worker如何知道job已经完成了？

    - worker向Coordinator要不到任务了，或者rpc连接失败了
    - worker拿到的任务是 “exist” 任务

- 中间过程文件按照 `mr-X-Y` 的格式给出 Mapper和Reducer的标识
