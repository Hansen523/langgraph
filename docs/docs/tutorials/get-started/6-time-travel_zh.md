# 时间旅行

在典型的聊天机器人工作流程中，用户会与机器人进行一次或多次交互以完成任务。[记忆功能](./3-add-memory.md)和[人在回路](./4-human-in-the-loop.md)机制允许在图状态中设置检查点，从而控制未来的响应。

如果用户想要从之前的某个响应点开始探索不同结果会怎样？或者如果想让用户能够"倒回"聊天机器人的工作进程来修正错误或尝试不同策略（这在自主软件工程师等应用中很常见），又该如何实现？

您可以使用LangGraph内置的**时间旅行**功能来创建这类交互体验。

!!! 注意

    本教程基于[自定义状态](./5-customize-state.md)构建。

## 1. 回滚图状态

通过图的`get_state_history`方法获取检查点来回滚图状态。然后可以从这个历史时间点恢复执行。

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
from langchain_core.messages import BaseMessage
from typing_extensions import TypedDict

from langgraph.checkpoint.memory import MemorySaver
from langgraph.graph import StateGraph, START, END
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
graph_builder.add_edge(START, "chatbot")

memory = MemorySaver()
graph = graph_builder.compile(checkpointer=memory)
```

## 2. 添加执行步骤

向图中添加执行步骤。每个步骤的状态都会被记录在状态历史中：

``` python
config = {"configurable": {"thread_id": "1"}}
events = graph.stream(
    {
        "messages": [
            {
                "role": "user",
                "content": (
                    "I'm learning LangGraph. "
                    "Could you do some research on it for me?"
                ),
            },
        ],
    },
    config,
    stream_mode="values",
)
for event in events:
    if "messages" in event:
        event["messages"][-1].pretty_print()
```

```
================================ Human Message =================================

I'm learning LangGraph. Could you do some research on it for me?
================================== Ai Message ==================================

[{'text': "Certainly! I'd be happy to research LangGraph for you. To get the most up-to-date and accurate information, I'll use the Tavily search engine to look this up. Let me do that for you now.", 'type': 'text'}, {'id': 'toolu_01BscbfJJB9EWJFqGrN6E54e', 'input': {'query': 'LangGraph latest information and features'}, 'name': 'tavily_search_results_json', 'type': 'tool_use'}]
Tool Calls:
  tavily_search_results_json (toolu_01BscbfJJB9EWJFqGrN6E54e)
 Call ID: toolu_01BscbfJJB9EWJFqGrN6E54e
  Args:
    query: LangGraph latest information and features
================================= Tool Message =================================
Name: tavily_search_results_json

[{"url": "https://blockchain.news/news/langchain-new-features-upcoming-events-update", "content": "LangChain, a leading platform in the AI development space, has released its latest updates, showcasing new use cases and enhancements across its ecosystem. According to the LangChain Blog, the updates cover advancements in LangGraph Platform, LangSmith's self-improving evaluators, and revamped documentation for LangGraph."}, {"url": "https://blog.langchain.dev/langgraph-platform-announce/", "content": "With these learnings under our belt, we decided to couple some of our latest offerings under LangGraph Platform. LangGraph Platform today includes LangGraph Server, LangGraph Studio, plus the CLI and SDK. ... we added features in LangGraph Server to deliver on a few key value areas. Below, we'll focus on these aspects of LangGraph Platform."}]
================================== Ai Message ==================================

Thank you for your patience. I've found some recent information about LangGraph for you. Let me summarize the key points:

1. LangGraph is part of the LangChain ecosystem, which is a leading platform in AI development.

2. Recent updates and features of LangGraph include:

   a. LangGraph Platform: This seems to be a cloud-based version of LangGraph, though specific details weren't provided in the search results.
...
3. Keep an eye on LangGraph Platform developments, as cloud-based solutions often provide an easier starting point for learners.
4. Consider how LangGraph fits into the broader LangChain ecosystem, especially its interaction with tools like LangSmith.

Is there any specific aspect of LangGraph you'd like to know more about? I'd be happy to do a more focused search on particular features or use cases.
Output is truncated. View as a scrollable element or open in a text editor. Adjust cell output settings...
```

```python
events = graph.stream(
    {
        "messages": [
            {
                "role": "user",
                "content": (
                    "Ya that's helpful. Maybe I'll "
                    "build an autonomous agent with it!"
                ),
            },
        ],
    },
    config,
    stream_mode="values",
)
for event in events:
    if "messages" in event:
        event["messages"][-1].pretty_print()
```

```
================================ Human Message =================================

Ya that's helpful. Maybe I'll build an autonomous agent with it!
================================== Ai Message ==================================

[{'text': "That's an exciting idea! Building an autonomous agent with LangGraph is indeed a great application of this technology. LangGraph is particularly well-suited for creating complex, multi-step AI workflows, which is perfect for autonomous agents. Let me gather some more specific information about using LangGraph for building autonomous agents.", 'type': 'text'}, {'id': 'toolu_01QWNHhUaeeWcGXvA4eHT7Zo', 'input': {'query': 'Building autonomous agents with LangGraph examples and tutorials'}, 'name': 'tavily_search_results_json', 'type': 'tool_use'}]
Tool Calls:
  tavily_search_results_json (toolu_01QWNHhUaeeWcGXvA4eHT7Zo)
 Call ID: toolu_01QWNHhUaeeWcGXvA4eHT7Zo
  Args:
    query: Building autonomous agents with LangGraph examples and tutorials
