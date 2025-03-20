# 运行时重建图

你可能需要为新运行使用不同的配置重建你的图。例如，你可能需要根据配置使用不同的图状态或图结构。本指南展示了如何做到这一点。

!!! note "注意"
    在大多数情况下，基于配置自定义行为应该由单个图处理，其中每个节点可以读取配置并根据其更改行为

## 先决条件

确保首先查看[此操作指南](./setup.md)，了解如何为部署设置你的应用。

## 定义图

假设你有一个应用，其中包含一个简单的图，该图调用LLM并将响应返回给用户。应用文件目录如下所示：

```
my-app/
|-- requirements.txt
|-- .env
|-- openai_agent.py     # 你的图代码
```

其中图在`openai_agent.py`中定义。

### 不重建

在标准的LangGraph API配置中，服务器使用在`openai_agent.py`顶层定义的编译图实例，如下所示：

```python
from langchain_openai import ChatOpenAI
from langgraph.graph import END, START, MessageGraph

model = ChatOpenAI(temperature=0)

graph_workflow = MessageGraph()

graph_workflow.add_node("agent", model)
graph_workflow.add_edge("agent", END)
graph_workflow.add_edge(START, "agent")

agent = graph_workflow.compile()
```

为了让服务器知道你的图，你需要在LangGraph API配置（`langgraph.json`）中指定包含`CompiledStateGraph`实例的变量的路径，例如：

```
{
    "dependencies": ["."],
    "graphs": {
        "openai_agent": "./openai_agent.py:agent",
    },
    "env": "./.env"
}
```

### 重建

为了使你的图在每次新运行时使用自定义配置重建，你需要重写`openai_agent.py`，改为提供一个_函数_，该函数接受配置并返回图（或编译图）实例。假设我们想为用户ID '1'返回现有的图，并为其他用户返回一个工具调用代理。我们可以如下修改`openai_agent.py`：

```python
from typing import Annotated
from typing_extensions import TypedDict
from langchain_openai import ChatOpenAI
from langgraph.graph import END, START, MessageGraph
from langgraph.graph.state import StateGraph
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode
from langchain_core.tools import tool
from langchain_core.messages import BaseMessage
from langchain_core.runnables import RunnableConfig


class State(TypedDict):
    messages: Annotated[list[BaseMessage], add_messages]


model = ChatOpenAI(temperature=0)

def make_default_graph():
    """创建一个简单的LLM代理"""
    graph_workflow = StateGraph(State)
    def call_model(state):
        return {"messages": [model.invoke(state["messages"])]}

    graph_workflow.add_node("agent", call_model)
    graph_workflow.add_edge("agent", END)
    graph_workflow.add_edge(START, "agent")

    agent = graph_workflow.compile()
    return agent


def make_alternative_graph():
    """创建一个工具调用代理"""

    @tool
    def add(a: float, b: float):
        """将两个数字相加。"""
        return a + b

    tool_node = ToolNode([add])
    model_with_tools = model.bind_tools([add])
    def call_model(state):
        return {"messages": [model_with_tools.invoke(state["messages"])]}

    def should_continue(state: State):
        if state["messages"][-1].tool_calls:
            return "tools"
        else:
            return END

    graph_workflow = StateGraph(State)

    graph_workflow.add_node("agent", call_model)
    graph_workflow.add_node("tools", tool_node)
    graph_workflow.add_edge("tools", "agent")
    graph_workflow.add_edge(START, "agent")
    graph_workflow.add_conditional_edges("agent", should_continue)

    agent = graph_workflow.compile()
    return agent


# 这是图创建函数，将根据提供的配置决定构建哪个图
def make_graph(config: RunnableConfig):
    user_id = config.get("configurable", {}).get("user_id")
    # 根据用户ID路由到不同的图状态/结构
    if user_id == "1":
        return make_default_graph()
    else:
        return make_alternative_graph()
```

最后，你需要在`langgraph.json`中指定图创建函数（`make_graph`）的路径：

```
{
    "dependencies": ["."],
    "graphs": {
        "openai_agent": "./openai_agent.py:make_graph",
    },
    "env": "./.env"
}
```

有关LangGraph API配置文件的更多信息，请参见[此处](../reference/cli.md#configuration-file)。