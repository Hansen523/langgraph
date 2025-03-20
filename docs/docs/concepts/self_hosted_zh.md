# 自托管

!!! note 前提条件

    - [LangGraph 平台](./langgraph_platform.md)
    - [部署选项](./deployment_options.md)

## 版本

自托管部署有两个版本：[自托管企业版](./deployment_options.md#self-hosted-enterprise) 和 [自托管精简版](./deployment_options.md#self-hosted-lite)。

### 自托管精简版

自托管精简版是 LangGraph 平台的有限版本，您可以在本地或以自托管方式运行（每年最多执行 100 万个节点）。

使用自托管精简版时，您需要使用 [LangSmith](https://smith.langchain.com/) API 密钥进行身份验证。

### 自托管企业版

自托管企业版是 LangGraph 平台的完整版本。

要使用自托管企业版，您必须获取一个许可证密钥，并在运行 Docker 镜像时传入。要获取许可证密钥，请发送邮件至 sales@langchain.dev。

## 要求

- 您使用 `langgraph-cli` 和/或 [LangGraph Studio](./langgraph_studio.md) 应用程序在本地测试图。
- 您使用 `langgraph build` 命令构建镜像。

## 工作原理

- 在您自己的基础设施上部署 Redis 和 Postgres 实例。
- 使用 [LangGraph CLI](./langgraph_cli.md) 构建 [LangGraph Server](./langgraph_server.md) 的 Docker 镜像。
- 部署一个将运行该 Docker 镜像的 Web 服务器，并传入必要的环境变量。

!!! warning "注意"

    LangGraph 平台部署视图可选地可用于自托管 LangGraph 部署。通过一键操作，自托管 LangGraph 部署可以部署在自托管 LangSmith 实例所在的同一 Kubernetes 集群中。

有关逐步说明，请参阅 [如何设置自托管 LangGraph 部署](../how-tos/deploy-self-hosted.md)。

## Helm Chart

如果您希望在 Kubernetes 上部署 LangGraph Cloud，可以使用此 [Helm chart](https://github.com/langchain-ai/helm/blob/main/charts/langgraph-cloud/README.md)。

## 相关

- [如何设置自托管 LangGraph 部署](../how-tos/deploy-self-hosted.md)。