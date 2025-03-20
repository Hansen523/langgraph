# LangGraph 服务器

!!! info "前提条件"
    - [LangGraph 平台](./langgraph_platform.md)
    - [LangGraph 术语表](low_level.md)

## 概述

LangGraph 服务器提供了一个用于创建和管理基于代理的应用程序的 API。它建立在[助手](assistants.md)的概念之上，这些助手是为特定任务配置的代理，并内置了[持久化](persistence.md#memory-store)和**任务队列**。这个多功能的 API 支持广泛的代理应用程序用例，从后台处理到实时交互。

## 主要特性

LangGraph 平台集成了代理部署的最佳实践，因此您可以专注于构建代理逻辑。

* **流式端点**：暴露[多种不同流式模式](streaming.md)的端点。我们使这些端点即使在长时间运行的代理中也能工作，这些代理可能在连续流事件之间间隔几分钟。
* **后台运行**：LangGraph 服务器支持通过端点启动后台助手，轮询助手运行状态，并通过 webhook 有效监控运行状态。
- **支持长时间运行**：我们用于运行助手的阻塞端点会发送定期心跳信号，防止在处理需要长时间完成的请求时意外关闭连接。
* **任务队列**：我们添加了任务队列，以确保在处理突发请求时不会丢失任何请求。
* **水平可扩展的基础架构**：LangGraph 服务器设计为水平可扩展，允许您根据需要扩展使用规模。
* **双文本支持**：很多时候，用户可能会以意想不到的方式与您的图形交互。例如，用户可能发送一条消息，在图形完成运行之前发送第二条消息。我们称之为["双文本"](double_texting.md)，并添加了四种不同的处理方式。
* **优化的检查点**：LangGraph 平台内置了一个专为 LangGraph 应用程序优化的[检查点](./persistence.md#checkpoints)。
* **人在回路中的端点**：我们暴露了支持[人在回路中](human_in_the_loop.md)功能所需的所有端点。
* **内存**：除了线程级持久化（由[检查点](./persistence.md#checkpoints)涵盖）之外，LangGraph 平台还内置了一个[内存存储](persistence.md#memory-store)。
* **定时任务**：内置支持调度任务，使您能够自动化应用程序中的常规操作，如数据清理或批处理。
* **Webhooks**：允许您的应用程序向外部系统发送实时通知和数据更新，便于与第三方服务集成，并根据特定事件触发操作。
* **监控**：LangGraph 服务器与[LangSmith](https://docs.smith.langchain.com/) 监控平台无缝集成，提供应用程序性能和健康的实时洞察。

## 您部署了什么？

当您部署 LangGraph 服务器时，您正在部署一个或多个[图形](#graphs)、用于[持久化](persistence.md)的数据库和一个任务队列。

### 图形

当您使用 LangGraph 服务器部署图形时，您正在部署一个[助手](assistants.md)的“蓝图”。

一个[助手](assistants.md)是一个图形加上该图形的特定[配置](low_level.md#configuration)设置。您可以为每个图形创建多个助手，每个助手都有独特的设置，以适应不同用例，这些用例可以由同一个图形提供支持。

部署后，LangGraph 服务器将自动为每个图形创建一个默认助手，使用图形的默认配置设置。

您可以通过 [LangGraph 服务器 API](#langgraph-server-api) 与助手交互。

!!! note

    我们通常认为一个图形实现了一个[代理](agentic_concepts.md)，但一个图形不一定需要实现一个代理。例如，一个图形可以实现一个简单的聊天机器人，只支持来回对话，而没有能力影响任何应用程序控制流。实际上，随着应用程序变得复杂，一个图形通常会实现一个更复杂的流程，可能使用[多个代理](./multi_agent.md)协同工作。

### 持久化和任务队列

LangGraph 服务器利用数据库进行[持久化](persistence.md)和任务队列。

目前，LangGraph 服务器仅支持 [Postgres](https://www.postgresql.org/) 作为数据库，[Redis](https://redis.io/) 作为任务队列。

如果您使用 [LangGraph 云](./langgraph_cloud.md) 进行部署，这些组件将由我们管理。如果您在自己的基础设施上部署 LangGraph 服务器，您需要自己设置和管理这些组件。

请查看 [部署选项](./deployment_options.md) 指南，了解这些组件的设置和管理方式。

## 应用程序结构

要部署 LangGraph 服务器应用程序，您需要指定要部署的图形，以及任何相关的配置设置，如依赖项和环境变量。

阅读 [应用程序结构](./application_structure.md) 指南，了解如何为部署构建您的 LangGraph 应用程序。

## LangGraph 服务器 API

LangGraph 服务器 API 允许您创建和管理[助手](assistants.md)、[线程](#threads)、[运行](#runs)、[定时任务](#cron-jobs)等。

[LangGraph 云 API 参考](../cloud/reference/api/api_ref.html) 提供了有关 API 端点和数据模型的详细信息。

### 助手

一个[助手](assistants.md)指的是一个[图形](#graphs)加上该图形的特定[配置](low_level.md#configuration)设置。

您可以将助手视为一个[代理](agentic_concepts.md)的保存配置。

在构建代理时，通常会进行快速更改，这些更改*不*改变图形逻辑。例如，仅更改提示或 LLM 选择可能会对代理的行为产生重大影响。助手提供了一种简单的方法来保存这些类型的代理配置更改。

### 线程

一个线程包含一系列[运行](#runs)的累积状态。如果在线程上执行运行，则助手底层图形的[状态](low_level.md#state)将持久化到线程中。

可以检索线程的当前和历史状态。为了持久化状态，必须在执行运行之前创建线程。

线程在特定时间点的状态称为[检查点](persistence.md#checkpoints)。检查点可用于在以后恢复线程的状态。

有关线程和检查点的更多信息，请参见 [LangGraph 概念指南](low_level.md#persistence) 的这一部分。

LangGraph 云 API 提供了多个用于创建和管理线程和线程状态的端点。请参见 [API 参考](../cloud/reference/api/api_ref.html#tag/threads) 了解更多细节。

### 运行

一次运行是对[助手](#assistants)的调用。每次运行可能有自己的输入、配置和元数据，这可能会影响底层图形的执行和输出。运行可以选择在[线程](#threads)上执行。

LangGraph 云 API 提供了多个用于创建和管理运行的端点。请参见 [API 参考](../cloud/reference/api/api_ref.html#tag/thread-runs/) 了解更多细节。

### 存储

存储是一个用于管理持久[键值存储](./persistence.md#memory-store)的 API，可从任何[线程](#threads)访问。

存储在您的 LangGraph 应用程序中实现[内存](./memory.md)时非常有用。

### 定时任务

在许多情况下，按计划运行助手是很有用的。

例如，假设您正在构建一个每天运行并通过电子邮件发送当天新闻摘要的助手。您可以使用定时任务每天下午 8:00 运行助手。

LangGraph 云支持定时任务，这些任务按用户定义的计划运行。用户指定一个计划、一个助手和一些输入。之后，在指定的计划上，服务器将：

- 创建一个具有指定助手的新线程
- 将指定的输入发送到该线程

请注意，这每次都会将相同的输入发送到线程。请参见创建定时任务的 [操作指南](../cloud/how-tos/cron_jobs.md)。

LangGraph 云 API 提供了多个用于创建和管理定时任务的端点。请参见 [API 参考](../cloud/reference/api/api_ref.html#tag/runscreate/POST/threads/{thread_id}/runs/crons) 了解更多细节。

### Webhooks

Webhooks 使您的 LangGraph 云应用程序能够与外部服务进行事件驱动通信。例如，您可能希望在完成对 LangGraph 云的 API 调用后向单独的服务发出更新。

许多 LangGraph 云端点接受 `webhook` 参数。如果此参数由可以接受 POST 请求的端点指定，LangGraph 云将在运行完成时发送请求。

请参见相应的 [操作指南](../cloud/how-tos/webhooks.md) 了解更多细节。

## 相关

* LangGraph [应用程序结构](./application_structure.md) 指南解释了如何为部署构建您的 LangGraph 应用程序。
* [LangGraph 平台的操作指南](../how-tos/index.md)。
* [LangGraph 云 API 参考](../cloud/reference/api/api_ref.html) 提供了有关 API 端点和数据模型的详细信息。