================================= Tool Message =================================
Name: tavily_search_results_json

[{"url": "https://towardsdatascience.com/building-autonomous-multi-tool-agents-with-gemini-2-0-and-langgraph-ad3d7bd5e79d", "content": "Building Autonomous Multi-Tool Agents with Gemini 2.0 and LangGraph | by Youness Mansar | Jan, 2025 | Towards Data Science Building Autonomous Multi-Tool Agents with Gemini 2.0 and LangGraph A practical tutorial with full code examples for building and running multi-tool agents Towards Data Science LLMs are remarkable — they can memorize vast amounts of information, answer general knowledge questions, write code, generate stories, and even fix your grammar. In this tutorial, we are going to build a simple LLM agent that is equipped with four tools that it can use to answer a user’s question. This Agent will have the following specifications: Follow Published in Towards Data Science --------------------------------- Your home for data science and AI. Follow Follow Follow"}, {"url": "https://github.com/anmolaman20/Tools_and_Agents", "content": "GitHub - anmolaman20/Tools_and_Agents: This repository provides resources for building AI agents using Langchain and Langgraph. This repository provides resources for building AI agents using Langchain and Langgraph. This repository provides resources for building AI agents using Langchain and Langgraph. This repository serves as a comprehensive guide for building AI-powered agents using Langchain and Langgraph. It provides hands-on examples, practical tutorials, and resources for developers and AI enthusiasts to master building intelligent systems and workflows. AI Agent Development: Gain insights into creating intelligent systems that think, reason, and adapt in real time. This repository is ideal for AI practitioners, developers exploring language models, or anyone interested in building intelligent systems. This repository provides resources for building AI agents using Langchain and Langgraph."}]
================================== Ai Message ==================================

Great idea! Building an autonomous agent with LangGraph is definitely an exciting project. Based on the latest information I've found, here are some insights and tips for building autonomous agents with LangGraph:

1. Multi-Tool Agents: LangGraph is particularly well-suited for creating autonomous agents that can use multiple tools. This allows your agent to have a diverse set of capabilities and choose the right tool for each task.

2. Integration with Large Language Models (LLMs): You can combine LangGraph with powerful LLMs like Gemini 2.0 to create more intelligent and capable agents. The LLM can serve as the "brain" of your agent, making decisions and generating responses.

3. Workflow Management: LangGraph excels at managing complex, multi-step AI workflows. This is crucial for autonomous agents that need to break down tasks into smaller steps and execute them in the right order.
...
6. Pay attention to how you structure the agent's decision-making process and workflow.
7. Don't forget to implement proper error handling and safety measures, especially if your agent will be interacting with external systems or making important decisions.

Building an autonomous agent is an iterative process, so be prepared to refine and improve your agent over time. Good luck with your project! If you need any more specific information as you progress, feel free to ask.
Output is truncated. View as a scrollable element or open in a text editor. Adjust cell output settings...
```

## 3. 重放完整状态历史

现在已向聊天机器人添加了执行步骤，您可以`重放`完整状态历史来查看所有发生过的交互。

``` python
to_replay = None
for state in graph.get_state_history(config):
    print("Num Messages: ", len(state.values["messages"]), "Next: ", state.next)
    print("-" * 80)
    if len(state.values["messages"]) == 6:
        # 我们基于状态中聊天消息数量，特定选择一个状态进行回放
        to_replay = state
```

```
Num Messages:  8 Next:  ()
--------------------------------------------------------------------------------
Num Messages:  7 Next:  ('chatbot',)
--------------------------------------------------------------------------------
Num Messages:  6 Next:  ('tools',)
--------------------------------------------------------------------------------
Num Messages:  5 Next:  ('chatbot',)
--------------------------------------------------------------------------------
Num Messages:  4 Next:  ('__start__',)
--------------------------------------------------------------------------------
Num Messages:  4 Next:  ()
--------------------------------------------------------------------------------
Num Messages:  3 Next:  ('chatbot',)
--------------------------------------------------------------------------------
Num Messages:  2 Next:  ('tools',)
--------------------------------------------------------------------------------
Num Messages:  1 Next:  ('chatbot',)
--------------------------------------------------------------------------------
Num Messages:  0 Next:  ('__start__',)
--------------------------------------------------------------------------------
```

图的每个步骤都会保存检查点。这__跨越多次调用__，因此您可以回滚整个线程的历史记录。

## 从检查点恢复

从`to_replay`状态恢复执行，该状态位于第二次图调用中`chatbot`节点之后。从这个点恢复将接着调用**action**节点。

```python
print(to_replay.next)
print(to_replay.config)
```

```
('tools',)
{'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1efd43e3-0c1f-6c4e-8006-891877d65740'}}
```

## 4. 从特定时间点加载状态

检查点`to_replay.config`中包含`checkpoint_id`时间戳。提供这个`checkpoint_id`值会指示LangGraph的检查点器从那个时间点**加载**状态。


``` python
# `to_replay.config`中的`checkpoint_id`对应我们已持久化到检查点器的某个状态
for event in graph.stream(None, to_replay.config, stream_mode="values"):
    if "messages" in event:
        event["messages"][-1].pretty_print()
