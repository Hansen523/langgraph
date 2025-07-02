# 如何部署独立容器

在部署前，请先阅读[独立容器部署的概念指南](../../concepts/langgraph_standalone_container.md)。

## 前提条件

1. 使用[LangGraph CLI](../../concepts/langgraph_cli.md)[在本地测试您的应用](../../tutorials/langgraph-platform/local-server.md)。
1. 使用[LangGraph CLI](../../concepts/langgraph_cli.md)构建Docker镜像（即执行`langgraph build`）。
1. 独立容器部署需要以下环境变量：
    1. `REDIS_URI`：连接Redis实例的详细信息。Redis将用作发布-订阅代理，以实现从后台运行流式传输实时输出。`REDIS_URI`的值必须是有效的[Redis连接URI](https://redis-py.readthedocs.io/en/stable/connections.html#redis.Redis.from_url)。

        !!! 注意 "共享Redis实例"
            多个自托管部署可以共享同一个Redis实例。例如，对于`部署A`，可以将`REDIS_URI`设置为`redis://<hostname_1>:<port>/1`；对于`部署B`，可以设置为`redis://<hostname_1>:<port>/2`。

            `1`和`2`是同一实例中不同的数据库编号，但`<hostname_1>`是共享的。**同一数据库编号不能用于不同的部署**。

    1. `DATABASE_URI`：Postgres连接详细信息。Postgres将用于存储助手、线程、运行数据，持久化线程状态和长期记忆，并以"精确一次"语义管理后台任务队列的状态。`DATABASE_URI`的值必须是有效的[Postgres连接URI](https://www.postgresql.org/docs/current/libpq-connect.html#LIBPQ-CONNSTRING-URIS)。

        !!! 注意 "共享Postgres实例"
            多个自托管部署可以共享同一个Postgres实例。例如，对于`部署A`，可以将`DATABASE_URI`设置为`postgres://<user>:<password>@/<database_name_1>?host=<hostname_1>`；对于`部署B`，可以设置为`postgres://<user>:<password>@/<database_name_2>?host=<hostname_1>`。

            `<database_name_1>`和`database_name_2`是同一实例中不同的数据库，但`<hostname_1>`是共享的。**同一数据库不能用于不同的部署**。

    1. `LANGSMITH_API_KEY`：（如果使用[Lite版](../../concepts/langgraph_server.md#server-versions)）LangSmith API密钥。此密钥将在服务器启动时用于一次性认证。
    1. `LANGGRAPH_CLOUD_LICENSE_KEY`：（如果使用[企业版](../../concepts/langgraph_data_plane.md#licensing)）LangGraph平台许可证密钥。此密钥将在服务器启动时用于一次性认证。
    1. `LANGSMITH_ENDPOINT`：要将跟踪数据发送到[自托管LangSmith](https://docs.smith.langchain.com/self_hosting)实例，请将`LANGSMITH_ENDPOINT`设置为自托管LangSmith实例的主机名。

## Kubernetes（Helm）

使用此[Helm chart](https://github.com/langchain-ai/helm/blob/main/charts/langgraph-cloud/README.md)将LangGraph Server部署到Kubernetes集群。

## Docker

运行以下`docker`命令：
```shell
docker run \
    --env-file .env \
    -p 8123:8000 \
    -e REDIS_URI="foo" \
    -e DATABASE_URI="bar" \
    -e LANGSMITH_API_KEY="baz" \
    my-image
```

!!! 注意

    * 您需要将`my-image`替换为前提步骤中构建的镜像名称（来自`langgraph build`），并为`REDIS_URI`、`DATABASE_URI`和`LANGSMITH_API_KEY`提供适当的值。
    * 如果您的应用需要额外的环境变量，可以以类似方式传递。

## Docker Compose

Docker Compose YAML文件：
```yml
volumes:
    langgraph-data:
        driver: local
services:
    langgraph-redis:
        image: redis:6
        healthcheck:
            test: redis-cli ping
            interval: 5s
            timeout: 1s
            retries: 5
    langgraph-postgres:
        image: postgres:16
        ports:
            - "5433:5432"
        environment:
            POSTGRES_DB: postgres
            POSTGRES_USER: postgres
            POSTGRES_PASSWORD: postgres
        volumes:
            - langgraph-data:/var/lib/postgresql/data
        healthcheck:
            test: pg_isready -U postgres
            start_period: 10s
            timeout: 1s
            retries: 5
            interval: 5s
    langgraph-api:
        image: ${IMAGE_NAME}
        ports:
            - "8123:8000"
        depends_on:
            langgraph-redis:
                condition: service_healthy
            langgraph-postgres:
                condition: service_healthy
        env_file:
            - .env
        environment:
            REDIS_URI: redis://langgraph-redis:6379
            LANGSMITH_API_KEY: ${LANGSMITH_API_KEY}
            POSTGRES_URI: postgres://postgres:postgres@langgraph-postgres:5432/postgres?sslmode=disable
```

您可以在同一文件夹中运行`docker compose up`命令。

这将在端口`8123`上启动LangGraph Server（如果想更改此端口，可以通过修改`langgraph-api`卷中的端口设置）。可以通过以下命令测试应用是否健康运行：

```shell
curl --request GET --url 0.0.0.0:8123/ok
```
假设一切运行正常，您应该会看到如下响应：

```shell
{"ok":true}
```