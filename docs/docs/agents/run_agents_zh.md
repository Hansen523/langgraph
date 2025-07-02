---
search:
  boost: 2
tags:
  - agent
hide:
  - tags
---

# 运行智能体


智能体支持同步和异步两种执行方式，可以使用`.invoke()`/`await .ainvoke()`获取完整响应，或使用`.stream()`/`.astream()`获取**增量式**[流式输出](../how-tos/streaming.md)。本节将介绍如何提供输入、解析输出、启用流式传输以及控制执行限制。


## 基础用法

智能体主要有两种执行模式：

- **同步模式**：使用`.invoke()`或`.stream()`
- **异步模式**：使用`await .ainvoke()`或`async for`配合`.astream()`

=== "同步调用"
    ```python
    from langgraph.prebuilt import create_react_agent

    agent = create_react_agent(...)

    # highlight-next-line
    response = agent.invoke({"messages": [{"role": "user", "content": "what is the weather in sf"}]})
    ```

=== "异步调用"
    ```python
    from langgraph.prebuilt import create_react_agent

    agent = create_react_agent(...)
    # highlight-next-line
    response = await agent.ainvoke({"messages": [{"role": "user", "content": "what is the weather in sf"}]})
    ```

## 输入与输出

智能体使用的语言模型要求输入为消息列表。因此，智能体的输入输出都以`messages`键存储在[状态](../concepts/low_level.md#working-with-messages-in-graph-state)中。

## 输入格式

智能体输入必须是包含`messages`键的字典。支持的格式包括：

| 格式              | 示例                                                                                                                       |
|-------------------|-------------------------------------------------------------------------------------------------------------------------------|
| 字符串            | `{"messages": "Hello"}` —— 将被解析为[HumanMessage](https://python.langchain.com/docs/concepts/messages/#humanmessage) |
| 消息字典          | `{"messages": {"role": "user", "content": "Hello"}}`                                                                          |
| 消息列表          | `{"messages": [{"role": "user", "content": "Hello"}]}`                                                                        |
| 自定义状态        | `{"messages": [{"role": "user", "content": "Hello"}], "user_name": "Alice"}` —— 使用自定义`state_schema`时适用               |

消息会自动转换为LangChain的内部消息格式。更多关于[LangChain消息](https://python.langchain.com/docs/concepts/messages/#langchain-messages)的信息，请参阅LangChain文档。

!!! tip "使用自定义智能体状态"

    您可以在输入字典中直接提供智能体状态模式定义的额外字段。这允许基于运行时数据或先前的工具输出来实现动态行为。  
    详见[上下文指南](./context.md)。

!!! note

    当`messages`输入为字符串时，会被转换为[HumanMessage](https://python.langchain.com/docs/concepts/messages/#humanmessage)。这与`create_react_agent`中的`prompt`参数不同，后者作为字符串输入时会被解析为[SystemMessage](https://python.langchain.com/docs/concepts/messages/#systemmessage)。


## 输出格式

智能体输出包含以下内容：

- `messages`：执行过程中交换的所有消息列表（用户输入、助手回复、工具调用）。
- 如果配置了[结构化输出](./agents.md#6-configure-structured-output)，可选包含`structured_response`。
- 如果使用自定义`state_schema`，输出中可能还会包含与您定义的字段对应的其他键。这些可以保存来自工具执行或提示逻辑的更新状态值。

关于使用自定义状态模式和处理上下文的更多细节，请参阅[上下文指南](./context.md)。

## 流式输出

智能体支持流式响应，以提升应用响应速度。包括：

- 每步执行后的**进度更新**
- 生成的**LLM令牌**
- 执行过程中的**自定义工具消息**

流式传输支持同步和异步模式：

=== "同步流式"

    ```python
    for chunk in agent.stream(
        {"messages": [{"role": "user", "content": "what is the weather in sf"}]},
        stream_mode="updates"
    ):
        print(chunk)
    ```

=== "异步流式"

    ```python
    async for chunk in agent.astream(
        {"messages": [{"role": "user", "content": "what is the weather in sf"}]},
        stream_mode="updates"
    ):
        print(chunk)
    ```

!!! tip

    完整细节请参阅[流式指南](../how-tos/streaming.md)。

## 最大迭代次数

为防止无限循环，可以设置递归限制来定义智能体在抛出`GraphRecursionError`前能执行的最大步骤数。您可以在运行时或通过`.with_config()`定义智能体时配置`recursion_limit`：

=== "运行时配置"

    ```python
    from langgraph.errors import GraphRecursionError
    from langgraph.prebuilt import create_react_agent

    max_iterations = 3
    # highlight-next-line
    recursion_limit = 2 * max_iterations + 1
    agent = create_react_agent(
        model="anthropic:claude-3-5-haiku-latest",
        tools=[get_weather]
    )

    try:
        response = agent.invoke(
            {"messages": [{"role": "user", "content": "what's the weather in sf"}]},
            # highlight-next-line
            {"recursion_limit": recursion_limit},
        )
    except GraphRecursionError:
        print("Agent stopped due to max iterations.")
    ```

=== "`.with_config()`配置"

    ```python
    from langgraph.errors import GraphRecursionError
    from langgraph.prebuilt import create_react_agent

    max_iterations = 3
    # highlight-next-line
    recursion_limit = 2 * max_iterations + 1
    agent = create_react_agent(
        model="anthropic:claude-3-5-haiku-latest",
        tools=[get_weather]
    )
    # highlight-next-line
    agent_with_recursion_limit = agent.with_config(recursion_limit=recursion_limit)

    try:
        response = agent_with_recursion_limit.invoke(
            {"messages": [{"role": "user", "content": "what's the weather in sf"}]},
        )
    except GraphRecursionError:
        print("Agent stopped due to max iterations.")
    ```

## 更多资源

* [LangChain中的异步编程](https://python.langchain.com/docs/concepts/async)