# 快速入门：启动本地 LangGraph 服务器

这是一个快速入门指南，帮助您在本地启动并运行 LangGraph 应用。

!!! info "要求"

    - Python >= 3.11
    - [LangGraph CLI](https://langchain-ai.github.io/langgraph/cloud/reference/cli/)：需要 langchain-cli[inmem] >= 0.1.58

## 安装 LangGraph CLI

```bash
pip install --upgrade "langgraph-cli[inmem]"
```

## 🌱 创建 LangGraph 应用

从 `react-agent` 模板创建一个新应用。该模板是一个简单的代理，可以灵活地扩展到许多工具。

=== "Python 服务器"

    ```shell
    langgraph new path/to/your/app --template react-agent-python 
    ```

=== "Node 服务器"

    ```shell
    langgraph new path/to/your/app --template react-agent-js
    ```

!!! tip "其他模板"

    如果您使用 `langgraph new` 而不指定模板，将会出现一个交互式菜单，允许您从可用模板列表中选择。

## 安装依赖

在您的新 LangGraph 应用的根目录中，以 `edit` 模式安装依赖项，以便服务器使用您的本地更改：

```shell
pip install -e .
```

## 创建 `.env` 文件

您将在新 LangGraph 应用的根目录中找到 `.env.example` 文件。在根目录中创建一个 `.env` 文件，并将 `.env.example` 文件的内容复制到其中，填写必要的 API 密钥：

```bash
LANGSMITH_API_KEY=lsv2...
TAVILY_API_KEY=tvly-...
ANTHROPIC_API_KEY=sk-
OPENAI_API_KEY=sk-...
```

??? note "获取 API 密钥"

    - **LANGSMITH_API_KEY**：前往 [LangSmith 设置页面](https://smith.langchain.com/settings)。然后点击 **创建 API 密钥**。
    - **ANTHROPIC_API_KEY**：从 [Anthropic](https://console.anthropic.com/) 获取 API 密钥。
    - **OPENAI_API_KEY**：从 [OpenAI](https://openai.com/) 获取 API 密钥。
    - **TAVILY_API_KEY**：在 [Tavily 网站](https://app.tavily.com/) 上获取 API 密钥。

## 🚀 启动 LangGraph 服务器

```shell
langgraph dev
```

这将启动本地 LangGraph API 服务器。如果成功运行，您应该会看到类似以下内容：

>    准备就绪！
> 
>    - API: [http://localhost:2024](http://localhost:2024/)
>     
>    - 文档: http://localhost:2024/docs
>     
>    - LangGraph Studio Web UI: https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:2024


!!! note "内存模式"

    `langgraph dev` 命令以内存模式启动 LangGraph 服务器。此模式适用于开发和测试目的。对于生产用途，您应该部署 LangGraph 服务器并访问持久存储后端。

    如果您想使用持久存储后端测试您的应用程序，可以使用 `langgraph up` 命令而不是 `langgraph dev`。您需要
    在您的机器上安装 `docker` 才能使用此命令。

## LangGraph Studio Web UI

LangGraph Studio Web 是一个专门的 UI，您可以连接到 LangGraph API 服务器，以启用本地应用程序的可视化、交互和调试。通过访问 `langgraph dev` 命令输出中提供的 URL，在 LangGraph Studio Web UI 中测试您的图。

>    - LangGraph Studio Web UI: https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:2024

!!! info "连接到具有自定义主机/端口的服务器"

    如果您使用自定义主机/端口运行 LangGraph API 服务器，可以通过更改 `baseUrl` URL 参数将 Studio Web UI 指向它。例如，如果您在端口 8000 上运行服务器，可以将上述 URL 更改为以下内容：

    ```
    https://smith.langchain.com/studio/baseUrl=http://127.0.0.1:8000
    ```


!!! warning "Safari 兼容性"
    
    目前，LangGraph Studio Web 在本地运行服务器时不支持 Safari。

## 测试 API

=== "Python SDK (异步)"

    **安装 LangGraph Python SDK**

    ```shell
    pip install langgraph-sdk
    ```

    **向助手发送消息（无线程运行）**

    ```python
    from langgraph_sdk import get_client

    client = get_client(url="http://localhost:2024")

    async for chunk in client.runs.stream(
        None,  # 无线程运行
        "agent", # 助手名称。在 langgraph.json 中定义。
        input={
            "messages": [{
                "role": "human",
                "content": "What is LangGraph?",
            }],
        },
        stream_mode="updates",
    ):
        print(f"Receiving new event of type: {chunk.event}...")
        print(chunk.data)
        print("\n\n")
    ```

=== "Python SDK (同步)"

    **安装 LangGraph Python SDK**

    ```shell
    pip install langgraph-sdk
    ```

    **向助手发送消息（无线程运行）**

    ```python
    from langgraph_sdk import get_sync_client

    client = get_sync_client(url="http://localhost:2024")

    for chunk in client.runs.stream(
        None,  # 无线程运行
        "agent", # 助手名称。在 langgraph.json 中定义。
        input={
            "messages": [{
                "role": "human",
                "content": "What is LangGraph?",
            }],
        },
        stream_mode="updates",
    ):
        print(f"Receiving new event of type: {chunk.event}...")
        print(chunk.data)
        print("\n\n")
    ```

=== "Javascript SDK"

    **安装 LangGraph JS SDK**

    ```shell
    npm install @langchain/langgraph-sdk
    ```

    **向助手发送消息（无线程运行）**

    ```js
    const { Client } = await import("@langchain/langgraph-sdk");

    // 仅在调用 langgraph dev 时更改了默认端口时才设置 apiUrl
    const client = new Client({ apiUrl: "http://localhost:2024"});

    const streamResponse = client.runs.stream(
        null, // 无线程运行
        "agent", // 助手 ID
        {
            input: {
                "messages": [
                    { "role": "user", "content": "What is LangGraph?"}
                ]
            },
            streamMode: "messages",
        }
    );

    for await (const chunk of streamResponse) {
        console.log(`Receiving new event of type: ${chunk.event}...`);
        console.log(JSON.stringify(chunk.data));
        console.log("\n\n");
    }
    ```

=== "Rest API"

    ```bash
    curl -s --request POST \
        --url "http://localhost:2024/runs/stream" \
        --header 'Content-Type: application/json' \
        --data "{
            \"assistant_id\": \"agent\",
            \"input\": {
                \"messages\": [
                    {
                        \"role\": \"human\",
                        \"content\": \"What is LangGraph?\"
                    }
                ]
            },
            \"stream_mode\": \"updates\"
        }" 
    ```

!!! tip "授权"

    如果您连接到远程服务器，则需要提供 LangSmith
    API 密钥进行授权。请参阅客户端的 API 参考以获取更多信息。

## 下一步

现在您已经在本地运行了 LangGraph 应用，通过探索部署和高级功能进一步推进您的旅程：

### 🌐 部署到 LangGraph 云

- **[LangGraph 云快速入门](../../cloud/quick_start.md)**：使用 LangGraph 云部署您的 LangGraph 应用。

### 📚 了解更多关于 LangGraph 平台的信息

通过这些资源扩展您的知识：

- **[LangGraph 平台概念](../../concepts/index.md#langgraph-platform)**：了解 LangGraph 平台的基础概念。  
- **[LangGraph 平台操作指南](../../how-tos/index.md#langgraph-platform)**：发现构建和部署应用程序的分步指南。

### 🛠️ 开发者参考

访问详细的开发和 API 使用文档：

- **[LangGraph 服务器 API 参考](../../cloud/reference/api/api_ref.html)**：探索 LangGraph 服务器 API 文档。  
- **[Python SDK 参考](../../cloud/reference/sdk/python_sdk_ref.md)**：探索 Python SDK API 参考。
- **[JS/TS SDK 参考](../../cloud/reference/sdk/js_ts_sdk_ref.md)**：探索 JS/TS SDK API 参考。