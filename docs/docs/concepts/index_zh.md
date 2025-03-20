---
title: 概念
description: LangGraph 的概念指南
---

# 概念指南

本指南提供了关于 LangGraph 框架及其背后关键概念的解释，同时也涵盖了更广泛的 AI 应用领域。

我们建议您在深入阅读本概念指南之前，至少先完成[快速入门](../tutorials/introduction.ipynb)。这将为您提供实际的操作背景，使您更容易理解这里讨论的概念。

概念指南不包含逐步的操作说明或具体的实现示例——这些内容可以在[教程](../tutorials/index.md)和[操作指南](../how-tos/index.md)中找到。如需详细的参考材料，请参阅[API 参考](../reference/index.md)。

## LangGraph

### 高层概述

- [为什么选择 LangGraph？](high_level.md)：LangGraph 及其目标的高层概述。

### 概念

- [LangGraph 术语表](low_level.md)：LangGraph 工作流被设计为图，其中节点代表不同的组件，边代表信息在它们之间的流动。本指南概述了与 LangGraph 图原语相关的关键概念。
- [常见的代理模式](agentic_concepts.md)：代理使用 LLM 来选择自己的控制流，以解决更复杂的问题！代理是许多 LLM 应用的关键构建块。本指南解释了不同类型的代理架构以及如何用它们来控制应用的流程。
- [多代理系统](multi_agent.md)：复杂的 LLM 应用通常可以分解为多个代理，每个代理负责应用的不同部分。本指南解释了构建多代理系统的常见模式。
- [断点](breakpoints.md)：断点允许在特定点暂停图的执行。断点允许逐步执行图，以便进行调试。
- [人在回路中](human_in_the_loop.md)：解释了将人类反馈集成到 LangGraph 应用中的不同方式。
- [时间旅行](time-travel.md)：时间旅行允许您重放 LangGraph 应用中的过去操作，以探索替代路径并调试问题。
- [持久化](persistence.md)：LangGraph 通过检查点实现了一个内置的持久化层。这个持久化层有助于支持诸如人在回路中、记忆、时间旅行和容错等强大功能。
- [记忆](memory.md)：AI 应用中的记忆指的是处理、存储和有效回忆过去交互信息的能力。有了记忆，您的代理可以从反馈中学习并适应用户的偏好。
- [流式处理](streaming.md)：流式处理对于增强基于 LLM 的应用程序的响应能力至关重要。通过逐步显示输出，即使在完整的响应准备好之前，流式处理也能显著改善用户体验（UX），特别是在处理 LLM 的延迟时。
- [功能 API](functional_api.md)：`@entrypoint` 和 `@task` 装饰器允许您将 LangGraph 功能添加到现有代码库中。
- [持久执行](durable_execution.md)：LangGraph 的内置[持久化](./persistence.md)层为工作流提供了持久执行，确保每个执行步骤的状态都保存到持久存储中。
- [Pregel](pregel.md)：Pregel 是 LangGraph 的运行时，负责管理 LangGraph 应用程序的执行。
- [常见问题](faq.md)：关于 LangGraph 的常见问题。

## LangGraph 平台

LangGraph 平台是一个用于在生产环境中部署代理应用的商业解决方案，基于开源的 LangGraph 框架构建。

LangGraph 平台提供了几种不同的部署选项，详见[部署选项指南](./deployment_options.md)。

!!! tip

    * LangGraph 是一个 MIT 许可的开源库，我们致力于为社区维护和发展它。
    * 您始终可以使用开源的 LangGraph 项目在自己的基础设施上部署 LangGraph 应用程序，而无需使用 LangGraph 平台。

### 高层概述

- [为什么选择 LangGraph 平台？](./langgraph_platform.md)：LangGraph 平台是一种有主见的部署和管理 LangGraph 应用的方式。本指南概述了 LangGraph 平台的关键功能和概念。
- [平台架构](./platform_architecture.md)：LangGraph 平台架构的高层概述。
- [可扩展性和弹性](./scalability_and_resilience.md)：LangGraph 平台设计为可扩展且弹性的。本文档解释了平台如何实现这一点。
- [部署选项](./deployment_options.md)：LangGraph 平台提供了四种部署选项：[自托管精简版](./self_hosted.md#self-hosted-lite)、[自托管企业版](./self_hosted.md#self-hosted-enterprise)、[自带云（BYOC）](./bring_your_own_cloud.md) 和 [云 SaaS](./langgraph_cloud.md)。本指南解释了这些选项之间的区别，以及哪些计划支持它们。
- [计划](./plans.md)：LangGraph 平台提供三种不同的计划：开发者版、增强版、企业版。本指南解释了这些选项之间的区别，每种选项可用的部署选项，以及如何注册每种计划。
- [模板应用](./template_applications.md)：参考应用，旨在帮助您在使用 LangGraph 时快速上手。

### 组件

LangGraph 平台由多个组件组成，这些组件共同支持 LangGraph 应用的部署和管理：

- [LangGraph 服务器](./langgraph_server.md)：LangGraph 服务器支持广泛的代理应用用例，从后台处理到实时交互。
- [LangGraph Studio](./langgraph_studio.md)：LangGraph Studio 是一个专门的 IDE，可以连接到 LangGraph 服务器，以便在本地实现应用的的可视化、交互和调试。
- [LangGraph CLI](./langgraph_cli.md)：LangGraph CLI 是一个命令行界面，用于与本地 LangGraph 进行交互。
- [Python/JS SDK](./sdk.md)：Python/JS SDK 提供了一种编程方式与部署的 LangGraph 应用进行交互。
- [远程图](../how-tos/use-remote-graph.md)：RemoteGraph 允许您与任何部署的 LangGraph 应用进行交互，就像它在本地运行一样。

### LangGraph 服务器

- [应用结构](./application_structure.md)：LangGraph 应用由一个或多个图、一个 LangGraph API 配置文件（`langgraph.json`）、一个指定依赖项的文件以及环境变量组成。
- [助手](./assistants.md)：助手是一种保存和管理 LangGraph 应用不同配置的方式。
- [Webhooks](./langgraph_server.md#webhooks)：Webhooks 允许运行的 LangGraph 应用在特定事件发生时将数据发送到外部服务。
- [定时任务](./langgraph_server.md#cron-jobs)：定时任务是一种在 LangGraph 应用中安排任务在特定时间运行的方式。
- [双重消息发送](./double_texting.md)：双重消息发送是 LLM 应用中的一个常见问题，用户可能在图完成运行之前发送多条消息。本指南解释了如何使用 LangGraph 部署处理双重消息发送。
- [认证与访问控制](./auth.md)：了解部署 LangGraph 平台时的认证和访问控制选项。

### 部署选项

- [自托管精简版](./self_hosted.md)：LangGraph 平台的免费版本（每年最多执行 100 万个节点），您可以本地运行或以自托管方式运行。
- [云 SaaS](./langgraph_cloud.md)：作为 LangSmith 的一部分托管。
- [自带云](./bring_your_own_cloud.md)：我们管理基础设施，因此您无需操心，但所有基础设施都在您的云中运行。
- [自托管企业版](./self_hosted.md)：完全由您管理。