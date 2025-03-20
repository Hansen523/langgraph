# LangGraph 平台

## 概述

LangGraph 平台是一个用于将智能体应用部署到生产环境的商业解决方案，基于开源的 [LangGraph 框架](./high_level.md) 构建。

LangGraph 平台由多个组件组成，这些组件协同工作，支持 LangGraph 应用的开发、部署、调试和监控：

- [LangGraph 服务器](./langgraph_server.md)：该服务器定义了一个观点明确的 API 和架构，结合了部署智能体应用的最佳实践，使您能够专注于构建智能体逻辑，而不是开发服务器基础设施。
- [LangGraph Studio](./langgraph_studio.md)：LangGraph Studio 是一个专用 IDE，可以连接到 LangGraph 服务器，以实现本地应用程序的可视化、交互和调试。
- [LangGraph CLI](./langgraph_cli.md)：LangGraph CLI 是一个命令行界面，帮助与本地 LangGraph 交互。
- [Python/JS SDK](./sdk.md)：Python/JS SDK 提供了一种编程方式与已部署的 LangGraph 应用程序进行交互。
- [远程图](../how-tos/use-remote-graph.md)：RemoteGraph 允许您与任何已部署的 LangGraph 应用程序进行交互，就好像它在本地运行一样。

![](img/lg_platform.png)

LangGraph 平台提供了一些不同的部署选项，详情请参阅 [部署选项指南](./deployment_options.md)。

## 为什么使用 LangGraph 平台？

**LangGraph 平台** 处理了将 LLM 应用部署到生产环境时常见的许多问题，使您能够专注于智能体逻辑，而不是管理服务器基础设施。

- **[流式支持](streaming.md)**：随着智能体变得越来越复杂，它们通常受益于将令牌输出和中间状态流式传输回用户。如果没有这个功能，用户将不得不在没有反馈的情况下等待可能很长的操作。LangGraph 服务器提供了 [多种流式模式](streaming.md)，针对各种应用需求进行了优化。

- **后台运行**：对于需要较长时间处理的智能体（例如，几个小时），保持一个开放的连接可能是不实际的。LangGraph 服务器支持在后台启动智能体运行，并提供轮询端点和 Webhook 来有效监控运行状态。

- **支持长时间运行**：普通的服务器设置在处理需要较长时间完成的请求时，经常会遇到超时或中断问题。LangGraph 服务器的 API 通过发送定期心跳信号，提供了对这些任务的强大支持，防止在长时间处理过程中意外关闭连接。

- **处理突发性**：某些应用程序，特别是那些有实时用户交互的应用程序，可能会遇到“突发性”请求负载，即大量请求同时到达服务器。LangGraph 服务器包括一个任务队列，确保在高负载下也能一致地处理请求而不会丢失。

- **[双重消息处理](double_texting.md)**：在用户驱动的应用程序中，用户快速发送多条消息是很常见的。这种“双重消息”如果处理不当，可能会中断智能体流程。LangGraph 服务器提供了内置策略来应对和管理此类交互。

- **[检查点和内存管理](persistence.md#checkpoints)**：对于需要持久化（例如，对话内存）的智能体，部署一个健壮的存储解决方案可能很复杂。LangGraph 平台包括优化的 [检查点](persistence.md#checkpoints) 和 [内存存储](persistence.md#memory-store)，跨会话管理状态，而无需定制解决方案。

- **[人在回路支持](human_in_the_loop.md)**：在许多应用程序中，用户需要一种干预智能体流程的方式。LangGraph 服务器提供了专门的端点用于人在回路场景，简化了将手动监督集成到智能体工作流中的过程。

通过使用 LangGraph 平台，您可以获得一个强大、可扩展的部署解决方案，减轻这些挑战，节省您手动实施和维护它们的精力。这使您能够更专注于构建有效的智能体行为，而不是解决部署基础设施问题。