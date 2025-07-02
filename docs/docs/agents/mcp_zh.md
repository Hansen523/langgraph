---
search:
  boost: 2
tags:
  - agent
hide:
  - tags
---

# 使用MCP协议

[模型上下文协议(MCP)](https://modelcontextprotocol.io/introduction)是一个开放协议，标准化了应用程序向语言模型提供工具和上下文的方式。LangGraph代理可以通过`langchain-mcp-adapters`库使用MCP服务器定义的工具。

![MCP](./assets/mcp.png)

安装`langchain-mcp-adapters`库以在LangGraph中使用MCP工具：

```bash
pip install langchain-mcp-adapters
```

## 使用MCP工具

`langchain-mcp-adapters`包使代理能够使用一个或多个MCP服务器定义的工具。


=== "在代理中使用"

    ```python title="使用MCP服务器定义工具的代理"
    # highlight-next-line
    from langchain_mcp_adapters.client import MultiServerMCPClient
    from langgraph.prebuilt import create_react_agent

    # highlight-next-line
    client = MultiServerMCPClient(
        {
            "math": {
                "command": "python",
                # 替换为math_server.py文件的绝对路径
                "args": ["/path/to/math_server.py"],
                "transport": "stdio",
            },
            "weather": {
                # 确保天气服务运行在8000端口
                "url": "http://localhost:8000/mcp",
                "transport": "streamable_http",
            }
        }
    )
    # highlight-next-line
    tools = await client.get_tools()
    agent = create_react_agent(
        "anthropic:claude-3-7-sonnet-latest",
        # highlight-next-line
        tools
    )
    math_response = await agent.ainvoke(
        {"messages": [{"role": "user", "content": "what's (3 + 5) x 12?"}]}
    )
    weather_response = await agent.ainvoke(
        {"messages": [{"role": "user", "content": "what is the weather in nyc?"}]}
    )
    ```

=== "在工作流中使用"

    ```python
    from langchain_mcp_adapters.client import MultiServerMCPClient
    from langgraph.graph import StateGraph, MessagesState, START
    from langgraph.prebuilt import ToolNode, tools_condition

    from langchain.chat_models import init_chat_model
    model = init_chat_model("openai:gpt-4.1")

    client = MultiServerMCPClient(
        {
            "math": {
                "command": "python",
                # 更新为math_server.py文件的完整绝对路径
                "args": ["./examples/math_server.py"],
                "transport": "stdio",
            },
            "weather": {
                # 确保天气服务运行在8000端口
                "url": "http://localhost:8000/mcp/",
                "transport": "streamable_http",
            }
        }
    )
    tools = await client.get_tools()

    def call_model(state: MessagesState):
        response = model.bind_tools(tools).invoke(state["messages"])
        return {"messages": response}

    builder = StateGraph(MessagesState)
    builder.add_node(call_model)
    builder.add_node(ToolNode(tools))
    builder.add_edge(START, "call_model")
    builder.add_conditional_edges(
        "call_model",
        tools_condition,
    )
    builder.add_edge("tools", "call_model")
    graph = builder.compile()
    math_response = await graph.ainvoke({"messages": "what's (3 + 5) x 12?"})
    weather_response = await graph.ainvoke({"messages": "what is the weather in nyc?"})
    ```



## 自定义MCP服务器

要创建自己的MCP服务器，可以使用`mcp`库。该库提供了定义工具并将其作为服务器运行的简单方法。

安装MCP库：

```bash
pip install mcp
```
使用以下参考实现来测试带有MCP工具服务器的代理。

```python title="数学服务器示例(stdio传输)"
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("Math")

@mcp.tool()
def add(a: int, b: int) -> int:
    """两数相加"""
    return a + b

@mcp.tool()
def multiply(a: int, b: int) -> int:
    """两数相乘"""
    return a * b

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

```python title="天气服务器示例(可流式HTTP传输)"
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("Weather")

@mcp.tool()
async def get_weather(location: str) -> str:
    """获取指定位置天气"""
    return "纽约永远阳光明媚"

if __name__ == "__main__":
    mcp.run(transport="streamable-http")
```

## 更多资源

- [MCP文档](https://modelcontextprotocol.io/introduction)
- [MCP传输协议文档](https://modelcontextprotocol.io/docs/concepts/transports)
- [langchain_mcp_adapters库](https://github.com/langchain-ai/langchain-mcp-adapters)