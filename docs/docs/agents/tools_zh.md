---
search:
  boost: 2  
tags:
  - agent
hide:
  - tags
---

# 工具

[工具](https://python.langchain.com/docs/concepts/tools/)是一种封装函数及其输入模式的方式，可以传递给支持工具调用的聊天模型。这使得模型能够请求执行该函数并传入特定参数。

您既可以[自定义工具](#define-simple-tools)，也可以使用LangChain提供的[预集成工具](#prebuilt-tools)。

## 定义简单工具

您可以将普通函数传递给`create_react_agent`作为工具使用：

```python
from langgraph.prebuilt import create_react_agent

def multiply(a: int, b: int) -> int:
    """乘法运算"""
    return a * b

create_react_agent(
    model="anthropic:claude-3-7-sonnet",
    tools=[multiply]
)
```

`create_react_agent`会自动将普通函数转换为[LangChain工具](https://python.langchain.com/docs/concepts/tools/#tool-interface)。

## 自定义工具

如需更精细控制工具行为，可使用`@tool`装饰器：

```python
# highlight-next-line
from langchain_core.tools import tool

# highlight-next-line
@tool("multiply_tool", parse_docstring=True)
def multiply(a: int, b: int) -> int:
    """两数相乘

    参数：
        a: 被乘数
        b: 乘数
    """
    return a * b
```

也可使用Pydantic自定义输入模式：

```python
from pydantic import BaseModel, Field

class MultiplyInputSchema(BaseModel):
    """乘法运算输入模式"""
    a: int = Field(description="被乘数")
    b: int = Field(description="乘数")

# highlight-next-line
@tool("multiply_tool", args_schema=MultiplyInputSchema)
def multiply(a: int, b: int) -> int:
    return a * b
```

更多自定义选项请参考[自定义工具指南](https://python.langchain.com/docs/how_to/custom_tools/)。

## 隐藏模型不可控参数

某些工具需要运行时参数（如用户ID或会话上下文），这些参数不应由模型控制。

可将此类参数放入代理的`state`或`config`中，在工具内部访问：

```python
from langgraph.prebuilt import InjectedState
from langgraph.prebuilt.chat_agent_executor import AgentState
from langchain_core.runnables import RunnableConfig

def my_tool(
    # 由LLM填充
    tool_arg: str,
    # 访问代理中的动态信息
    # highlight-next-line
    state: Annotated[AgentState, InjectedState],
    # 访问代理调用时传入的静态数据
    # highlight-next-line
    config: RunnableConfig,
) -> str:
    """自定义工具"""
    do_something_with_state(state["messages"])
    do_something_with_config(config)
    ...
```

## 禁用并行工具调用

部分模型提供商支持并行执行多个工具，但允许用户禁用此功能。

对于支持的提供商，可通过`model.bind_tools()`设置`parallel_tool_calls=False`来禁用：

```python
from langchain.chat_models import init_chat_model

def add(a: int, b: int) -> int:
    """加法运算"""
    return a + b

def multiply(a: int, b: int) -> int:
    """乘法运算"""
    return a * b

model = init_chat_model("anthropic:claude-3-5-sonnet-latest", temperature=0)
tools = [add, multiply]
agent = create_react_agent(
    # 禁用并行工具调用
    # highlight-next-line
    model=model.bind_tools(tools, parallel_tool_calls=False),
    tools=tools
)

agent.invoke(
    {"messages": [{"role": "user", "content": "计算3+5和4*7"}]}
)
```

## 直接返回工具结果

设置`return_direct=True`可立即返回工具结果并终止代理循环：

```python
from langchain_core.tools import tool

# highlight-next-line
@tool(return_direct=True)
def add(a: int, b: int) -> int:
    """加法运算"""
    return a + b

agent = create_react_agent(
    model="anthropic:claude-3-7-sonnet-latest",
    tools=[add]
)

agent.invoke(
    {"messages": [{"role": "user", "content": "3+5等于多少？"}]}
)
```

## 强制使用工具

通过`model.bind_tools()`中的`tool_choice`参数可强制代理使用特定工具：

```python
from langchain_core.tools import tool

# highlight-next-line
@tool(return_direct=True)
def greet(user_name: str) -> int:
    """问候用户"""
    return f"你好{user_name}！"

tools = [greet]

agent = create_react_agent(
    # highlight-next-line
    model=model.bind_tools(tools, tool_choice={"type": "tool", "name": "greet"}),
    tools=tools
)

agent.invoke(
    {"messages": [{"role": "user", "content": "我是Bob"}]}
)
```

!!! 警告 "避免无限循环"

    强制使用工具时需设置停止条件，否则可能导致无限循环。安全措施包括：
    
    - 使用[`return_direct=True`](#return-tool-results-directly)在执行后终止循环
    - 设置[`recursion_limit`](../concepts/low_level.md#recursion-limit)限制执行步数

## 处理工具错误

默认情况下，代理会捕获工具调用中的所有异常，并将错误信息传递给LLM。通过[`ToolNode`][langgraph.prebuilt.tool_node.ToolNode]的`handle_tool_errors`参数可自定义错误处理方式：

=== "启用错误处理（默认）"

    ```python
    from langgraph.prebuilt import create_react_agent

    def multiply(a: int, b: int) -> int:
        """乘法运算"""
        if a == 42:
            raise ValueError("终极错误")
        return a * b

    # 默认启用错误处理
    agent = create_react_agent(
        model="anthropic:claude-3-7-sonnet-latest",
        tools=[multiply]
    )
    agent.invoke(
        {"messages": [{"role": "user", "content": "42乘以7是多少？"}]}
    )
    ```

=== "禁用错误处理"

    ```python
    from langgraph.prebuilt import create_react_agent, ToolNode

    def multiply(a: int, b: int) -> int:
        """乘法运算"""
        if a == 42:
            raise ValueError("终极错误")
        return a * b

    # highlight-next-line
    tool_node = ToolNode(
        [multiply],
        # highlight-next-line
        handle_tool_errors=False  # (1)!
    )
    agent_no_error_handling = create_react_agent(
        model="anthropic:claude-3-7-sonnet-latest",
        tools=tool_node
    )
    agent_no_error_handling.invoke(
        {"messages": [{"role": "user", "content": "42乘以7是多少？"}]}
    )
    ```

    1. 禁用默认的错误处理机制，完整策略参见[API文档][langgraph.prebuilt.tool_node.ToolNode]

=== "自定义错误处理"

    ```python
    from langgraph.prebuilt import create_react_agent, ToolNode

    def multiply(a: int, b: int) -> int:
        """乘法运算"""
        if a == 42:
            raise ValueError("终极错误")
        return a * b

    # highlight-next-line
    tool_node = ToolNode(
        [multiply],
        # highlight-next-line
        handle_tool_errors="禁止使用42作为被乘数，请调换运算数！"  # (1)!
    )
    agent_custom_error_handling = create_react_agent(
        model="anthropic:claude-3-7-sonnet-latest",
        tools=tool_node
    )
    agent_custom_error_handling.invoke(
        {"messages": [{"role": "user", "content": "42乘以7是多少？"}]}
    )
    ```

    1. 发生异常时向LLM发送自定义消息，完整策略参见[API文档][langgraph.prebuilt.tool_node.ToolNode]

更多工具错误处理选项详见[API文档][langgraph.prebuilt.tool_node.ToolNode]

## 内存管理

LangGraph支持从工具中访问短期和长期记忆。关于记忆管理的详细指南请参考：
* [读取](../how-tos/memory/add-memory.md#read-short-term)和[写入](../how-tos/memory/add-memory.md#write-short-term)短期记忆
* [读取](../how-tos/memory/add-memory.md#read-long-term)和[写入](../how-tos/memory/add-memory.md#write-long-term)长期记忆

## 预集成工具

通过向`create_react_agent`的`tools`参数传递工具规格字典，可以使用模型提供商预置的工具。例如使用OpenAI的`web_search_preview`工具：

```python
from langgraph.prebuilt import create_react_agent

agent = create_react_agent(
    model="openai:gpt-4o-mini", 
    tools=[{"type": "web_search_preview"}]
)
response = agent.invoke(
    {"messages": ["今天有什么正能量新闻？"]}
)
```

LangChain还提供丰富的预集成工具，涵盖API调用、数据库操作、文件系统、网络数据等场景，可快速扩展代理功能。

完整工具列表请浏览[LangChain集成目录](https://python.langchain.com/docs/integrations/tools/)

常用工具类别包括：
- **搜索**：Bing、SerpAPI、Tavily
- **代码解释器**：Python REPL、Node.js REPL
- **数据库**：SQL、MongoDB、Redis
- **网络数据**：网页抓取与浏览
- **API**：OpenWeatherMap、NewsAPI等

这些集成工具均可通过上述示例中的`tools`参数配置使用。