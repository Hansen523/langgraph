# 构建基础聊天机器人

在本教程中，您将构建一个基础聊天机器人。这个机器人将是后续系列教程的基础，您将逐步为其添加更复杂的功能，并在此过程中学习LangGraph的关键概念。让我们开始吧！🌟

## 先决条件

开始本教程前，请确保您拥有支持工具调用功能的LLM访问权限，例如[OpenAI](https://platform.openai.com/api-keys)、[Anthropic](https://console.anthropic.com/settings/keys)或[Google Gemini](https://ai.google.dev/gemini-api/docs/api-key)。

## 1. 安装包

安装所需包：

```bash
pip install -U langgraph langsmith
```

!!! 提示

    注册LangSmith可以快速发现问题并提升LangGraph项目的性能。LangSmith让您使用追踪数据来调试、测试和监控用LangGraph构建的LLM应用。更多入门信息，请参阅[LangSmith文档](https://docs.smith.langchain.com)。

## 2. 创建`StateGraph`

现在您可以用LangGraph创建基础聊天机器人。这个机器人将直接响应用户消息。

首先创建`StateGraph`。`StateGraph`对象将我们的聊天机器人定义为"状态机"。我们将添加`nodes`来表示LLM和机器人可以调用的函数，以及`edges`来指定机器人如何在各个功能间转换。

```python
from typing import Annotated

from typing_extensions import TypedDict

from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages


class State(TypedDict):
    # 消息类型为"list"。注释中的`add_messages`函数定义了如何更新此状态键
    # (此处是将消息追加到列表而非覆盖)
    messages: Annotated[list, add_messages]


graph_builder = StateGraph(State)
```

我们的图现在可以处理两个关键任务：

1. 每个`node`可以接收当前`State`作为输入并输出状态更新。
2. 使用`Annotated`语法配合预构建的[`add_messages`](https://langchain-ai.github.io/langgraph/reference/graphs/?h=add+messages#add_messages)函数，对`messages`的更新会追加到现有列表而非覆盖。

------

!!! 提示 "概念"

    定义图时，第一步是定义其`State`。`State`包含图的模式和[reducer函数](https://langchain-ai.github.io/langgraph/concepts/low_level/#reducers)来处理状态更新。在我们的例子中，`State`是有一个键的`TypedDict`：`messages`。使用[`add_messages`](https://langchain-ai.github.io/langgraph/reference/graphs/#langgraph.graph.message.add_messages) reducer函数将新消息追加到列表而非覆盖。没有reducer注释的键会覆盖之前的值。了解更多关于状态、reducer和相关概念的信息，请参阅[LangGraph参考文档](https://langchain-ai.github.io/langgraph/reference/graphs/#langgraph.graph.message.add_messages)。

## 3. 添加节点

接下来添加"`chatbot`"节点。**节点**代表工作单元，通常是常规Python函数。

首先选择聊天模型：

{!snippets/chat_model_tabs.md!}

<!---
```python
from langchain.chat_models import init_chat_model

llm = init_chat_model("anthropic:claude-3-5-sonnet-latest")
```
-->

现在将聊天模型整合到一个简单节点中：

```python

def chatbot(state: State):
    return {"messages": [llm.invoke(state["messages"])]}


# 第一个参数是唯一节点名
# 第二个参数是节点被调用时将执行的函数或对象
graph_builder.add_node("chatbot", chatbot)
```

**注意**`chatbot`节点函数如何接受当前`State`作为输入，并返回包含键"messages"下更新后的`messages`列表的字典。这是所有LangGraph节点函数的基本模式。

我们`State`中的`add_messages`函数会将LLM的响应消息追加到状态中已有的任何消息上。

## 4. 添加`entry`入口点

添加`entry`入口点告诉图每次运行时**从哪里开始工作**：

```python
graph_builder.add_edge(START, "chatbot")
```

## 5. 添加`exit`出口点

添加`exit`出口点表示**图应在何处完成执行**。这对于更复杂的流程很有帮助，即使在这样简单的图中，添加结束节点也能提高清晰度。

```python
graph_builder.add_edge("chatbot", END)
```
这告诉图在运行chatbot节点后终止。

## 6. 编译图

在运行图之前，我们需要编译它。我们可以调用`compile()`方法在图表构建器上，创建一个可以在状态上调用的`CompiledGraph`。

```python
graph = graph_builder.compile()
```

## 7. 可视化图(可选)

您可以使用`get_graph`方法和"draw"方法之一，如`draw_ascii`或`draw_png`，来可视化图。各种draw方法需要额外的依赖。

```python
from IPython.display import Image, display

try:
    display(Image(graph.get_graph().draw_mermaid_png()))
except Exception:
    # 这需要一些额外依赖，是可选的
    pass
```

![基础聊天机器人图](basic-chatbot.png)


## 8. 运行聊天机器人

现在运行聊天机器人！

!!! 提示

    您可以随时输入`quit`、`exit`或`q`退出聊天循环。

```python
def stream_graph_updates(user_input: str):
    for event in graph.stream({"messages": [{"role": "user", "content": user_input}]}):
        for value in event.values():
            print("助理:", value["messages"][-1].content)


while True:
    try:
        user_input = input("用户: ")
        if user_input.lower() in ["quit", "exit", "q"]:
            print("再见!")
            break
        stream_graph_updates(user_input)
    except:
        # 如果input()不可用时的后备方案
        user_input = "你对LangGraph了解多少？"
        print("用户: " + user_input)
        stream_graph_updates(user_input)
        break
```

```
助理: LangGraph是一个旨在帮助使用语言模型构建有状态多代理应用的库。它提供了创建工作流和状态机的工具，用于协调多个AI代理或语言模型交互。LangGraph建立在LangChain之上，利用其组件同时添加基于图的协调能力。对于开发超出简单查询-响应交互的更复杂、有状态的AI应用特别有用。
再见!
```

**恭喜！**您已使用LangGraph构建了第一个聊天机器人。该机器人可以通过接收用户输入并使用LLM生成响应来进行基本对话。您可以在[LangSmith Trace](https://smith.langchain.com/public/7527e308-9502-4894-b347-f34385740d5a/r)中检查上述调用。

以下是本教程的完整代码：

```python
from typing import Annotated

from langchain.chat_models import init_chat_model
from typing_extensions import TypedDict

from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages


class State(TypedDict):
    messages: Annotated[list, add_messages]


graph_builder = StateGraph(State)


llm = init_chat_model("anthropic:claude-3-5-sonnet-latest")


def chatbot(state: State):
    return {"messages": [llm.invoke(state["messages"])]}


# 第一个参数是唯一节点名
# 第二个参数是节点被调用时将执行的函数或对象
graph_builder.add_node("chatbot", chatbot)
graph_builder.add_edge(START, "chatbot")
graph_builder.add_edge("chatbot", END)
graph = graph_builder.compile()
```

## 下一步

您可能注意到机器人的知识仅限于其训练数据中的内容。在下一部分中，我们将[添加网络搜索工具](./2-add-tools.md)以扩展机器人的知识并增强其能力。