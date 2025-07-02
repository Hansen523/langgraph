---
search:
  boost: 2
tags:
  - agent
hide:
  - tags
---

# LangGraph 快速入门

本指南展示如何使用 LangGraph 提供的**预制**、**可复用**组件，这些组件旨在帮助您快速可靠地构建智能体系统。

## 前提条件

开始教程前，请确保具备以下条件：
- 有效的 [Anthropic](https://console.anthropic.com/settings/keys) API 密钥

## 1. 安装依赖

若尚未安装，请执行以下命令安装 LangGraph 和 LangChain：

```
pip install -U langgraph "langchain[anthropic]"
```

!!! info 
    安装 LangChain 是为了让智能体能够调用 [模型](https://python.langchain.com/docs/integrations/chat/)。

## 2. 创建智能体

使用 [`create_react_agent`][langgraph.prebuilt.chat_agent_executor.create_react_agent] 创建智能体：

```python
from langgraph.prebuilt import create_react_agent

def get_weather(city: str) -> str:  # (1)!
    """获取指定城市的天气信息"""
    return f"{city}总是阳光明媚！"

agent = create_react_agent(
    model="anthropic:claude-3-7-sonnet-latest",  # (2)!
    tools=[get_weather],  # (3)!
    prompt="你是一个乐于助人的助手"  # (4)!
)

# 运行智能体
agent.invoke(
    {"messages": [{"role": "user", "content": "旧金山的天气怎么样"}]}
)
```

1. 定义智能体使用的工具。工具可以是普通 Python 函数。更高级的工具用法请参考 [工具](./tools.md) 页面。
2. 指定智能体使用的语言模型。配置详情请参阅 [模型](./models.md)。
3. 提供工具列表。
4. 设置语言模型的系统提示（指令）。

## 3. 配置大语言模型

使用 [init_chat_model](https://python.langchain.com/api_reference/langchain/chat_models/langchain.chat_models.base.init_chat_model.html) 配置模型参数（如温度系数）：

```python
from langchain.chat_models import init_chat_model
from langgraph.prebuilt import create_react_agent

# highlight-next-line
model = init_chat_model(
    "anthropic:claude-3-7-sonnet-latest",
    # highlight-next-line
    temperature=0
)

agent = create_react_agent(
    # highlight-next-line
    model=model,
    tools=[get_weather],
)
```

更多配置说明请参考 [模型](./models.md)。

## 4. 添加自定义提示

提示词用于指导大语言模型的行为，可添加以下类型：

* **静态提示**：固定字符串作为**系统消息**
* **动态提示**：基于**运行时**输入或配置生成的消息列表

=== "静态提示"

    定义固定字符串或消息列表：

    ```python
    from langgraph.prebuilt import create_react_agent

    agent = create_react_agent(
        model="anthropic:claude-3-7-sonnet-latest",
        tools=[get_weather],
        # 永不改变的静态提示
        # highlight-next-line
        prompt="永远不要回答关于天气的问题"
    )

    agent.invoke(
        {"messages": [{"role": "user", "content": "旧金山的天气怎么样"}]}
    )
    ```

=== "动态提示"

    根据状态和配置动态生成消息列表：

    ```python
    from langchain_core.messages import AnyMessage
    from langchain_core.runnables import RunnableConfig
    from langgraph.prebuilt.chat_agent_executor import AgentState
    from langgraph.prebuilt import create_react_agent

    # highlight-next-line
    def prompt(state: AgentState, config: RunnableConfig) -> list[AnyMessage]:  # (1)!
        user_name = config["configurable"].get("user_name")
        system_msg = f"你是一个乐于助人的助手。请称呼用户为{user_name}。"
        return [{"role": "system", "content": system_msg}] + state["messages"]

    agent = create_react_agent(
        model="anthropic:claude-3-7-sonnet-latest",
        tools=[get_weather],
        # highlight-next-line
        prompt=prompt
    )

    agent.invoke(
        {"messages": [{"role": "user", "content": "旧金山的天气怎么样"}]},
        # highlight-next-line
        config={"configurable": {"user_name": "张三"}}
    )
    ```

    1. 动态提示允许包含非消息的 [上下文](./context.md)，例如：
        - 运行时传递的信息（如`user_id`或API凭证，通过`config`）
        - 多步推理过程中更新的智能体状态（通过`state`）
        动态提示可以是接收`state`和`config`并返回消息列表的函数。

更多信息请参阅 [上下文](./context.md)。

## 5. 添加记忆功能

要实现多轮对话，需通过 [持久化](../concepts/persistence.md) 功能提供`checkpointer`，并在运行时配置包含会话ID（`thread_id`）的config：

```python
from langgraph.prebuilt import create_react_agent
from langgraph.checkpoint.memory import InMemorySaver

# highlight-next-line
checkpointer = InMemorySaver()

agent = create_react_agent(
    model="anthropic:claude-3-7-sonnet-latest",
    tools=[get_weather],
    # highlight-next-line
    checkpointer=checkpointer  # (1)!
)

# 运行智能体
# highlight-next-line
config = {"configurable": {"thread_id": "1"}}
sf_response = agent.invoke(
    {"messages": [{"role": "user", "content": "旧金山的天气怎么样"}]},
    # highlight-next-line
    config  # (2)!
)
ny_response = agent.invoke(
    {"messages": [{"role": "user", "content": "纽约的天气呢？"}]},
    # highlight-next-line
    config
)
```

1. `checkpointer`允许存储工具调用循环的每一步状态，支持 [短期记忆](../how-tos/memory/add-memory.md#add-short-term-memory) 和 [人工干预](../concepts/human_in_the_loop.md) 功能。
2. 相同的`thread_id`可恢复之前对话。

记忆功能会将智能体状态存储在检查点数据库（使用`InMemorySaver`时存储在内存中）。

注意第二个调用使用相同`thread_id`时，会自动包含首次对话的历史消息。

更多信息请参考 [记忆功能](../how-tos/memory/add-memory.md)。

## 6. 配置结构化输出

使用`response_format`参数生成符合模式的结构化响应（可用`Pydantic`模型或`TypedDict`定义），结果将通过`structured_response`字段访问：

```python
from pydantic import BaseModel
from langgraph.prebuilt import create_react_agent

class WeatherResponse(BaseModel):
    conditions: str

agent = create_react_agent(
    model="anthropic:claude-3-7-sonnet-latest",
    tools=[get_weather],
    # highlight-next-line
    response_format=WeatherResponse  # (1)!
)

response = agent.invoke(
    {"messages": [{"role": "user", "content": "旧金山的天气怎么样"}]}
)

# highlight-next-line
response["structured_response"]
```

1. 提供`response_format`时，智能体循环的最后会增加一个步骤：将消息历史传递给LLM生成结构化响应。
   如需为此LLM提供系统提示，可使用元组`(prompt, schema)`，例如`response_format=(prompt, WeatherResponse)`。

!!! 注意 "LLM后处理"
    结构化输出需要额外调用LLM来格式化响应。

## 后续步骤
- [本地部署智能体](../tutorials/langgraph-platform/local-server.md)
- [了解预制智能体](../agents/overview.md)
- [LangGraph平台快速入门](../cloud/quick_start.md)