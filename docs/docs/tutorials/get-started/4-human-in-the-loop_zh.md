# 添加人工介入控制

智能代理可能不可靠，有时需要人工输入才能成功完成任务。同样，对于某些操作，您可能希望在运行前要求人工批准，以确保一切按预期运行。

LangGraph的[持久化层](../../concepts/persistence.md)支持**人工介入**工作流，允许根据用户反馈暂停和恢复执行。实现这一功能的主要接口是[`interrupt`](../../how-tos/human_in_the_loop/add-human-in-the-loop.md)函数。在节点内部调用`interrupt`将暂停执行。可以通过传递[命令](../../concepts/low_level.md#command)来恢复执行，并接收来自人工的新输入。`interrupt`在易用性上类似于Python内置的`input()`函数，[但有一些注意事项](../../how-tos/human_in_the_loop/add-human-in-the-loop.md)。

!!! 注意

    本教程基于[添加记忆功能](./3-add-memory.md)构建。

## 1. 添加`human_assistance`工具

从[为聊天机器人添加记忆](./3-add-memory.md)教程的现有代码开始，向聊天机器人添加`human_assistance`工具。该工具使用`interrupt`来接收人工提供的信息。

首先选择一个聊天模型：

{!snippets/chat_model_tabs.md!}

<!---
```python
from langchain.chat_models import init_chat_model

llm = init_chat_model("anthropic:claude-3-5-sonnet-latest")
```
-->

现在我们可以将其与额外的工具一起整合到`StateGraph`中：

```python hl_lines="12 19 20 21 22 23"
from typing import Annotated

from langchain_tavily import TavilySearch
from langchain_core.tools import tool
from typing_extensions import TypedDict

from langgraph.checkpoint.memory import MemorySaver
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode, tools_condition

from langgraph.types import Command, interrupt

class State(TypedDict):
    messages: Annotated[list, add_messages]

graph_builder = StateGraph(State)

@tool
def human_assistance(query: str) -> str:
    """请求人工协助。"""
    human_response = interrupt({"query": query})
    return human_response["data"]

tool = TavilySearch(max_results=2)
tools = [tool, human_assistance]
llm_with_tools = llm.bind_tools(tools)

def chatbot(state: State):
    message = llm_with_tools.invoke(state["messages"])
    # 由于我们会在工具执行期间中断，
    # 我们禁用并行工具调用，以避免在恢复时重复任何工具调用。
    assert len(message.tool_calls) <= 1
    return {"messages": [message]}

graph_builder.add_node("chatbot", chatbot)

tool_node = ToolNode(tools=tools)
graph_builder.add_node("tools", tool_node)

graph_builder.add_conditional_edges(
    "chatbot",
    tools_condition,
)
graph_builder.add_edge("tools", "chatbot")
graph_builder.add_edge(START, "chatbot")
```

!!! 提示

    有关人工介入工作流的更多信息和示例，请参见[人工介入](../../concepts/human_in_the_loop.md)。

## 2. 编译图

像之前一样，我们使用检查器来编译图：

```python
memory = MemorySaver()

graph = graph_builder.compile(checkpointer=memory)
```

## 3. 可视化图（可选）

可视化图时，您会看到与之前相同的布局——只是添加了新的工具！

```python
from IPython.display import Image, display

try:
    display(Image(graph.get_graph().draw_mermaid_png()))
except Exception:
    # 这需要一些额外的依赖项，是可选的
    pass
```

![带工具的聊天机器人图](chatbot-with-tools.png)

## 4. 向聊天机器人提问

现在，向聊天机器人提出一个问题，这将触发新的`human_assistance`工具：

```python
user_input = "我需要一些构建AI代理的专家指导。你能为我请求协助吗？"
config = {"configurable": {"thread_id": "1"}}

events = graph.stream(
    {"messages": [{"role": "user", "content": user_input}]},
    config,
    stream_mode="values",
)
for event in events:
    if "messages" in event:
        event["messages"][-1].pretty_print()
```

```
================================ 人类消息 =================================

我需要一些构建AI代理的专家指导。你能为我请求协助吗？
================================== AI消息 ==================================

[{'text': "当然可以！我很乐意为您请求关于构建AI代理的专家协助。为此，我将使用human_assistance函数来转达您的请求。现在让我为您操作。", 'type': 'text'}, {'id': 'toolu_01ABUqneqnuHNuo1vhfDFQCW', 'input': {'query': '用户正在请求关于构建AI代理的专家指导。您能提供一些专家建议或资源吗？'}, 'name': 'human_assistance', 'type': 'tool_use'}]
工具调用:
  human_assistance (toolu_01ABUqneqnuHNuo1vhfDFQCW)
 调用ID: toolu_01ABUqneqnuHNuo1vhfDFQCW
  参数:
    query: 用户正在请求关于构建AI代理的专家指导。您能提供一些专家建议或资源吗？
```

聊天机器人生成了一个工具调用，但随后执行被中断。如果您检查图的状态，会发现它停在了工具节点：

```python
snapshot = graph.get_state(config)
snapshot.next
```

```
('tools',)
```

!!! 信息 额外信息

    仔细看看`human_assistance`工具：

    ```python
    @tool
    def human_assistance(query: str) -> str:
        """请求人工协助。"""
        human_response = interrupt({"query": query})
        return human_response["data"]
    ```

    类似于Python内置的`input()`函数，在工具内部调用`interrupt`将暂停执行。进度基于[检查器](../../concepts/persistence.md#checkpointer-libraries)持久化；因此，如果它使用Postgres持久化，只要数据库运行，就可以随时恢复。在这个例子中，它使用内存检查器持久化，只要Python内核运行，就可以随时恢复。

## 5. 恢复执行

要恢复执行，传递一个包含工具期望数据的[`命令`](../../concepts/low_level.md#command)对象。该数据的格式可以根据需要自定义。在这个例子中，使用一个带有键`"data"`的字典：

```python
human_response = (
    "我们专家在这里为您提供帮助！我们建议您查看LangGraph来构建您的代理。"
    "它比简单的自主代理更可靠和可扩展。"
)

human_command = Command(resume={"data": human_response})

events = graph.stream(human_command, config, stream_mode="values")
for event in events:
    if "messages" in event:
        event["messages"][-1].pretty_print()
```

```
================================== AI消息 ==================================

[{'text': "当然可以！我很乐意为您请求关于构建AI代理的专家协助。为此，我将使用human_assistance函数来转达您的请求。现在让我为您操作。", 'type': 'text'}, {'id': 'toolu_01ABUqneqnuHNuo1vhfDFQCW', 'input': {'query': '用户正在请求关于构建AI代理的专家指导。您能提供一些专家建议或资源吗？'}, 'name': 'human_assistance', 'type': 'tool_use'}]
工具调用:
  human_assistance (toolu_01ABUqneqnuHNuo1vhfDFQCW)
 调用ID: toolu_01ABUqneqnuHNuo1vhfDFQCW
  参数:
    query: 用户正在请求关于构建AI代理的专家指导。您能提供一些专家建议或资源吗？
================================= 工具消息 =================================
名称: human_assistance

我们专家在这里为您提供帮助！我们建议您查看LangGraph来构建您的代理。它比简单的自主代理更可靠和可扩展。
================================== AI消息 ==================================

感谢您的耐心等待。我已经收到了关于您请求构建AI代理指导的专家建议。以下是专家提出的建议：

专家建议您研究LangGraph来构建您的AI代理。他们提到，LangGraph是一个比简单自主代理更可靠和可扩展的选择。

LangGraph可能是一个专门设计用于创建具有高级功能的AI代理的框架或库。根据这个建议，以下是几点需要考虑的：

1. 可靠性：专家强调LangGraph比简单的自主代理方法更可靠。这可能意味着它具有更好的稳定性、错误处理或一致的性能。

2. 可扩展性：LangGraph被描述为更可扩展，这表明它可能提供了一个灵活的架构，允许您轻松添加新功能或修改现有功能，随着您的代理需求的发展。

3. 高级功能：鉴于它被推荐为“简单的自主代理”，LangGraph可能提供了更复杂的工具和技术来构建复杂的AI代理。
...
2. 查找专门专注于使用LangGraph构建AI代理的教程或指南。
3. 检查是否有任何社区论坛或讨论组，您可以在其中提问并获得其他使用LangGraph的开发者的支持。

如果您想了解更多关于LangGraph的具体信息，或对这个建议有任何问题，请随时提问，我可以向专家请求进一步的协助。
输出被截断。以可滚动元素查看或在文本编辑器中打开。调整单元格输出设置...
```

输入已被接收并作为工具消息处理。查看此调用的[LangSmith跟踪](https://smith.langchain.com/public/9f0f87e3-56a7-4dde-9c76-b71675624e91/r)以查看在上述调用中完成的确切工作。注意，在第一步中加载了状态，以便我们的聊天机器人可以从它停止的地方继续。

**恭喜！**您已经使用`interrupt`为您的聊天机器人添加了人工介入执行功能，允许在需要时进行人工监督和干预。这打开了您可以与AI系统创建的潜在UI。由于您已经添加了**检查器**，只要底层持久化层运行，图可以**无限期**暂停，并随时恢复，就像什么都没发生过一样。

查看下面的代码片段，回顾本教程中的图：

{!snippets/chat_model_tabs.md!}

```python
from typing import Annotated

from langchain_tavily import TavilySearch
from langchain_core.tools import tool
from typing_extensions import TypedDict

from langgraph.checkpoint.memory import MemorySaver
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode, tools_condition
from langgraph.types import Command, interrupt

class State(TypedDict):
    messages: Annotated[list, add_messages]

graph_builder = StateGraph(State)

@tool
def human_assistance(query: str) -> str:
    """请求人工协助。"""
    human_response = interrupt({"query": query})
    return human_response["data"]

tool = TavilySearch(max_results=2)
tools = [tool, human_assistance]
llm_with_tools = llm.bind_tools(tools)

def chatbot(state: State):
    message = llm_with_tools.invoke(state["messages"])
    assert(len(message.tool_calls) <= 1)
    return {"messages": [message]}

graph_builder.add_node("chatbot", chatbot)

tool_node = ToolNode(tools=tools)
graph_builder.add_node("tools", tool_node)

graph_builder.add_conditional_edges(
    "chatbot",
    tools_condition,
)
graph_builder.add_edge("tools", "chatbot")
graph_builder.add_edge(START, "chatbot")

memory = MemorySaver()
graph = graph_builder.compile(checkpointer=memory)
```

## 下一步

到目前为止，教程示例依赖于一个简单的状态，其中有一个条目：消息列表。您可以用这个简单的状态走得很远，但如果您想在不依赖消息列表的情况下定义复杂的行为，可以[向状态添加额外的字段](./5-customize-state.md)。