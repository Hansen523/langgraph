---
search:
  boost: 2
---

# LangGraph 数据平面

术语"数据平面"广义上指代 [LangGraph 服务器](./langgraph_server.md)（部署实例）、每个服务器对应的基础设施，以及持续从 [LangGraph 控制平面](./langgraph_control_plane.md)轮询更新的"监听器"应用。

## 服务器基础设施

除 [LangGraph 服务器](./langgraph_server.md)本身外，以下各服务器的基础设施组件也被纳入"数据平面"的广义定义：
- PostgreSQL
- Redis
- 密钥存储
- 自动扩缩容组件

## "监听器"应用

数据平面"监听器"应用定期调用 [控制平面 API](../concepts/langgraph_control_plane.md#control-plane-api) 以：
- 判断是否需要创建新部署
- 判断现有部署是否需要更新（即新建修订版本）
- 判断现有部署是否需要删除

换言之，数据平面"监听器"读取控制平面的最新状态（期望状态），并采取行动调整待处理部署（当前状态）以匹配最新状态。

## PostgreSQL

PostgreSQL 是 LangGraph 服务器中所有用户、运行和长期记忆数据的持久化层。它既存储检查点（详见[此处](./persistence.md)），也保存服务器资源（线程、运行记录、助手和定时任务），以及长期记忆存储中的条目（详见[此处](./persistence.md#memory-store)）。

## Redis

Redis 在每个 LangGraph 服务器中用于服务器与队列工作线程间的通信，并存储临时元数据。用户或运行数据不会存储在 Redis 中。

### 通信机制

LangGraph 服务器中的所有运行都由部署实例内的后台工作线程池执行。为实现某些运行特性（如取消操作和输出流式传输），需要在服务器与处理特定运行的工作线程间建立双向通信通道。我们使用 Redis 来组织这种通信：

1. 使用 Redis 列表作为唤醒工作线程的机制，当新运行创建时立即通知。该列表仅存储哨兵值，不包含实际运行信息。工作线程随后从 PostgreSQL 获取运行详情。
2. 结合 Redis 字符串和 PubSub 频道，服务器将运行取消请求传递给对应工作线程。
3. 工作线程通过 Redis PubSub 频道广播运行中的流式输出。服务器中所有打开的 `/stream` 请求都会订阅该频道，并将到达的事件实时转发至响应。Redis 中不会持久化任何事件。

### 临时元数据

LangGraph 服务器中的运行可能因特定故障重试（当前仅针对运行期间遇到的短暂 PostgreSQL 错误）。为限制重试次数（当前限制为每次运行最多3次尝试），我们会在运行被拾取时将尝试次数记录到 Redis 字符串中。该记录仅包含运行ID，不涉及其他具体信息，且会在短期内自动过期。

## 数据平面特性

本节描述数据平面的各项功能特性。

### 自动扩缩容

[`生产`类型](../concepts/langgraph_control_plane.md#deployment-types)部署会自动扩容至最多10个容器。扩缩容依据3项指标：
1. CPU利用率
2. 内存利用率
3. 待处理（进行中）[运行](./assistants.md#execution)数量

CPU利用率以75%为目标值，内存利用率同样以75%为目标。对于待处理运行数量，每容器以10个运行为目标值。例如：若当前有1个容器但20个待处理运行，则会扩容至2个容器（20/2=10）。

各项指标独立计算，最终采用建议容器数最多的指标结果。缩容操作会延迟30分钟执行，此冷却期可避免频繁扩缩。

### 静态IP地址

!!! info "仅限云SaaS"
    静态IP地址仅适用于 [云SaaS](../concepts/langgraph_cloud.md) 部署。

2025年1月6日后创建的部署，所有流量都将通过NAT网关传输。该网关的静态IP地址因数据区域而异，详见下表：

| 美国区域       | 欧洲区域       |
|----------------|----------------|
| 35.197.29.146  | 34.13.192.67   |
| 34.145.102.123 | 34.147.105.64  |
| 34.169.45.153  | 34.90.22.166   |
| 34.82.222.17   | 34.147.36.213  |
| 35.227.171.135 | 34.32.137.113  | 
| 34.169.88.30   | 34.91.238.184  |
| 34.19.93.202   | 35.204.101.241 |
| 34.19.34.50    | 35.204.48.32   |

### 自定义PostgreSQL

!!! info 
    自定义PostgreSQL仅适用于 [自托管数据平面](../concepts/langgraph_self_hosted_data_plane.md) 和 [自托管控制平面](../concepts/langgraph_self_hosted_control_plane.md) 部署。

可通过设置 [`POSTGRES_URI_CUSTOM`](../cloud/reference/env_var.md#postgres_uri_custom) 环境变量使用自定义PostgreSQL实例替代[控制平面自动创建的数据库](./langgraph_control_plane.md#database-provisioning)。多个部署可共享同一PostgreSQL实例（需使用不同数据库名），但禁止不同部署共用相同数据库。

### 自定义Redis

!!! info
    自定义Redis仅适用于 [自托管数据平面](../concepts/langgraph_self_hosted_control_plane.md) 和 [自托管控制平面](../concepts/langgraph_self_hosted_control_plane.md) 部署。

通过设置 [REDIS_URI_CUSTOM](../cloud/reference/env_var.md#redis_uri_custom) 环境变量可使用自定义Redis实例。多个部署可共享同一Redis实例（需使用不同数据库编号），但禁止不同部署共用相同数据库编号。

### LangSmith追踪

LangGraph服务器自动配置为向LangSmith发送追踪数据，各部署选项差异如下：

| 云SaaS       | 自托管数据平面       | 自托管控制平面               | 独立容器                 |
|--------------|----------------------|------------------------------|--------------------------|
| 必选<br>追踪至LangSmith SaaS | 可选<br>禁用追踪或追踪至LangSmith SaaS | 可选<br>禁用追踪或追踪至自托管LangSmith | 可选<br>禁用追踪或选择追踪目标 |

### 遥测数据

LangGraph服务器自动配置上报计费所需的遥测元数据，各部署选项差异如下：

| 云SaaS       | 自托管数据平面       | 自托管控制平面               | 独立容器                 |
|--------------|----------------------|------------------------------|--------------------------|
| 发送至LangSmith SaaS | 发送至LangSmith SaaS | 空气隔离许可证需自报用量<br>平台许可证发送至LangSmith SaaS | 空气隔离许可证需自报用量<br>平台许可证发送至LangSmith SaaS |

### 许可验证

LangGraph服务器自动执行许可证验证，各部署选项差异如下：

| 云SaaS       | 自托管数据平面       | 自托管控制平面               | 独立容器                 |
|--------------|----------------------|------------------------------|--------------------------|
| 通过LangSmith SaaS验证API密钥 | 通过LangSmith SaaS验证API密钥 | 验证空气隔离许可证或通过LangSmith SaaS验证平台许可证 | 验证空气隔离许可证或通过LangSmith SaaS验证平台许可证 |