# 自定义状态

在本教程中，您将为状态添加额外字段，以便在不依赖消息列表的情况下定义复杂行为。聊天机器人将使用其搜索工具查找特定信息，并将其转发给人类进行审核。

!!! note
    本教程基于[添加人工审核控制](./4-human-in-the-loop.md)构建。

## 1. 向状态添加键值

通过添加`name`和`birthday`键到状态中，更新聊天机器人以研究实体的生日：

```python
from typing import Annotated

from typing_extensions import TypedDict

from langgraph.graph.message import add_messages


class State(TypedDict):
    messages: Annotated[list, add_messages]
    # highlight-next-line
    name: str
    # highlight-next-line
    birthday: str
```

将这些信息添加到状态中，使得其他图节点（如下游存储或处理信息的节点）以及图的持久层可以轻松访问。

## 2. 在工具内更新状态

现在，在`human_assistance`工具内填充状态键值。这使得人类可以在信息存储到状态之前进行审核。使用[`Command`](../../concepts/low_level.md#using-inside-tools)从工具内部发出状态更新。

``` python
from langchain_core.messages import ToolMessage
from langchain_core.tools import InjectedToolCallId, tool

from langgraph.types import Command, interrupt

@tool
# 注意，因为我们正在为状态更新生成ToolMessage，
# 通常需要对应工具调用的ID。我们可以使用
# LangChain的InjectedToolCallId来表示这个参数
# 不应在工具的模式中向模型展示。
def human_assistance(
    name: str, birthday: str, tool_call_id: Annotated[str, InjectedToolCallId]
) -> str:
    """请求人类协助。"""
    human_response = interrupt(
        {
            "question": "Is this correct?",
            "name": name,
            "birthday": birthday,
        },
    )
    # 如果信息正确，按原样更新状态。
    if human_response.get("correct", "").lower().startswith("y"):
        verified_name = name
        verified_birthday = birthday
        response = "Correct"
    # 否则，从人类审核者处接收信息。
    else:
        verified_name = human_response.get("name", name)
        verified_birthday = human_response.get("birthday", birthday)
        response = f"Made a correction: {human_response}"

    # 这次我们在工具内明确用ToolMessage更新状态。
    state_update = {
        "name": verified_name,
        "birthday": verified_birthday,
        "messages": [ToolMessage(response, tool_call_id=tool_call_id)],
    }
    # 我们在工具中返回一个Command对象来更新状态。
    return Command(update=state_update)
```

图的其余部分保持不变。

## 3. 提示聊天机器人

提示聊天机器人查找LangGraph库的“生日”，并在获得所需信息后引导其使用`human_assistance`工具进行审核。通过在工具的参数中设置`name`和`birthday`，强制聊天机器人生成这些字段的提案。

```python
user_input = (
    "Can you look up when LangGraph was released? "
    "When you have the answer, use the human_assistance tool for review."
)
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
================================ Human Message =================================

Can you look up when LangGraph was released? When you have the answer, use the human_assistance tool for review.
================================== Ai Message ==================================

[{'text': "Certainly! I'll start by searching for information about LangGraph's release date using the Tavily search function. Then, I'll use the human_assistance tool for review.", 'type': 'text'}, {'id': 'toolu_01JoXQPgTVJXiuma8xMVwqAi', 'input': {'query': 'LangGraph release date'}, 'name': 'tavily_search_results_json', 'type': 'tool_use'}]
Tool Calls:
  tavily_search_results_json (toolu_01JoXQPgTVJXiuma8xMVwqAi)
 Call ID: toolu_01JoXQPgTVJXiuma8xMVwqAi
  Args:
    query: LangGraph release date
================================= Tool Message =================================
Name: tavily_search_results_json

[{"url": "https://blog.langchain.dev/langgraph-cloud/", "content": "We also have a new stable release of LangGraph. By LangChain 6 min read Jun 27, 2024 (Oct '24) Edit: Since the launch of LangGraph Platform, we now have multiple deployment options alongside LangGraph Studio - which now fall under LangGraph Platform. LangGraph Platform is synonymous with our Cloud SaaS deployment option."}, {"url": "https://changelog.langchain.com/announcements/langgraph-cloud-deploy-at-scale-monitor-carefully-iterate-boldly", "content": "LangChain - Changelog | ☁ 🚀 LangGraph Platform: Deploy at scale, monitor LangChain LangSmith LangGraph LangChain LangSmith LangGraph LangChain LangSmith LangGraph LangChain Changelog Sign up for our newsletter to stay up to date DATE: The LangChain Team LangGraph LangGraph Platform ☁ 🚀 LangGraph Platform: Deploy at scale, monitor carefully, iterate boldly DATE: June 27, 2024 AUTHOR: The LangChain Team LangGraph Platform is now in closed beta, offering scalable, fault-tolerant deployment for LangGraph agents. LangGraph Platform also includes a new playground-like studio for debugging agent failure modes and quick iteration: Join the waitlist today for LangGraph Platform. And to learn more, read our blog post announcement or check out our docs. Subscribe By clicking subscribe, you accept our privacy policy and terms and conditions."}]
================================== Ai Message ==================================

[{'text': "Based on the search results, it appears that LangGraph was already in existence before June 27, 2024, when LangGraph Platform was announced. However, the search results don't provide a specific release date for the original LangGraph. \n\nGiven this information, I'll use the human_assistance tool to review and potentially provide more accurate information about LangGraph's initial release date.", 'type': 'text'}, {'id': 'toolu_01JDQAV7nPqMkHHhNs3j3XoN', 'input': {'name': 'Assistant', 'birthday': '2023-01-01'}, 'name': 'human_assistance', 'type': 'tool_use'}]
Tool Calls:
  human_assistance (toolu_01JDQAV7nPqMkHHhNs3j3XoN)
 Call ID: toolu_01JDQAV7nPqMkHHhNs3j3XoN
  Args:
    name: Assistant
    birthday: 2023-01-01
```

我们再次在`human_assistance`工具中遇到了`interrupt`。

## 4. 添加人工协助

聊天机器人未能识别正确的日期，因此为其提供信息：

```python
human_command = Command(
    resume={
        "name": "LangGraph",
        "birthday": "Jan 17, 2024",
    },
)

events = graph.stream(human_command, config, stream_mode="values")
for event in events:
    if "messages" in event:
        event["messages"][-1].pretty_print()
```

```
================================== Ai Message ==================================

[{'text': "Based on the search results, it appears that LangGraph was already in existence before June 27, 2024, when LangGraph Platform was announced. However, the search results don't provide a specific release date for the original LangGraph. \n\nGiven this information, I'll use the human_assistance tool to review and potentially provide more accurate information about LangGraph's initial release date.", 'type': 'text'}, {'id': 'toolu_01JDQAV7nPqMkHHhNs3j3XoN', 'input': {'name': 'Assistant', 'birthday': '2023-01-01'}, 'name': 'human_assistance', 'type': 'tool_use'}]
Tool Calls:
  human_assistance (toolu_01JDQAV7nPqMkHHhNs3j3XoN)
 Call ID: toolu_01JDQAV7nPqMkHHhNs3j3XoN
  Args:
    name: Assistant
    birthday: 2023-01-01
================================= Tool Message =================================
Name: human_assistance

Made a correction: {'name': 'LangGraph', 'birthday': 'Jan 17, 2024'}
================================== Ai Message ==================================

Thank you for the human assistance. I can now provide you with the correct information about LangGraph's release date.

LangGraph was initially released on January 17, 2024. This information comes from the human assistance correction, which is more accurate than the search results I initially found.

To summarize:
1. LangGraph's original release date: January 17, 2024
2. LangGraph Platform announcement: June 27, 2024

It's worth noting that LangGraph had been in development and use for some time before the LangGraph Platform announcement, but the official initial release of LangGraph itself was on January 17, 2024.
```

注意，这些字段现在反映在状态中：

```python
snapshot = graph.get_state(config)

{k: v for k, v in snapshot.values.items() if k in ("name", "birthday")}
```

```
{'name': 'LangGraph', 'birthday': 'Jan 17, 2024'}
```

这使得它们对下游节点（如进一步处理或存储信息的节点）易于访问。

## 5. 手动更新状态

LangGraph提供了对应用程序状态的高度控制。例如，在任何时候（包括中断时），您可以使用`graph.update_state`手动覆盖键值：

``` python
graph.update_state(config, {"name": "LangGraph (library)"})
```

```
{'configurable': {'thread_id': '1',
  'checkpoint_ns': '',
  'checkpoint_id': '1efd4ec5-cf69-6352-8006-9278f1730162'}}
```

## 6. 查看新值

如果您调用`graph.get_state`，可以看到新值已反映：

``` python
snapshot = graph.get_state(config)

{k: v for k, v in snapshot.values.items() if k in ("name", "birthday")}
```

```
{'name': 'LangGraph (library)', 'birthday': 'Jan 17, 2024'}
```

手动状态更新将在LangSmith中[生成跟踪记录](https://smith.langchain.com/public/7ebb7827-378d-49fe-9f6c-5df0e90086c8/r)。如果需要，它们也可以用于[控制人工审核工作流](../../how-tos/human_in_the_loop/add-human-in-the-loop.md)。通常建议使用`interrupt`函数，因为它允许在人工审核交互中独立于状态更新传输数据。

**恭喜！** 您已向状态添加了自定义键值以支持更复杂的工作流，并学会了从工具内部生成状态更新。

查看下面的代码片段以回顾本教程中的图：

{!snippets/chat_model_tabs.md!}

<!---
```python
from langchain.chat_models import init_chat_model

llm = init_chat_model("anthropic:claude-3-5-sonnet-latest")
```
-->

```python
from typing import Annotated

from langchain_tavily import TavilySearch
from langchain_core.messages import ToolMessage
from langchain_core.tools import InjectedToolCallId, tool
from typing_extensions import TypedDict

from langgraph.checkpoint.memory import MemorySaver
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode, tools_condition
from langgraph.types import Command, interrupt

class State(TypedDict):
    messages: Annotated[list, add_messages]
    name: str
    birthday: str

@tool
def human_assistance(
    name: str, birthday: str, tool_call_id: Annotated[str, InjectedToolCallId]
) -> str:
    """Request assistance from a human."""
    human_response = interrupt(
        {
            "question": "Is this correct?",
            "name": name,
            "birthday": birthday,
        },
    )
    if human_response.get("correct", "").lower().startswith("y"):
        verified_name = name
        verified_birthday = birthday
        response = "Correct"
    else:
        verified_name = human_response.get("name", name)
        verified_birthday = human_response.get("birthday", birthday)
        response = f"Made a correction: {human_response}"

    state_update = {
        "name": verified_name,
        "birthday": verified_birthday,
        "messages": [ToolMessage(response, tool_call_id=tool_call_id)],
    }
    return Command(update=state_update)


tool = TavilySearch(max_results=2)
tools = [tool, human_assistance]
llm_with_tools = llm.bind_tools(tools)

def chatbot(state: State):
    message = llm_with_tools.invoke(state["messages"])
    assert(len(message.tool_calls) <= 1)
    return {"messages": [message]}

graph_builder = StateGraph(State)
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

在完成LangGraph基础教程之前，还有一个概念需要回顾：将`checkpointing`和`state updates`连接到[时间旅行](./6-time-travel.md)。