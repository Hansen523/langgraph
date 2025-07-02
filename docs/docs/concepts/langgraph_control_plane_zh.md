---
search:
  boost: 2
---

# LangGraph控制平面

术语"控制平面"泛指用户创建和更新[LangGraph服务器](./langgraph_server.md)（部署）的控制平面UI，以及支持该UI体验的控制平面API。

当用户通过控制平面UI进行更新时，这些更新会被存储在控制平面状态中。[LangGraph数据平面](./langgraph_data_plane.md)的"监听器"应用通过调用控制平面API来轮询这些更新。

## 控制平面UI

通过控制平面UI，您可以：

- 查看待处理部署的列表。
- 查看单个部署的详细信息。
- 创建新部署。
- 更新部署。
- 更新部署的环境变量。
- 查看部署的构建和服务器日志。
- 查看部署的CPU和内存使用情况等指标。
- 删除部署。

控制平面UI内置于[LangSmith](https://docs.smith.langchain.com/langgraph_cloud)中。

## 控制平面API

本节描述控制平面API的数据模型。该API用于创建、更新和删除部署，但不对外公开访问。

### 部署

部署是LangGraph服务器的一个实例。单个部署可以有许多修订版本。

### 修订版本

修订版本是部署的一个迭代。创建新部署时会自动生成初始修订版本。要部署代码变更或更新环境变量，必须创建新的修订版本。

### 环境变量

环境变量是为部署设置的。所有环境变量都以机密形式存储（即保存在机密存储中）。

## 控制平面特性

本节描述控制平面的各种特性。

### 部署类型

为简化操作，控制平面提供两种资源分配不同的部署类型：`开发`和`生产`。

| **部署类型** | **CPU/内存**  | **伸缩性**       | **数据库**                                                                     |
|--------------|---------------|------------------|-------------------------------------------------------------------------------|
| 开发         | 1 CPU, 1 GB内存 | 最多1个容器      | 10 GB磁盘，无备份                                                             |
| 生产         | 2 CPU, 2 GB内存 | 最多10个容器     | 磁盘自动扩展，自动备份，高可用性（多区域配置）                                |

CPU和内存资源按容器分配。

!!! warning "不可变部署类型"

    部署创建后，其类型无法更改。

!!! info "资源自定义"
    对于`生产`类型部署，可根据用例和容量限制手动增加资源。联系support@langchain.dev申请增加资源。

    对于`开发`类型部署，可根据用例和容量限制手动增加数据库磁盘大小。大多数情况下应配置[TTL](../how-tos/ttl/configure_ttl.md)来管理磁盘使用。联系support@langchain.dev申请增加资源。

    [自托管数据平面](../concepts/langgraph_self_hosted_data_plane.md)和[自托管控制平面](../concepts/langgraph_self_hosted_control_plane.md)部署的资源可完全自定义。

### 数据库配置

控制平面与[LangGraph数据平面](./langgraph_data_plane.md)的"监听器"应用协作，自动为每个部署创建Postgres数据库。该数据库作为部署的[持久层](../concepts/persistence.md)。

在实现LangGraph应用时，开发者无需配置[检查点](../concepts/persistence.md#checkpointer-libraries)。系统会自动为图配置检查点，任何手动配置的检查点都会被自动配置的检查点替换。

无法直接访问数据库。所有访问都通过[LangGraph服务器](../concepts/langgraph_server.md)进行。

数据库在部署删除前不会被删除。

!!! info
    [自托管数据平面](../concepts/langgraph_self_hosted_data_plane.md)和[自托管控制平面](../concepts/langgraph_self_hosted_control_plane.md)部署可配置自定义Postgres实例。

### 异步部署

部署和修订版本的基础设施是异步配置和部署的，不会在提交后立即完成。目前部署可能需要几分钟时间。

- 创建新部署时会新建数据库。数据库创建是一次性步骤，导致初始修订版本的部署时间较长。
- 为部署创建后续修订版本时不涉及数据库创建步骤，因此部署时间显著缩短。
- 每个修订版本的部署过程包含构建步骤，可能需要几分钟。

控制平面与[LangGraph数据平面](./langgraph_data_plane.md)的"监听器"应用协作实现异步部署。

### 监控

部署就绪后，控制平面会监控部署并记录以下指标：

- 部署的CPU和内存使用情况
- 容器重启次数
- 副本数量（会随[自动扩展](../concepts/langgraph_data_plane.md#autoscaling)增加）
- [Postgres](../concepts/langgraph_data_plane.md#postgres)的CPU、内存和磁盘使用情况

这些指标以图表形式显示在控制平面UI中。

### LangSmith集成

每个部署会自动创建同名的[LangSmith](https://docs.smith.langchain.com/)追踪项目。创建部署时无需指定`LANGCHAIN_TRACING`和`LANGSMITH_API_KEY`/`LANGCHAIN_API_KEY`环境变量，它们由控制平面自动设置。

删除部署时，追踪数据和追踪项目不会被删除。