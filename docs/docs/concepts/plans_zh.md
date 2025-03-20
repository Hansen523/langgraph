# LangGraph 平台计划


## 概述
LangGraph 平台是一个用于在生产环境中部署代理应用的商业解决方案。
使用该平台有三种不同的计划。

- **开发者**：所有 [LangSmith](https://smith.langchain.com/) 用户都可以访问此计划。您只需创建一个 LangSmith 账户即可注册此计划。这将使您能够使用 [自托管精简版](./deployment_options.md#自托管精简版) 部署选项。
- **Plus**：所有拥有 [Plus 账户](https://docs.smith.langchain.com/administration/pricing) 的 [LangSmith](https://smith.langchain.com/) 用户都可以访问此计划。您只需将您的 LangSmith 账户升级为 Plus 计划类型即可注册此计划。这将使您能够使用 [云](./deployment_options.md#云-saas) 部署选项。
- **企业版**：此计划与 LangSmith 计划分开。您可以通过联系 sales@langchain.dev 注册此计划。这将使您能够使用所有部署选项：[云](./deployment_options.md#云-saas)、[自带云](./deployment_options.md#自带云) 和 [自托管企业版](./deployment_options.md#自托管企业版)


## 计划详情

|                                                                  | 开发者                                   | Plus                                                  | 企业版                                          |
|------------------------------------------------------------------|---------------------------------------------|-------------------------------------------------------|-----------------------------------------------------|
| 部署选项                                               | 自托管精简版                            | 云                                                 | 自托管企业版、云、自带云 |
| 使用情况                                                     | 免费，每年限制执行 1M 个节点 | 在 Beta 期间免费，之后按执行的节点收费 | 自定义                                              |
| 用于检索和更新状态及会话历史的 API | ✅                                           | ✅                                                     | ✅                                                   |
| 用于检索和更新长期记忆的 API                | ✅                                           | ✅                                                     | ✅                                                   |
| 可水平扩展的任务队列和服务器                    | ✅                                           | ✅                                                     | ✅                                                   |
| 输出和中间步骤的实时流式传输            | ✅                                           | ✅                                                     | ✅                                                   |
| 助手 API（用于 LangGraph 应用的可配置模板）       | ✅                                           | ✅                                                     | ✅                                                   |
| 定时任务调度                                                  | --                                          | ✅                                                     | ✅                                                   |
| 用于原型设计的 LangGraph Studio                                 | 	✅                                         | ✅                                                    | ✅                                                  |
| 调用 LangGraph API 的身份验证和授权        | --                                          | 即将推出！                                          | 即将推出！                                        |
| 智能缓存以减少对 LLM API 的流量                       | --                                          | 即将推出！                                          | 即将推出！                                        |
| 用于状态的发布/订阅 API                                  | --                                          | 即将推出！                                          | 即将推出！                                        |
| 调度优先级                                        | --                                          | 即将推出！                                          | 即将推出！                                        |

有关定价信息，请参阅 [LangGraph 平台定价](https://www.langchain.com/langgraph-platform-pricing)。

## 相关信息

更多信息，请参阅：

* [部署选项概念指南](./deployment_options.md)
* [LangGraph 平台定价](https://www.langchain.com/langgraph-platform-pricing)
* [LangSmith 计划](https://docs.smith.langchain.com/administration/pricing)