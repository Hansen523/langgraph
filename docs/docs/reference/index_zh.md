---
title: 参考文档
description: LangGraph API 参考手册
search:
  boost: 0.5
---

<style>
.md-sidebar {
  display: block !important;
}
</style>

# 参考文档

欢迎查阅 LangGraph 参考文档！本手册详细介绍了使用 LangGraph 构建应用程序时的核心接口。每个章节涵盖生态系统的不同组成部分。

!!! tip
    如果您是初次接触，请先阅读 [LangGraph 基础](../concepts/why-langgraph.md) 了解主要概念和使用模式。

## LangGraph 核心

开源 LangGraph 库的核心 API

- [图结构](graphs.md)：核心图抽象及使用方法
- [函数式 API](func.md)：图结构的函数式编程接口
- [Pregel](pregel.md)：Pregel 启发的计算模型
- [检查点](checkpoints.md)：保存和恢复图状态
- [存储](store.md)：存储后端与选项
- [缓存](cache.md)：性能优化缓存机制
- [类型](types.md)：图组件类型定义
- [配置](config.md)：配置选项
- [错误](errors.md)：错误类型与处理
- [常量](constants.md)：全局常量
- [信道](channels.md)：消息传递与信道

## 预制组件

针对常见工作流、智能体和其他模式的高级抽象

- [智能体](agents.md)：内置智能体模式
- [监督器](supervisor.md)：编排与委托
- [群组](swarm.md)：多智能体协作
- [MCP 适配器](mcp.md)：外部系统集成

## LangGraph 平台

部署和连接 LangGraph 平台的工具

- [命令行工具](../cloud/reference/cli.md)：用于构建和部署 LangGraph 平台应用的命令行接口
- [服务器 API](../cloud/reference/api/api_ref.md)：LangGraph 服务器的 REST API
- [SDK (Python)](../cloud/reference/sdk/python_sdk_ref.md)：与 LangGraph 服务器实例交互的 Python SDK
- [SDK (JS/TS)](../cloud/reference/sdk/js_ts_sdk_ref.md)：与 LangGraph 服务器实例交互的 JavaScript/TypeScript SDK
- [远程图](remote_graph.md)：连接 LangGraph 服务器实例的 `Pregel` 抽象
- [环境变量](../cloud/reference/env_var.md)：使用 LangGraph 平台部署时支持的配置变量