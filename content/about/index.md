+++
date = '2026-07-18T15:05:31+08:00'
draft = false
title = 'About'
layout = 'single'
+++

## 关于我

蒋帅（Jansen Jiang），腾讯 Go 后端开发工程师，2022 年 7 月加入腾讯，目前是 CodeBuddy 团队成员、WorkBuddy 项目负责人。

### 教育背景

- **中山大学** · 硕士 · 信息与通信工程（2019 - 2022）
- **河北工业大学** · 学士 · 电子与信息工程（2015 - 2019）

### 项目经历

**AI PaaS 建设**（Go / Python / C++）

- 将 Embedding 推理服务从 FastAPI 迁移至 Triton Inference Server，通过 dynamic batching 与 warm_up 预热消除冷启动，算力资源消耗降低 20%+
- 针对 TTS 流式推理链路，引入异步读写缓冲区将音频生成与数据传输并行化，首包耗时降低 200ms
- 通过封装 `io.Reader` 在流式传输层透明拦截数据，以非侵入方式实现 ChatGPT 代理服务的访问记录落库与计费

**混元助手项目孵化**（Go）

- 优化服务链路生命周期管理，实现服务终止时下游资源的即时释放
- 实现异步流式内容检测，消除检测服务对推理链路的阻塞
- 基于滑动窗口在输出端实现流式引用插入，避免 URL 占用推理 token

**多模态数据研效平台**（Go / Python）

- 主导平台整体架构设计，构建基于 DAG 的算子调度引擎
- 引入 DuckDB 替代传统关系型方案，实现数据集高效索引与多维聚合查询
- 基于 Starlark 设计统一数据格式转换层，以适配器模式降低新业务接入成本

### 专业技能

- 熟悉数据结构与算法，LeetCode 常年 Guardian 段位
- 掌握计算机网络、操作系统等计算机专业基础知识
- 熟悉 Go 语言，掌握 Context、goroutine 等底层原理，熟悉并发编程与微服务治理
- 熟悉 Redis 在缓存、分布式锁及消息队列场景下的工程实践
- 熟悉 Python 语言及 AI 推理服务生态，具备 Triton Inference Server 部署与性能调优经验

### 个人荣誉

- 腾讯技术突破奖（混元大模型联合项目团队）— 2023 年 12 月
- TPC 腾讯程序设计竞赛前 100 — 2022 年 11 月

### 联系方式

- GitHub: [@aha-jansen](https://github.com/aha-jansen)
- Blog: [CSDN](https://blog.csdn.net/QSTARTmachine)
- Resume: [aha-jansen/resume](https://github.com/aha-jansen/resume)
