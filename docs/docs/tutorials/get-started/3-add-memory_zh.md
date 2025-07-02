# 添加记忆功能

目前聊天机器人已经能够[使用工具](./2-add-tools.md)来回答用户问题，但它无法记住之前的对话上下文。这限制了它进行连贯多轮对话的能力。

LangGraph通过**持久化检查点**解决了这个问题。如果在编译图(graph)时提供`checkpointer`并在调用图时提供`thread_id`，LangGraph会自动保存每一步的状态。当使用相同的`thread_id`再次调用图时，系统会加载保存的状态，让聊天机器人能从上一次中断处继续对话。

稍后我们会看到，**检查点**功能比简单的聊天记忆强大得多——它允许你随时保存和恢复复杂状态，用于错误恢复、人工介入的工作流程、时间旅行交互等场景。但现在，让我们先添加检查点功能来实现多轮对话。

!!! 注意
    本教程基于[添加工具](./2-add-tools.md)章节继续构建。

## 1. 创建`MemorySaver`检查点

创建`MemorySaver`检查点：

``` python
from langgraph.checkpoint.memory import MemorySaver

memory = MemorySaver()
```

这是一个内存检查点，适合教程使用。但在生产环境中，你可能会改用`SqliteSaver`或`PostgresSaver`并连接数据库。

## 2. 编译图

使用提供的检查点编译图，系统会在处理每个节点时对`State`进行检查点保存：

``` python
graph = graph_builder.compile(checkpointer=memory)
```

``` python
from IPython.display import Image, display

try:
    display(Image(graph.get_graph().draw_mermaid_png()))
except Exception:
    # 这需要额外依赖，是可选的
    pass
```

## 3. 与聊天机器人交互

现在可以与你的机器人对话了！

1. 选择一个线程作为本次对话的键：

    ```python
    config = {"configurable": {"thread_id": "1"}}
    ```

2. 调用你的聊天机器人：

    ```python
    user_input = "Hi there! My name is Will."

    # config是stream()或invoke()的**第二个位置参数**！
    events = graph.stream(
        {"messages": [{"role": "user", "content": user_input}]},
        config,
        stream_mode="values",
    )
    for event in events:
        event["messages"][-1].pretty_print()
    ```

    ```
    ================================ Human Message =================================

    Hi there! My name is Will.
    ================================== Ai Message ==================================

    Hello Will! It's nice to meet you. How can I assist you today? Is there anything specific you'd like to know or discuss?
    ```

    !!! 注意 
        config作为**第二个位置参数**提供给图的调用。重要的是它_没有_嵌套在图输入(`{'messages': []}`)中。

## 4. 提出后续问题

提出一个后续问题：

```python
user_input = "Remember my name?"

# config是stream()或invoke()的**第二个位置参数**！
events = graph.stream(
    {"messages": [{"role": "user", "content": user_input}]},
    config,
    stream_mode="values",
)
for event in events:
    event["messages"][-1].pretty_print()
```

```
================================ Human Message =================================

Remember my name?
================================== Ai Message ==================================

Of course, I remember your name, Will. I always try to pay attention to important details that users share with me. Is there anything else you'd like to talk about or any questions you have? I'm here to help with a wide range of topics or tasks.
```

