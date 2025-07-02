---
search:
  boost: 2
---

# 独立容器部署

要部署[LangGraph服务器](../concepts/langgraph_server.md)，请参照[如何部署独立容器](../cloud/deployment/standalone_container.md)操作指南。

## 概述

独立容器部署是最自由的部署模式。该方案不包含[控制平面](./langgraph_control_plane.md)，所有[数据平面](./langgraph_data_plane.md)基础设施均由您自行管理。

|                   | [控制平面](../concepts/langgraph_control_plane.md) | [数据平面](../concepts/langgraph_data_plane.md) |
|-------------------|-------------------|------------|
| **组成要素** | 不适用 | <ul><li>LangGraph服务器集群</li><li>Postgres、Redis等依赖组件</li></ul> |
| **部署位置** | 不适用 | 您的云环境 |
| **运维责任方** | 不适用 | 您自行负责 |

!!! warning

      LangGraph平台不可部署于无服务器环境。冷启动可能导致任务丢失，且自动扩缩容无法可靠运行。

## 架构设计

![独立容器架构图](./img/langgraph_platform_deployment_architecture.png)

## 计算平台支持

### Kubernetes

独立容器部署方案支持在Kubernetes集群上部署数据平面基础设施。

### Docker

该方案同样支持在任何Docker兼容的计算平台部署数据平面组件。

## Lite与Enterprise版本对比

独立容器部署支持[两种服务器版本](../concepts/langgraph_server.md#langgraph-server)：

- `Lite`版本免费但功能受限
- `Enterprise`版本需定制报价，提供完整功能特性

功能差异详情请参阅[LangGraph服务器](../concepts/langgraph_server.md#server-versions)文档。