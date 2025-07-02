# 上下文

**上下文工程**是一种构建动态系统的实践，该系统以正确的格式提供正确的信息和工具，使语言模型能够合理完成任务。

上下文包括任何可以塑造行为、不在消息列表之外的数据。这可以是：

- 运行时传递的信息，如`user_id`或API凭证
- 在多步推理过程中更新的内部状态
- 来自先前交互的持久性记忆或事实

LangGraph提供了**三种**主要的上下文提供方式：

| 类型                                                                         | 描述                                   | 可变性？ | 生命周期                |
|------------------------------------------------------------------------------|-----------------------------------------------|----------|-------------------------|
| [**配置**](#config-static-context)                                         | 运行开始时传递的数据             | ❌        | 每次运行                 |
| [**短期记忆（状态）**](#short-term-memory-mutable-context)          | 执行过程中可能变化的动态数据 | ✅        | 每次运行或会话 |
| [**长期记忆（存储）**](#long-term-memory-cross-conversation-context) | 可在不同会话间共享的数据       | ✅        | 跨会话持久    |

## 提供运行时上下文

### 配置（静态上下文）

配置用于不可变数据，如用户元数据或API密钥。当您有在运行过程中不会改变的值时使用。

使用名为**"configurable"**的键来指定配置，该键专门用于此目的：

```python
graph.invoke( # (1)!
    {"messages": [{"role": "user", "content": "hi!"}]}, # (2)!
    # highlight-next-line
    config={"configurable": {"user_id": "user_123"}} # (3)!
)
```

1. 这是对代理或图的调用。`invoke`方法用提供的输入运行底层图。
2. 此示例使用消息作为输入，这很常见，但您的应用程序可能使用不同的输入结构。
3. 这是传递配置数据的地方。`config`参数允许您提供代理在执行过程中可以使用的额外上下文。

=== "代理提示"

    ```python
    from langchain_core.messages import AnyMessage
    from langchain_core.runnables import RunnableConfig
    from langgraph.prebuilt.chat_agent_executor import AgentState
    from langgraph.prebuilt import create_react_agent

    # highlight-next-line
    def prompt(state: AgentState, config: RunnableConfig) -> list[AnyMessage]:
        user_name = config["configurable"].get("user_name")
        system_msg = f"You are a helpful assistant. Address the user as {user_name}."
        return [{"role": "system", "content": system_msg}] + state["messages"]

    agent = create_react_agent(
        model="anthropic:claude-3-7-sonnet-latest",
        tools=[get_weather],
        prompt=prompt
    )

    agent.invoke(
        {"messages": [{"role": "user", "content": "what is the weather in sf"}]},
        # highlight-next-line
        config={"configurable": {"user_name": "John Smith"}}
    )
    ```

    * 详见[代理](../agents/agents.md)部分。

=== "工作流节点"

    ```python
    from langchain_core.runnables import RunnableConfig

    # highlight-next-line
    def node(state: State, config: RunnableConfig):
        user_name = config["configurable"].get("user_name")
        ...
    ```

    * 详见[图API](https://langchain-ai.github.io/langgraph/how-tos/graph-api/#add-runtime-configuration)。

=== "在工具中"

    ```python
    from langchain_core.runnables import RunnableConfig

    @tool
    # highlight-next-line
    def get_user_info(config: RunnableConfig) -> str:
        """根据用户ID检索用户信息。"""
        user_id = config["configurable"].get("user_id")
        return "User is John Smith" if user_id == "user_123" else "Unknown user"
    ```

    详见[工具调用指南](../how-tos/tool-calling.md#configuration)。

### 短期记忆（可变上下文）

状态作为运行期间的[短期记忆](../concepts/memory.md)。它保存执行过程中可能演变的动态数据，如来自工具或LLM输出的值。

=== "在代理中"

    示例展示如何将状态纳入代理**提示**。

    状态也可以被代理的**工具**访问，这些工具可以按需读取或更新状态。详见[工具调用指南](../how-tos/tool-calling.md#short-term-memory)。

    ```python
    from langchain_core.messages import AnyMessage
    from langchain_core.runnables import RunnableConfig
    from langgraph.prebuilt import create_react_agent
    from langgraph.prebuilt.chat_agent_executor import AgentState

    # highlight-next-line
    class CustomState(AgentState): # (1)!
        user_name: str

    def prompt(
        # highlight-next-line
        state: CustomState
    ) -> list[AnyMessage]:
        user_name = state["user_name"]
        system_msg = f"You are a helpful assistant. User's name is {user_name}"
        return [{"role": "system", "content": system_msg}] + state["messages"]

    agent = create_react_agent(
        model="anthropic:claude-3-7-sonnet-latest",
        tools=[...],
        # highlight-next-line
        state_schema=CustomState, # (2)!
        prompt=prompt
    )

    agent.invoke({
        "messages": "hi!",
        "user_name": "John Smith"
    })
    ```

    1. 定义一个扩展`AgentState`或`MessagesState`的自定义状态模式。
    2. 将自定义状态模式传递给代理。允许代理在执行过程中访问和修改状态。

=== "在工作流中"

    ```python
    from typing_extensions import TypedDict
    from langchain_core.messages import AnyMessage
    from langgraph.graph import StateGraph

    # highlight-next-line
    class CustomState(TypedDict): # (1)!
        messages: list[AnyMessage]
        extra_field: int

    # highlight-next-line
    def node(state: CustomState): # (2)!
        messages = state["messages"]
        ...
        return { # (3)!
            # highlight-next-line
            "extra_field": state["extra_field"] + 1
        }

    builder = StateGraph(State)
    builder.add_node(node)
    builder.set_entry_point("node")
    graph = builder.compile()
    ```
    
    1. 定义自定义状态
    2. 在任何节点或工具中访问状态
    3. 图API设计为尽可能轻松地与状态配合工作。节点的返回值代表对状态的请求更新。

!!! tip "启用记忆"

    更多关于如何启用记忆的详细信息，请参见[记忆指南](../how-tos/memory/add-memory.md)。这是一个强大的功能，允许您在多次调用之间保持代理状态。否则，状态仅限定于单次运行。

### 长期记忆（跨会话上下文）

对于跨越*多个*会话或对话的上下文，LangGraph允许通过`store`访问**长期记忆**。这可用于读取或更新持久性事实（如用户档案、偏好、先前交互）。

更多信息，请参见[记忆指南](../how-tos/memory/add-memory.md)。