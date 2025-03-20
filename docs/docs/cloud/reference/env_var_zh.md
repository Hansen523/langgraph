# 环境变量

LangGraph 云服务器支持特定的环境变量来配置部署。

## `DD_API_KEY`

指定 `DD_API_KEY`（您的 [Datadog API 密钥](https://docs.datadoghq.com/account_management/api-app-keys/)）以自动为部署启用 Datadog 跟踪。指定其他 [`DD_*` 环境变量](https://ddtrace.readthedocs.io/en/stable/configuration.html) 以配置跟踪工具。

如果指定了 `DD_API_KEY`，应用程序进程将被包装在 [`ddtrace-run` 命令](https://ddtrace.readthedocs.io/en/stable/installation_quickstart.html) 中。通常需要其他 `DD_*` 环境变量（例如 `DD_SITE`、`DD_ENV`、`DD_SERVICE`、`DD_TRACE_ENABLED`）来正确配置跟踪工具。有关更多详细信息，请参阅 [`DD_*` 环境变量](https://ddtrace.readthedocs.io/en/stable/configuration.html)。

## `LANGCHAIN_TRACING_SAMPLING_RATE`

发送到 LangSmith 的跟踪的采样率。有效值：`0` 到 `1` 之间的任何浮点数。

有关更多详细信息，请参阅 <a href="https://docs.smith.langchain.com/how_to_guides/tracing/sample_traces" target="_blank">LangSmith 文档</a>。

## `LANGGRAPH_AUTH_TYPE`

LangGraph 云服务器部署的认证类型。有效值：`langsmith`、`noop`。

对于 LangGraph 云的部署，此环境变量会自动设置。对于本地开发或认证由外部处理的部署（例如自托管），将此环境变量设置为 `noop`。

## `LANGSMITH_RUNS_ENDPOINTS`

仅适用于 [自带云 (BYOC)](../../concepts/bring_your_own_cloud.md) 部署与 [自托管 LangSmith](https://docs.smith.langchain.com/self_hosting)。

设置此环境变量以使 BYOC 部署将跟踪发送到自托管的 LangSmith 实例。`LANGSMITH_RUNS_ENDPOINTS` 的值是一个 JSON 字符串：`{"<自托管_LANGSMITH_主机名>":"<LANGSMITH_API_KEY>"}`。

`自托管_LANGSMITH_主机名` 是自托管 LangSmith 实例的主机名。它必须对 BYOC 部署可访问。`LANGSMITH_API_KEY` 是从自托管 LangSmith 实例生成的 LangSmith API 密钥。

## `N_JOBS_PER_WORKER`

LangGraph 云任务队列的每个工作者的作业数。默认为 `10`。

## `POSTGRES_URI_CUSTOM`

仅适用于 [自带云 (BYOC)](../../concepts/bring_your_own_cloud.md) 部署。

指定 `POSTGRES_URI_CUSTOM` 以使用外部管理的 Postgres 实例。`POSTGRES_URI_CUSTOM` 的值必须是有效的 [Postgres 连接 URI](https://www.postgresql.org/docs/current/libpq-connect.html#LIBPQ-CONNSTRING-URIS)。

Postgres：

- 版本 15.8 或更高。
- 必须存在初始数据库，并且连接 URI 必须引用该数据库。

控制平面功能：

- 如果指定了 `POSTGRES_URI_CUSTOM`，LangGraph 控制平面将不会为服务器配置数据库。
- 如果删除了 `POSTGRES_URI_CUSTOM`，LangGraph 控制平面将不会为服务器配置数据库，并且不会删除外部管理的 Postgres 实例。
- 如果删除了 `POSTGRES_URI_CUSTOM`，部署修订将不会成功。一旦指定了 `POSTGRES_URI_CUSTOM`，它必须在部署的整个生命周期内始终设置。
- 如果删除了部署，LangGraph 控制平面不会删除外部管理的 Postgres 实例。
- `POSTGRES_URI_CUSTOM` 的值可以更新。例如，URI 中的密码可以更新。

数据库连接性：

- 外部管理的 Postgres 实例必须对 ECS 集群中的 LangGraph 服务器服务可访问。BYOC 用户负责确保连接性。
- 例如，如果配置了 AWS RDS Postgres 实例，可以在与 ECS 集群相同的 VPC (`langgraph-cloud-vpc`) 中配置，并使用 `langgraph-cloud-service-sg` 安全组以确保连接性。