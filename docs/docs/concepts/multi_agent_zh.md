# 多智能体系统

[智能体](./agentic_concepts.md#agent-architectures)是**使用LLM来决定应用程序控制流的系统**。随着这些系统的开发，它们可能会变得越来越复杂，使其更难以管理和扩展。例如，你可能会遇到以下问题：

- 智能体拥有太多工具，并且在决定下一步调用哪个工具时做出错误决策
- 上下文变得过于复杂，单个智能体难以跟踪
- 系统中需要多个专业领域（例如规划者、研究者、数学专家等）

为了解决这些问题，你可以考虑将应用程序分解为多个较小的独立智能体，并将它们组合成一个**多智能体系统**。这些独立智能体可以简单到一个提示和一个LLM调用，也可以复杂到一个[ReAct](./agentic_concepts.md#react-implementation)智能体（甚至更复杂！）。

使用多智能体系统的主要好处是：

- **模块化**：独立的智能体使得开发、测试和维护智能体系统更加容易。
- **专业化**：你可以创建专注于特定领域的专家智能体，这有助于提高整体系统性能。
- **控制**：你可以明确控制智能体之间的通信方式（而不是依赖于函数调用）。

## 多智能体架构

![](./img/multi_agent/architectures.png)

在多智能体系统中，有几种方式可以连接智能体：

- **网络**：每个智能体都可以与[其他所有智能体](https://langchain-ai.github.io/langgraph/tutorials/multi_agent/multi-agent-collaboration/)通信。任何智能体都可以决定下一步调用哪个智能体。
- **监督者**：每个智能体与一个[监督者](https://langchain-ai.github.io/langgraph/tutorials/multi_agent/agent_supervisor/)智能体通信。监督者智能体决定下一步调用哪个智能体。
- **监督者（工具调用）**：这是监督者架构的一个特例。个体智能体可以表示为工具。在这种情况下，监督者智能体使用工具调用的LLM来决定调用哪个智能体工具，以及传递给这些智能体的参数。
- **分层**：你可以定义一个[监督者的监督者](https://langchain-ai.github.io/langgraph/tutorials/multi_agent/hierarchical_agent_teams/)的多智能体系统。这是监督者架构的泛化，允许更复杂的控制流。
- **自定义多智能体工作流**：每个智能体仅与一部分智能体通信。流程的某些部分是确定性的，只有部分智能体可以决定下一步调用哪个智能体。

### 交接

在多智能体架构中，智能体可以表示为图节点。每个智能体节点执行其步骤，并决定是完成执行还是路由到另一个智能体，包括可能路由到自身（例如，循环运行）。多智能体交互中的一个常见模式是**交接**，即一个智能体将控制权交给另一个智能体。交接允许你指定：

- **目的地**：要导航到的目标智能体（例如，要去的节点名称）
- **负载**：传递给该智能体的[信息](#communication-between-agents)（例如，状态更新）

要在LangGraph中实现交接，智能体节点可以返回[`Command`](./low_level.md#command)对象，该对象允许你结合控制流和状态更新：

```python
def agent(state) -> Command[Literal["agent", "another_agent"]]:
    # 路由/停止的条件可以是任何东西，例如LLM工具调用/结构化输出等
    goto = get_next_agent(...)  # 'agent' / 'another_agent'
    return Command(
        # 指定下一步调用哪个智能体
        goto=goto,
        # 更新图状态
        update={"my_state_key": "my_state_value"}
    )
```

在更复杂的场景中，每个智能体节点本身是一个图（即[子图](./low_level.md#subgraphs)），其中一个智能体子图中的节点可能希望导航到另一个智能体。例如，如果你有两个智能体`alice`和`bob`（父图中的子图节点），并且`alice`需要导航到`bob`，你可以在`Command`对象中设置`graph=Command.PARENT`：

```python
def some_node_inside_alice(state)
    return Command(
        goto="bob",
        update={"my_state_key": "my_state_value"},
        # 指定要导航到的图（默认为当前图）
        graph=Command.PARENT,
    )
```

!!! 注意
    如果你需要支持使用`Command(graph=Command.PARENT)`通信的子图的可视化，你需要将它们包装在一个带有`Command`注释的节点函数中，例如，而不是这样：

    ```python
    builder.add_node(alice)
    ```

    你需要这样做：

    ```python
    def call_alice(state) -> Command[Literal["bob"]]:
        return alice.invoke(state)

    builder.add_node("alice", call_alice)
    ```

#### 作为工具的交接

最常见的智能体类型之一是ReAct风格的工具调用智能体。对于这些类型的智能体，一个常见的模式是将交接包装在工具调用中，例如：

```python
def transfer_to_bob(state):
    """转移到bob。"""
    return Command(
        goto="bob",
        update={"my_state_key": "my_state_value"},
        graph=Command.PARENT,
    )
```

这是从工具更新图状态的一个特例，除了状态更新外，还包括控制流。

!!! 重要

    如果你想使用返回`Command`的工具，你可以使用预构建的[`create_react_agent`][langgraph.prebuilt.chat_agent_executor.create_react_agent] / [`ToolNode`][langgraph.prebuilt.tool_node.ToolNode]组件，或者实现你自己的工具执行节点，该节点收集工具返回的`Command`对象并返回它们的列表，例如：
    
    ```python
    def call_tools(state):
        ...
        commands = [tools_by_name[tool_call["name"]].invoke(tool_call) for tool_call in tool_calls]
        return commands
    ```

现在让我们更详细地看看不同的多智能体架构。

### 网络

在这种架构中，智能体被定义为图节点。每个智能体都可以与其他所有智能体通信（多对多连接），并且可以决定下一步调用哪个智能体。这种架构适用于没有明确智能体层次结构或特定调用顺序的问题。

```python
from typing import Literal
from langchain_openai import ChatOpenAI
from langgraph.types import Command
from langgraph.graph import StateGraph, MessagesState, START, END

model = ChatOpenAI()

def agent_1(state: MessagesState) -> Command[Literal["agent_2", "agent_3", END]]:
    # 你可以将状态的相关部分传递给LLM（例如，state["messages"]）
    # 以决定下一步调用哪个智能体。一个常见的模式是调用模型
    # 并返回一个结构化输出（例如，强制它返回一个带有"next_agent"字段的输出）
    response = model.invoke(...)
    # 根据LLM的决策路由到其中一个智能体或退出
    # 如果LLM返回"__end__"，图将完成执行
    return Command(
        goto=response["next_agent"],
        update={"messages": [response["content"]]},
    )

def agent_2(state: MessagesState) -> Command[Literal["agent_1", "agent_3", END]]:
    response = model.invoke(...)
    return Command(
        goto=response["next_agent"],
        update={"messages": [response["content"]]},
    )

def agent_3(state: MessagesState) -> Command[Literal["agent_1", "agent_2", END]]:
    ...
    return Command(
        goto=response["next_agent"],
        update={"messages": [response["content"]]},
    )

builder = StateGraph(MessagesState)
builder.add_node(agent_1)
builder.add_node(agent_2)
builder.add_node(agent_3)

builder.add_edge(START, "agent_1")
network = builder.compile()
```

### 监督者

在这种架构中，我们将智能体定义为节点，并添加一个监督者节点（LLM），该节点决定下一步应该调用哪个智能体节点。我们使用[`Command`](./low_level.md#command)根据监督者的决策将执行路由到适当的智能体节点。这种架构也适合并行运行多个智能体或使用[map-reduce](../how-tos/map-reduce.ipynb)模式。

```python
from typing import Literal
from langchain_openai import ChatOpenAI
from langgraph.types import Command
from langgraph.graph import StateGraph, MessagesState, START, END

model = ChatOpenAI()

def supervisor(state: MessagesState) -> Command[Literal["agent_1", "agent_2", END]]:
    # 你可以将状态的相关部分传递给LLM（例如，state["messages"]）
    # 以决定下一步调用哪个智能体。一个常见的模式是调用模型
    # 并返回一个结构化输出（例如，强制它返回一个带有"next_agent"字段的输出）
    response = model.invoke(...)
    # 根据监督者的决策路由到其中一个智能体或退出
    # 如果监督者返回"__end__"，图将完成执行
    return Command(goto=response["next_agent"])

def agent_1(state: MessagesState) -> Command[Literal["supervisor"]]:
    # 你可以将状态的相关部分传递给LLM（例如，state["messages"]）
    # 并添加任何额外的逻辑（不同的模型、自定义提示、结构化输出等）
    response = model.invoke(...)
    return Command(
        goto="supervisor",
        update={"messages": [response]},
    )

def agent_2(state: MessagesState) -> Command[Literal["supervisor"]]:
    response = model.invoke(...)
    return Command(
        goto="supervisor",
        update={"messages": [response]},
    )

builder = StateGraph(MessagesState)
builder.add_node(supervisor)
builder.add_node(agent_1)
builder.add_node(agent_2)

builder.add_edge(START, "supervisor")

supervisor = builder.compile()
```

查看这个[教程](https://langchain-ai.github.io/langgraph/tutorials/multi_agent/agent_supervisor/)以获取监督者多智能体架构的示例。

### 监督者（工具调用）

在[监督者](#supervisor)架构的这种变体中，我们将个体智能体定义为**工具**，并在监督者节点中使用工具调用的LLM。这可以实现为一个[ReAct](./agentic_concepts.md#react-implementation)风格的智能体，具有两个节点——一个LLM节点（监督者）和一个执行工具（在这种情况下是智能体）的工具调用节点。

```python
from typing import Annotated
from langchain_openai import ChatOpenAI
from langgraph.prebuilt import InjectedState, create_react_agent

model = ChatOpenAI()

# 这是将作为工具调用的智能体函数
# 注意，你可以通过InjectedState注解将状态传递给工具
def agent_1(state: Annotated[dict, InjectedState]):
    # 你可以将状态的相关部分传递给LLM（例如，state["messages"]）
    # 并添加任何额外的逻辑（不同的模型、自定义提示、结构化输出等）
    response = model.invoke(...)
    # 将LLM响应作为字符串返回（预期的工具响应格式）
    # 这将由预构建的create_react_agent（监督者）自动转换为ToolMessage
    return response.content

def agent_2(state: Annotated[dict, InjectedState]):
    response = model.invoke(...)
    return response.content

tools = [agent_1, agent_2]
# 构建一个带有工具调用的监督者的最简单方法是使用预构建的ReAct智能体图
# 它由一个工具调用的LLM节点（即监督者）和一个工具执行节点组成
supervisor = create_react_agent(model, tools)
```

### 分层

随着你向系统中添加更多智能体，监督者可能会变得难以管理所有智能体。监督者可能会开始做出关于下一步调用哪个智能体的错误决策，上下文可能会变得过于复杂，单个监督者难以跟踪。换句话说，你最终会遇到最初促使你采用多智能体架构的相同问题。

为了解决这个问题，你可以**分层**设计你的系统。例如，你可以创建由个体监督者管理的独立、专业化的智能体团队，并由一个顶级监督者来管理这些团队。

```python
from typing import Literal
from langchain_openai import ChatOpenAI
from langgraph.graph import StateGraph, MessagesState, START, END
from langgraph.types import Command
model = ChatOpenAI()

# 定义团队1（与单个监督者示例相同）

def team_1_supervisor(state: MessagesState) -> Command[Literal["team_1_agent_1", "team_1_agent_2", END]]:
    response = model.invoke(...)
    return Command(goto=response["next_agent"])

def team_1_agent_1(state: MessagesState) -> Command[Literal["team_1_supervisor"]]:
    response = model.invoke(...)
    return Command(goto="team_1_supervisor", update={"messages": [response]})

def team_1_agent_2(state: MessagesState) -> Command[Literal["team_1_supervisor"]]:
    response = model.invoke(...)
    return Command(goto="team_1_supervisor", update={"messages": [response]})

team_1_builder = StateGraph(Team1State)
team_1_builder.add_node(team_1_supervisor)
team_1_builder.add_node(team_1_agent_1)
team_1_builder.add_node(team_1_agent_2)
team_1_builder.add_edge(START, "team_1_supervisor")
team_1_graph = team_1_builder.compile()

# 定义团队2（与单个监督者示例相同）
class Team2State(MessagesState):
    next: Literal["team_2_agent_1", "team_2_agent_2", "__end__"]

def team_2_supervisor(state: Team2State):
    ...

def team_2_agent_1(state: Team2State):
    ...

def team_2_agent_2(state: Team2State):
    ...

team_2_builder = StateGraph(Team2State)
...
team_2_graph = team_2_builder.compile()


# 定义顶级监督者

builder = StateGraph(MessagesState)
def top_level_supervisor(state: MessagesState) -> Command[Literal["team_1_graph", "team_2_graph", END]]:
    # 你可以将状态的相关部分传递给LLM（例如，state["messages"]）
    # 以决定下一步调用哪个团队。一个常见的模式是调用模型
    # 并返回一个结构化输出（例如，强制它返回一个带有"next_team"字段的输出）
    response = model.invoke(...)
    # 根据监督者的决策路由到其中一个团队或退出
    # 如果监督者返回"__end__"，图将完成执行
    return Command(goto=response["next_team"])

builder = StateGraph(MessagesState)
builder.add_node(top_level_supervisor)
builder.add_node("team_1_graph", team_1_graph)
builder.add_node("team_2_graph", team_2_graph)
builder.add_edge(START, "top_level_supervisor")
builder.add_edge("team_1_graph", "top_level_supervisor")
builder.add_edge("team_2_graph", "top_level_supervisor")
graph = builder.compile()
```

### 自定义多智能体工作流

在这种架构中，我们将个体智能体添加为图节点，并提前定义智能体的调用顺序，形成一个自定义工作流。在LangGraph中，工作流可以通过两种方式定义：

- **显式控制流（普通边）**：LangGraph允许你通过[普通图边](./low_level.md#normal-edges)显式定义应用程序的控制流（即智能体如何通信的顺序）。这是上述架构中最确定性的变体——我们总是提前知道下一步将调用哪个智能体。

- **动态控制流（Command）**：在LangGraph中，你可以允许LLM决定应用程序控制流的一部分。这可以通过使用[`Command`](./low_level.md#command)来实现。一个特例是[监督者工具调用](#supervisor-tool-calling)架构。在这种情况下，为监督者智能体提供支持的工具调用LLM将决定工具（智能体）的调用顺序。

```python
from langchain_openai import ChatOpenAI
from langgraph.graph import StateGraph, MessagesState, START

model = ChatOpenAI()

def agent_1(state: MessagesState):
    response = model.invoke(...)
    return {"messages": [response]}

def agent_2(state: MessagesState):
    response = model.invoke(...)
    return {"messages": [response]}

builder = StateGraph(MessagesState)
builder.add_node(agent_1)
builder.add_node(agent_2)
# 显式定义流程
builder.add_edge(START, "agent_1")
builder.add_edge("agent_1", "agent_2")
```

## 智能体之间的通信

构建多智能体系统时，最重要的事情是弄清楚智能体如何通信。有几个不同的考虑因素：

- 智能体是通过[图状态还是工具调用](#graph-state-vs-tool-calls)进行通信？
- 如果两个智能体有[不同的状态模式](#different-state-schemas)怎么办？
- 如何通过[共享消息列表](#shared-message-list)进行通信？

### 图状态 vs 工具调用

智能体之间传递的“负载”是什么？在上面讨论的大多数架构中，智能体通过[图状态](./low_level.md#state)进行通信。在[工具调用的监督者](#supervisor-tool-calling)的情况下，负载是工具调用的参数。

![](./img/multi_agent/request.png)

#### 图状态

要通过图状态进行通信，个体智能体需要定义为[图节点](./low_level.md#nodes)。这些可以作为函数或整个[子图](./low_level.md#subgraphs)添加。在图执行的每一步，智能体节点接收图的当前状态，执行智能体代码，然后将更新后的状态传递给下一个节点。

通常，智能体节点共享一个单一的[状态模式](./low_level.md#schema)。然而，你可能希望设计具有[不同状态模式](#different-state-schemas)的智能体节点。

### 不同的状态模式

一个智能体可能需要与其他智能体不同的状态模式。例如，搜索智能体可能只需要跟踪查询和检索到的文档。在LangGraph中有两种方法可以实现这一点：

- 定义具有单独状态模式的[子图](./low_level.md#subgraphs)智能体。如果子图和父图之间没有共享的状态键（通道），重要的是[添加输入/输出转换](https://langchain-ai.github.io/langgraph/how-tos/subgraph-transform-state/)，以便父图知道如何与子图通信。
- 定义