# 常见问题解答

常见问题及其答案！

## 我需要使用LangChain才能使用LangGraph吗？两者有什么区别？

不需要。LangGraph是一个用于复杂代理系统的编排框架，比LangChain代理更底层且可控。LangChain提供了一个标准接口，用于与模型和其他组件交互，适用于直接的链和检索流程。

## LangGraph与其他代理框架有什么不同？

其他代理框架可以处理简单、通用的任务，但在处理公司需求的复杂任务时表现不足。LangGraph提供了一个更具表现力的框架，能够处理公司独特的任务，而不限制用户使用单一的黑箱认知架构。

## LangGraph会影响我的应用程序性能吗？

LangGraph不会给你的代码增加任何开销，并且是专门为流式工作流设计的。

## LangGraph是开源的吗？它是免费的吗？

是的。LangGraph是一个MIT许可的开源库，可以免费使用。

## LangGraph和LangGraph平台有什么区别？

LangGraph是一个有状态的编排框架，为代理工作流增加了额外的控制。LangGraph平台是一个用于部署和扩展LangGraph应用程序的服务，提供了一个用于构建代理用户体验的API，以及一个集成的开发者工作室。

| 功能                | LangGraph（开源）                                      | LangGraph平台                                                                                     |
|---------------------|--------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| 描述                | 用于代理应用程序的有状态编排框架                       | 用于部署LangGraph应用程序的可扩展基础设施                                                         |
| SDK                 | Python和JavaScript                                    | Python和JavaScript                                                                                |
| HTTP API            | 无                                                     | 有 - 用于检索和更新状态或长期记忆，或创建可配置的助手                                             |
| 流式处理            | 基本                                                   | 专用模式，支持逐令牌消息                                                                         |
| 检查点              | 社区贡献                                               | 开箱即用支持                                                                                     |
| 持久层              | 自管理                                                 | 管理的Postgres，具有高效存储                                                                     |
| 部署                | 自管理                                                 | • 云SaaS <br> • 免费自托管 <br> • 企业版（自带云或付费自托管）                                    |
| 可扩展性            | 自管理                                                 | 任务队列和服务器的自动扩展                                                                       |
| 容错性              | 自管理                                                 | 自动重试                                                                                         |
| 并发控制            | 简单线程                                               | 支持双文本处理                                                                                   |
| 调度                | 无                                                     | Cron调度                                                                                         |
| 监控                | 无                                                     | 与LangSmith集成，用于可观察性                                                                    |
| IDE集成             | LangGraph Studio                                      | LangGraph Studio                                                                                  |

## LangGraph平台有哪些部署选项？

我们目前为LangGraph应用程序提供以下部署选项：

- [‍自托管精简版](./deployment_options.md#self-hosted-lite)：LangGraph平台的免费（最多执行100万个节点）有限版本，可以在本地或以自托管方式运行。此版本需要LangSmith API密钥，并将所有使用情况记录到LangSmith。比付费计划的功能少。
- [云SaaS](./deployment_options.md#cloud-saas)：作为LangSmith的一部分完全管理和托管，自动更新且无需维护。
- [‍自带云（BYOC）](./deployment_options.md#bring-your-own-cloud)：在你的VPC内部署LangGraph平台，作为服务进行配置和运行。将数据保留在你的环境中，同时将服务管理外包。
- [自托管企业版](./deployment_options.md#self-hosted-enterprise)：完全在你自己的基础设施上部署LangGraph。

## LangGraph平台是开源的吗？

不是。LangGraph平台是专有软件。

有一个免费的自托管版本，可以访问基本功能。云SaaS部署选项在测试期间是免费的，但最终将成为一个付费服务。我们将在收费前提供充分的通知，并为早期采用者提供优惠价格。自带云（BYOC）和自托管企业版选项也是付费服务。[联系我们的销售团队](https://www.langchain.com/contact-sales)了解更多信息。

更多信息请参见我们的[LangGraph平台定价页面](https://www.langchain.com/pricing-langgraph-platform)。

## LangGraph是否适用于不支持工具调用的LLM？

是的！你可以将LangGraph与任何LLM一起使用。我们使用支持工具调用的LLM的主要原因是，这通常是让LLM决定下一步做什么的最方便的方式。如果你的LLM不支持工具调用，你仍然可以使用它——你只需要编写一些逻辑，将原始的LLM字符串响应转换为一个关于下一步做什么的决定。

## LangGraph是否适用于开源LLM？

是的！LangGraph对底层使用的LLM完全无关。我们在大多数教程中使用闭源LLM的主要原因是它们无缝支持工具调用，而开源LLM通常不支持。但工具调用并不是必须的（参见[此部分](#does-langgraph-work-with-llms-that-dont-support-tool-calling)），所以你完全可以将LangGraph与开源LLM一起使用。

## 我可以在不记录到LangSmith的情况下使用LangGraph Studio吗？

可以！你可以使用[LangGraph Server的开发版本](../tutorials/langgraph-platform/local-server.md)在本地运行后端。
这将连接到作为LangSmith一部分托管的Studio前端。
如果你设置环境变量`LANGSMITH_TRACING=false`，则不会向LangSmith发送任何跟踪信息。