```

```
================================== Ai Message ==================================

[{'text': "That's an exciting idea! Building an autonomous agent with LangGraph is indeed a great application of this technology. LangGraph is particularly well-suited for creating complex, multi-step AI workflows, which is perfect for autonomous agents. Let me gather some more specific information about using LangGraph for building autonomous agents.", 'type': 'text'}, {'id': 'toolu_01QWNHhUaeeWcGXvA4eHT7Zo', 'input': {'query': 'Building autonomous agents with LangGraph examples and tutorials'}, 'name': 'tavily_search_results_json', 'type': 'tool_use'}]
Tool Calls:
  tavily_search_results_json (toolu_01QWNHhUaeeWcGXvA4eHT7Zo)
 Call ID: toolu_01QWNHhUaeeWcGXvA4eHT7Zo
  Args:
    query: Building autonomous agents with LangGraph examples and tutorials
================================= Tool Message =================================
Name: tavily_search_results_json

[{"url": "https://towardsdatascience.com/building-autonomous-multi-tool-agents-with-gemini-2-0-and-langgraph-ad3d7bd5e79d", "content": "Building Autonomous Multi-Tool Agents with Gemini 2.0 and LangGraph | by Youness Mansar | Jan, 2025 | Towards Data Science Building Autonomous Multi-Tool Agents with Gemini 2.0 and LangGraph A practical tutorial with full code examples for building and running multi-tool agents Towards Data Science LLMs are remarkable — they can memorize vast amounts of information, answer general knowledge questions, write code, generate stories, and even fix your grammar. In this tutorial, we are going to build a simple LLM agent that is equipped with four tools that it can use to answer a user’s question. This Agent will have the following specifications: Follow Published in Towards Data Science --------------------------------- Your home for data science and AI. Follow Follow Follow"}, {"url": "https://github.com/anmolaman20/Tools_and_Agents", "content": "GitHub - anmolaman20/Tools_and_Agents: This repository provides resources for building AI agents using Langchain and Langgraph. This repository provides resources for building AI agents using Langchain and Langgraph. This repository provides resources for building AI agents using Langchain and Langgraph. This repository serves as a comprehensive guide for building AI-powered agents using Langchain and Langgraph. It provides hands-on examples, practical tutorials, and resources for developers and AI enthusiasts to master building intelligent systems and workflows. AI Agent Development: Gain insights into creating intelligent systems that think, reason, and adapt in real time. This repository is ideal for AI practitioners, developers exploring language models, or anyone interested in building intelligent systems. This repository provides resources for building AI agents using Langchain and Langgraph."}]
================================== Ai Message ==================================

Great idea! Building an autonomous agent with LangGraph is indeed an excellent way to apply and deepen your understanding of the technology. Based on the search results, I can provide you with some insights and resources to help you get started:

1. Multi-Tool Agents:
   LangGraph is well-suited for building autonomous agents that can use multiple tools. This allows your agent to have a variety of capabilities and choose the appropriate tool based on the task at hand.

2. Integration with Large Language Models (LLMs):
   There's a tutorial that specifically mentions using Gemini 2.0 (Google's LLM) with LangGraph to build autonomous agents. This suggests that LangGraph can be integrated with various LLMs, giving you flexibility in choosing the language model that best fits your needs.

3. Practical Tutorials:
   There are tutorials available that provide full code examples for building and running multi-tool agents. These can be invaluable as you start your project, giving you a concrete starting point and demonstrating best practices.
...

Remember, building an autonomous agent is an iterative process. Start simple and gradually increase complexity as you become more comfortable with LangGraph and its capabilities.

Would you like more information on any specific aspect of building your autonomous agent with LangGraph?
Output is truncated. View as a scrollable element or open in a text editor. Adjust cell output settings...
```

图从`action`节点恢复了执行。您可以从上方打印的第一个值（来自搜索引擎工具的响应）看出这一点。

**恭喜！**您现在已经掌握了LangGraph中时间旅行检查点遍历的功能。能够回滚状态并探索不同路径，为调试、实验和交互式应用开辟了无限可能。

## 深入学习

通过以下内容继续您的LangGraph探索之旅：

- **[LangGraph Server快速入门](../../tutorials/langgraph-platform/local-server.md)**: 本地启动LangGraph服务器，并使用REST API和LangGraph Studio Web UI与之交互。
- **[LangGraph Platform快速入门](../../cloud/quick_start.md)**: 使用LangGraph Platform部署您的LangGraph应用。
- **[LangGraph Platform概念](../../concepts/langgraph_platform.md)**: 了解LangGraph Platform的基础概念。