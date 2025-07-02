---
tags:
  - mcp
  - platform
hide:
  - tags
---

# LangGraph Server中的MCP端点

**模型上下文协议（MCP）**是一种开放式协议，用于以模型无关的格式描述工具和数据源，使LLM能够通过结构化API发现和使用它们。

[LangGraph Server](./langgraph_server.md)使用[可流式HTTP传输](https://spec.modelcontextprotocol.io/specification/2025-03-26/basic/transports/#streamable-http)实现了MCP。这使得LangGraph**智能体**可以作为**MCP工具**暴露，从而支持任何兼容MCP并支持可流式HTTP的客户端使用。

MCP端点位于[LangGraph Server](./langgraph_server.md)的`/mcp`路径。

## 要求

要使用MCP，请确保已安装以下依赖项：

- `langgraph-api >= 0.2.3`
- `langgraph-sdk >= 0.1.61`

可通过以下命令安装：

```bash
pip install "langgraph-api>=0.2.3" "langgraph-sdk>=0.1.61"
```

## 将智能体暴露为MCP工具

部署后，您的智能体会以如下配置出现在MCP端点中：

- **工具名称**：智能体的名称。
- **工具描述**：智能体的描述。
- **工具输入模式**：智能体的输入模式。

### 设置名称和描述

您可以在`langgraph.json`中设置智能体的名称和描述：

```json
{
    "graphs": {
        "my_agent": {
            "path": "./my_agent/agent.py:graph",
            "description": "A description of what the agent does"
        }
    },
    "env": ".env"
}
```

部署后，可使用LangGraph SDK更新名称和描述。

### 模式

定义清晰、最小化的输入和输出模式，避免向LLM暴露不必要的内部复杂性。

默认的[MessagesState](./low_level.md#messagesstate)使用`AnyMessage`，它支持多种消息类型，但对LLM直接暴露过于宽泛。

相反，应定义**自定义智能体或工作流**，使用明确类型的输入输出结构。

例如，回答文档问题的工作流可以如下：

```python
from langgraph.graph import StateGraph, START, END
from typing_extensions import TypedDict

# 定义输入模式
class InputState(TypedDict):
    question: str

# 定义输出模式
class OutputState(TypedDict):
    answer: str

# 合并输入和输出
class OverallState(InputState, OutputState):
    pass

# 定义处理节点
def answer_node(state: InputState):
    # 替换为实际逻辑并执行有用操作
    return {"answer": "bye", "question": state["question"]}

# 构建带有明确模式的图
builder = StateGraph(OverallState, input_schema=InputState, output_schema=OutputState)
builder.add_node(answer_node)
builder.add_edge(START, "answer_node")
builder.add_edge("answer_node", END)
graph = builder.compile()

# 运行图
print(graph.invoke({"question": "hi"}))
```

更多详情参见[低级概念指南](https://langchain-ai.github.io/langgraph/concepts/low_level/#state)。

## 使用概览

启用MCP的步骤：

- 升级到langgraph-api>=0.2.3。如果部署LangGraph平台，创建新版本时会自动完成。
- MCP工具（智能体）将自动暴露。
- 使用支持可流式HTTP的MCP兼容客户端连接。

### 客户端

使用MCP兼容客户端连接到LangGraph服务器。以下示例展示了如何用不同编程语言连接。

=== "JavaScript/TypeScript"

    ```bash
    npm install @modelcontextprotocol/sdk
    ```

    > **注意**
    > 将`serverUrl`替换为您的LangGraph服务器URL并按需配置认证头。

    ```js
    import { Client } from "@modelcontextprotocol/sdk/client/index.js";
    import { StreamableHTTPClientTransport } from "@modelcontextprotocol/sdk/client/streamableHttp.js";

    // 连接到LangGraph MCP端点
    async function connectClient(url) {
        const baseUrl = new URL(url);
        const client = new Client({
            name: 'streamable-http-client',
            version: '1.0.0'
        });

        const transport = new StreamableHTTPClientTransport(baseUrl);
        await client.connect(transport);

        console.log("Connected using Streamable HTTP transport");
        console.log(JSON.stringify(await client.listTools(), null, 2));
        return client;
    }

    const serverUrl = "http://localhost:2024/mcp";

    connectClient(serverUrl)
        .then(() => {
            console.log("Client connected successfully");
        })
        .catch(error => {
            console.error("Failed to connect client:", error);
        });
    ```

=== "Python"

    安装适配器：

    ```bash
    pip install langchain-mcp-adapters
    ```

    以下是连接到远程MCP端点并使用智能体作为工具的示例：

    ```python
    # 创建stdio连接的服务器参数
    from mcp import ClientSession
    from mcp.client.streamable_http import streamablehttp_client
    import asyncio

    from langchain_mcp_adapters.tools import load_mcp_tools
    from langgraph.prebuilt import create_react_agent

    server_params = {
        "url": "https://mcp-finance-agent.xxx.us.langgraph.app/mcp",
        "headers": {
            "X-Api-Key":"lsv2_pt_your_api_key"
        }
    }

    async def main():
        async with streamablehttp_client(**server_params) as (read, write, _):
            async with ClientSession(read, write) as session:
                # 初始化连接
                await session.initialize()

                # 将远程图作为工具加载
                tools = await load_mcp_tools(session)

                # 创建并运行带有工具的react智能体
                agent = create_react_agent("openai:gpt-4.1", tools)

                # 用消息调用智能体
                agent_response = await agent.ainvoke({"messages": "What can the finance agent do for me?"})
                print(agent_response)

    if __name__ == "__main__":
        asyncio.run(main())
    ```

## 会话行为

当前LangGraph MCP实现不支持会话。每个`/mcp`请求都是无状态且独立的。

## 认证

`/mcp`端点使用与LangGraph API相同的认证。设置详情请参考[认证指南](./auth.md)。

## 禁用MCP

要在`langgraph.json`配置文件中禁用MCP端点，将`disable_mcp`设为`true`：

```json
{
  "http": {
    "disable_mcp": true
  }
}
```

这将阻止服务器暴露`/mcp`端点。