# 部署选项

!!! info "前提条件"

    - [LangGraph 平台](./langgraph_platform.md)
    - [LangGraph 服务器](./langgraph_server.md)
    - [LangGraph 平台计划](./plans.md)

## 概述

LangGraph 平台有四种主要的部署选项：

1. **[自托管精简版](#自托管精简版)**: 适用于所有计划。

2. **[自托管企业版](#自托管企业版)**: 仅适用于**企业**计划。

3. **[云 SaaS](#云-saas)**: 适用于**Plus**和**企业**计划。

4. **[自带云](#自带云)**: 仅适用于**企业**计划，**且仅在 AWS 上**。

有关不同计划的更多信息，请参阅[LangGraph 平台计划](./plans.md)。

以下指南将解释这些部署选项之间的区别。

## 自托管企业版

!!! important

    自托管企业版仅适用于**企业**计划。

!!! warning "注意"

    LangGraph 平台部署视图可选择性地用于自托管企业版 LangGraph 部署。通过点击一次，自托管 LangGraph 部署可以部署在与自托管 LangSmith 实例相同的 Kubernetes 集群中。

使用自托管企业版部署时，您需要负责管理基础设施，包括设置和维护所需的数据库和 Redis 实例。

您将使用 [LangGraph CLI](./langgraph_cli.md) 构建一个 Docker 镜像，然后可以在您自己的基础设施上进行部署。

更多信息，请参阅：

* [自托管概念指南](./self_hosted.md)
* [自托管部署操作指南](../how-tos/deploy-self-hosted.md)

## 自托管精简版

!!! important

    自托管精简版适用于所有计划。

!!! warning "注意"

    LangGraph 平台部署视图可选择性地用于自托管精简版 LangGraph 部署。通过点击一次，自托管 LangGraph 部署可以部署在与自托管 LangSmith 实例相同的 Kubernetes 集群中。

自托管精简版部署选项是 LangGraph 平台的免费版本（每年最多执行 100 万个节点），您可以在本地或以自托管方式运行。

使用自托管精简版部署时，您需要负责管理基础设施，包括设置和维护所需的数据库和 Redis 实例。

您将使用 [LangGraph CLI](./langgraph_cli.md) 构建一个 Docker 镜像，然后可以在您自己的基础设施上进行部署。

[Cron 任务](../cloud/how-tos/cron_jobs.md)不适用于自托管精简版部署。

更多信息，请参阅：

* [自托管概念指南](./self_hosted.md)
* [自托管部署操作指南](../how-tos/deploy-self-hosted.md)

## 云 SaaS

!!! important

    LangGraph 平台的云 SaaS 版本仅适用于**Plus**和**企业**计划。

[云 SaaS](./langgraph_cloud.md) 版本的 LangGraph 平台作为 [LangSmith](https://smith.langchain.com/) 的一部分进行托管。

LangGraph 平台的云 SaaS 版本提供了一种简单的方式来部署和管理您的 LangGraph 应用程序。

此部署选项提供对 LangGraph 平台 UI（在 LangSmith 内）的访问权限，并与 GitHub 集成，允许您从 GitHub 上的任何代码库部署代码。

更多信息，请参阅：

* [云 SaaS 概念指南](./langgraph_cloud.md)
* [如何部署到云 SaaS](../cloud/deployment/cloud.md)

## 自带云

!!! important

    LangGraph 平台的自带云版本仅适用于**企业**计划。

这结合了云和自托管的最佳特性。通过 LangGraph 平台 UI（在 LangSmith 内）创建您的部署，我们负责管理基础设施，因此您无需操心。所有基础设施都在您的云中运行。目前仅在 AWS 上可用。

更多信息，请参阅：

* [自带云概念指南](./bring_your_own_cloud.md)

## 相关

更多信息，请参阅：

* [LangGraph 平台计划](./plans.md)
* [LangGraph 平台定价](https://www.langchain.com/langgraph-platform-pricing)
* [部署操作指南](../how-tos/index.md#deployment)