**注意**：我们没有使用外部列表来存储记忆——一切都由检查点处理！你可以查看完整的执行过程在[LangSmith跟踪记录](https://smith.langchain.com/public/29ba22b5-6d40-4fbe-8d27-b369e3329c84/r)中。

不相信？尝试使用不同的配置：

```python
# 唯一区别是我们将`thread_id`改为"2"而不是"1"
events = graph.stream(
    {"messages": [{"role": "user", "content": user_input}]},
    # highlight-next-line
    {"configurable": {"thread_id": "2"}},
    stream_mode="values",
)
for event in events:
    event["messages"][-1].pretty_print()
```

```
================================ Human Message =================================

Remember my name?
================================== Ai Message ==================================

I apologize, but I don't have any previous context or memory of your name. As an AI assistant, I don't retain information from past conversations. Each interaction starts fresh. Could you please tell me your name so I can address you properly in this conversation?
```

**注意**：我们唯一改变的是配置中的`thread_id`。可以对比查看这次调用的[LangSmith跟踪记录](https://smith.langchain.com/public/51a62351-2f0a-4058-91cc-9996c5561428/r)。

## 5. 检查状态

到目前为止，我们已经在两个不同的线程中创建了几个检查点。但检查点中保存了什么？要检查图在任何时间点的`state`，可以调用`get_state(config)`。

```python
snapshot = graph.get_state(config)
snapshot
```

```
StateSnapshot(values={'messages': [HumanMessage(content='Hi there! My name is Will.', additional_kwargs={}, response_metadata={}, id='8c1ca919-c553-4ebf-95d4-b59a2d61e078'), AIMessage(content="Hello Will! It's nice to meet you. How can I assist you today? Is there anything specific you'd like to know or discuss?", additional_kwargs={}, response_metadata={'id': 'msg_01WTQebPhNwmMrmmWojJ9KXJ', 'model': 'claude-3-5-sonnet-20240620', 'stop_reason': 'end_turn', 'stop_sequence': None, 'usage': {'input_tokens': 405, 'output_tokens': 32}}, id='run-58587b77-8c82-41e6-8a90-d62c444a261d-0', usage_metadata={'input_tokens': 405, 'output_tokens': 32, 'total_tokens': 437}), HumanMessage(content='Remember my name?', additional_kwargs={}, response_metadata={}, id='daba7df6-ad75-4d6b-8057-745881cea1ca'), AIMessage(content="Of course, I remember your name, Will. I always try to pay attention to important details that users share with me. Is there anything else you'd like to talk about or any questions you have? I'm here to help with a wide range of topics or tasks.", additional_kwargs={}, response_metadata={'id': 'msg_01E41KitY74HpENRgXx94vag', 'model': 'claude-3-5-sonnet-20240620', 'stop_reason': 'end_turn', 'stop_sequence': None, 'usage': {'input_tokens': 444, 'output_tokens': 58}}, id='run-ffeaae5c-4d2d-4ddb-bd59-5d5cbf2a5af8-0', usage_metadata={'input_tokens': 444, 'output_tokens': 58, 'total_tokens': 502})]}, next=(), config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef7d06e-93e0-6acc-8004-f2ac846575d2'}}, metadata={'source': 'loop', 'writes': {'chatbot': {'messages': [AIMessage(content="Of course, I remember your name, Will. I always try to pay attention to important details that users share with me. Is there anything else you'd like to talk about or any questions you have? I'm here to help with a wide range of topics or tasks.", additional_kwargs={}, response_metadata={'id': 'msg_01E41KitY74HpENRgXx94vag', 'model': 'claude-3-5-sonnet-20240620', 'stop_reason': 'end_turn', 'stop_sequence': None, 'usage': {'input_tokens': 444, 'output_tokens': 58}}, id='run-ffeaae5c-4d2d-4ddb-bd59-5d5cbf2a5af8-0', usage_metadata={'input_tokens': 444, 'output_tokens': 58, 'total_tokens': 502})]}}, 'step': 4, 'parents': {}}, created_at='2024-09-27T19:30:10.820758+00:00', parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef7d06e-859f-6206-8003-e1bd3c264b8f'}}, tasks=())
```

```
snapshot.next  # (由于图在这一轮已结束，`next`为空。如果从图调用内部获取状态，next会告诉你下一步将执行哪个节点)
```

上面的快照包含当前状态值、对应配置和下一个要处理的节点。在我们的案例中，图已达到`END`状态，所以`next`为空。

**恭喜！** 你的聊天机器人现在可以借助LangGraph的检查点系统跨会话保持对话状态了。这为更自然、有上下文的交互开启了令人兴奋的可能性。LangGraph的检查点甚至能处理**任意复杂的图状态**，比简单的聊天记忆更富有表现力和强大功能。

查看下面的代码片段以回顾本教程中的图：

{!snippets/chat_model_tabs.md!}

<!---
```python
from langchain.chat_models import init_chat_model

llm = init_chat_model("anthropic:claude-3-5-sonnet-latest")
```
-->

```python hl_lines="36 37"
from typing import Annotated

from langchain.chat_models import init_chat_model
from langchain_tavily import TavilySearch
from langchain_core.messages import BaseMessage
from typing_extensions import TypedDict

from langgraph.checkpoint.memory import MemorySaver
from langgraph.graph import StateGraph
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode, tools_condition

class State(TypedDict):
    messages: Annotated[list, add_messages]

graph_builder = StateGraph(State)

tool = TavilySearch(max_results=2)
tools = [tool]
llm_with_tools = llm.bind_tools(tools)

def chatbot(state: State):
    return {"messages": [llm_with_tools.invoke(state["messages"])]}

graph_builder.add_node("chatbot", chatbot)

tool_node = ToolNode(tools=[tool])
graph_builder.add_node("tools", tool_node)

graph_builder.add_conditional_edges(
    "chatbot",
    tools_condition,
)
graph_builder.add_edge("tools", "chatbot")
graph_builder.set_entry_point("chatbot")
memory = MemorySaver()
graph = graph_builder.compile(checkpointer=memory)
```

## 下一步

在下一个教程中，你将[为聊天机器人添加人工介入功能](./4-human-in-the-loop.md)，以处理可能需要指导或验证才能继续的情况。