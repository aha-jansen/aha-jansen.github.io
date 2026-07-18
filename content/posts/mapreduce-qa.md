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



3. Mapper 如何区分 Reducer？

    Mapper 不用区分 Reducer，只需要将中间结果写成文件，等待 Reducer 取文件

4. 取文件这个动作是 Coordinator做，然后通过rpc发送给Reducer，还是Reducer 直接读文件？怎么知道中间过程文件已经写完了？

    Coordinator 只负责“指路”，它会跟踪所有 Mapper 的状态，当 Mapper 任务完成后，Coordinator 会记录下：“Mapper 1 已经完成，它的中间文件存放在节点 A 的 `/tmp/map1.out`”。

    Reducer 启动后，会向 Coordinator 询问：“哪些 Mapper 已经干完了？文件在哪？”。拿到列表后，Reducer 会通过网络连接到存放中间文件的节点，主动读取（拉取）数据。

5. 使用`ioutil.TempFile` 创建临时文件，然后在完成之后再重命名该文件，避免文件读写导致的冲突。

6. job完成，worker就需要退出，那么worker如何知道job已经完成了？
    - worker向Coordinator要不到任务了，或者rpc连接失败了
    - worker拿到的任务是 “exist” 任务

7. 中间过程文件按照 `mr-X-Y` 的格式给出 Mapper和Reducer的标识，让 Reducer 可以自主找到对应的中间结果，方便 Reduce 进行归并排序。