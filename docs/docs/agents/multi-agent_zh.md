---
search:
  boost: 2
tags:
  - agent
hide:
  - tags
---

# 多智能体系统

当单个智能体需要处理多领域任务或管理大量工具时往往会力不从心。为此，可以将智能体拆分为多个独立的小型智能体，组合成一个[多智能体系统](../concepts/multi_agent.md)。

在多智能体系统中，智能体间需要相互通信。它们通过[移交控制权](#handoffs)机制实现交互——该机制定义了控制权转移的目标智能体及传递的数据内容。

目前最流行的两种多智能体架构是：

- [监管者模式](#supervisor) —— 由中央监管智能体协调各个子智能体。监管者控制所有通信流和任务分配，根据当前上下文和任务需求决定调用哪个智能体。
- [群体模式](#swarm) —— 智能体基于各自专长动态移交控制权。系统会记录最后活跃的智能体，确保后续交互能延续与该智能体的对话。

## 监管者模式

![监管者模式](./assets/supervisor.png)

使用[`langgraph-supervisor`](https://github.com/langchain-ai/langgraph-supervisor-py)库创建监管者多智能体系统：

```bash
pip install langgraph-supervisor
```

```python
from langchain_openai import ChatOpenAI
from langgraph.prebuilt import create_react_agent
# highlight-next-line
from langgraph_supervisor import create_supervisor

def book_hotel(hotel_name: str):
    """Book a hotel"""
    return f"Successfully booked a stay at {hotel_name}."

def book_flight(from_airport: str, to_airport: str):
    """Book a flight"""
    return f"Successfully booked a flight from {from_airport} to {to_airport}."

flight_assistant = create_react_agent(
    model="openai:gpt-4o",
    tools=[book_flight],
    prompt="You are a flight booking assistant",
    # highlight-next-line
    name="flight_assistant"
)

hotel_assistant = create_react_agent(
    model="openai:gpt-4o",
    tools=[book_hotel],
    prompt="You are a hotel booking assistant",
    # highlight-next-line
    name="hotel_assistant"
)

# highlight-next-line
supervisor = create_supervisor(
    agents=[flight_assistant, hotel_assistant],
    model=ChatOpenAI(model="gpt-4o"),
    prompt=(
        "You manage a hotel booking assistant and a"
        "flight booking assistant. Assign work to them."
    )
).compile()

for chunk in supervisor.stream(
    {
        "messages": [
            {
                "role": "user",
                "content": "book a flight from BOS to JFK and a stay at McKittrick Hotel"
            }
        ]
    }
):
    print(chunk)
    print("\n")
```

## 群体模式

![群体模式](./assets/swarm.png)

使用[`langgraph-swarm`](https://github.com/langchain-ai/langgraph-swarm-py)库创建群体多智能体系统：

```bash
pip install langgraph-swarm
```

```python
from langgraph.prebuilt import create_react_agent
# highlight-next-line
from langgraph_swarm import create_swarm, create_handoff_tool

transfer_to_hotel_assistant = create_handoff_tool(
    agent_name="hotel_assistant",
    description="Transfer user to the hotel-booking assistant.",
)
transfer_to_flight_assistant = create_handoff_tool(
    agent_name="flight_assistant",
    description="Transfer user to the flight-booking assistant.",
)

flight_assistant = create_react_agent(
    model="anthropic:claude-3-5-sonnet-latest",
    # highlight-next-line
    tools=[book_flight, transfer_to_hotel_assistant],
    prompt="You are a flight booking assistant",
    # highlight-next-line
    name="flight_assistant"
)
hotel_assistant = create_react_agent(
    model="anthropic:claude-3-5-sonnet-latest",
    # highlight-next-line
    tools=[book_hotel, transfer_to_flight_assistant],
    prompt="You are a hotel booking assistant",
    # highlight-next-line
    name="hotel_assistant"
)

# highlight-next-line
swarm = create_swarm(
    agents=[flight_assistant, hotel_assistant],
    default_active_agent="flight_assistant"
).compile()

for chunk in swarm.stream(
    {
        "messages": [
            {
                "role": "user",
                "content": "book a flight from BOS to JFK and a stay at McKittrick Hotel"
            }
        ]
    }
):
    print(chunk)
    print("\n")
```

## 控制权移交

多智能体交互中的常见模式是**控制权移交**，即一个智能体将控制权*移交*给另一个智能体。移交机制允许指定：

- **目标**：要转移到的目标智能体
- **数据负载**：传递给该智能体的信息

`langgraph-supervisor`（监管者移交控制权给单个智能体）和`langgraph-swarm`（单个智能体可移交控制权给其他智能体）都使用了这种机制。

要在`create_react_agent`中实现控制权移交，需要：

1. 创建能转移控制权给其他智能体的特殊工具

    ```python
    def transfer_to_bob():
        """Transfer to bob."""
        return Command(
            # 目标智能体（节点）名称
            # highlight-next-line
            goto="bob",
            # 发送给该智能体的数据
            # highlight-next-line
            update={"messages": [...]},
            # 指示LangGraph需要导航至
            # 父图中的智能体节点
            # highlight-next-line
            graph=Command.PARENT,
        )
    ```

1. 创建具有移交工具的独立智能体：

    ```python
    flight_assistant = create_react_agent(
        ..., tools=[book_flight, transfer_to_hotel_assistant]
    )
    hotel_assistant = create_react_agent(
        ..., tools=[book_hotel, transfer_to_flight_assistant]
    )
    ```

1. 定义包含各智能体节点的父图：

    ```python
    from langgraph.graph import StateGraph, MessagesState
    multi_agent_graph = (
        StateGraph(MessagesState)
        .add_node(flight_assistant)
        .add_node(hotel_assistant)
        ...
    )
    ```

综合示例如下，实现包含航班预订助手和酒店预订助手的简单多智能体系统：

```python
from typing import Annotated
from langchain_core.tools import tool, InjectedToolCallId
from langgraph.prebuilt import create_react_agent, InjectedState
from langgraph.graph import StateGraph, START, MessagesState
from langgraph.types import Command

def create_handoff_tool(*, agent_name: str, description: str | None = None):
    name = f"transfer_to_{agent_name}"
    description = description or f"Transfer to {agent_name}"

    @tool(name, description=description)
    def handoff_tool(
        # highlight-next-line
        state: Annotated[MessagesState, InjectedState], # (1)!
        # highlight-next-line
        tool_call_id: Annotated[str, InjectedToolCallId],
    ) -> Command:
        tool_message = {
            "role": "tool",
            "content": f"Successfully transferred to {agent_name}",
            "name": name,
            "tool_call_id": tool_call_id,
        }
        return Command(  # (2)!
            # highlight-next-line
            goto=agent_name,  # (3)!
            # highlight-next-line
            update={"messages": state["messages"] + [tool_message]},  # (4)!
            # highlight-next-line
            graph=Command.PARENT,  # (5)!
        )
    return handoff_tool

# 移交工具
transfer_to_hotel_assistant = create_handoff_tool(
    agent_name="hotel_assistant",
    description="Transfer user to the hotel-booking assistant.",
)
transfer_to_flight_assistant = create_handoff_tool(
    agent_name="flight_assistant",
    description="Transfer user to the flight-booking assistant.",
)

# 简单工具
def book_hotel(hotel_name: str):
    """Book a hotel"""
    return f"Successfully booked a stay at {hotel_name}."

def book_flight(from_airport: str, to_airport: str):
    """Book a flight"""
    return f"Successfully booked a flight from {from_airport} to {to_airport}."

# 定义智能体
flight_assistant = create_react_agent(
    model="anthropic:claude-3-5-sonnet-latest",
    # highlight-next-line
    tools=[book_flight, transfer_to_hotel_assistant],
    prompt="You are a flight booking assistant",
    # highlight-next-line
    name="flight_assistant"
)
hotel_assistant = create_react_agent(
    model="anthropic:claude-3-5-sonnet-latest",
    # highlight-next-line
    tools=[book_hotel, transfer_to_flight_assistant],
    prompt="You are a hotel booking assistant",
    # highlight-next-line
    name="hotel_assistant"
)

# 定义多智能体图
multi_agent_graph = (
    StateGraph(MessagesState)
    .add_node(flight_assistant)
    .add_node(hotel_assistant)
    .add_edge(START, "flight_assistant")
    .compile()
)

# 运行多智能体系统
for chunk in multi_agent_graph.stream(
    {
        "messages": [
            {
                "role": "user",
                "content": "book a flight from BOS to JFK and a stay at McKittrick Hotel"
            }
        ]
    }
):
    print(chunk)
    print("\n")
```

1. 访问智能体状态
2. `Command`原语允许将状态更新和节点转移指定为单一操作，适用于实现控制权移交
3. 要移交到的智能体或节点名称
4. 将智能体的消息**添加**到父图的**状态**中作为移交部分。下一个智能体将看到父状态
5. 指示LangGraph需要导航至**父**多智能体图中的智能体节点

!!! 注意
    该移交实现基于以下假设：

    - 每个智能体接收多智能体系统中所有智能体的完整消息历史作为输入
    - 每个智能体将其内部消息历史输出到多智能体系统的全局消息历史中

    参阅LangGraph的[监管者](https://github.com/langchain-ai/langgraph-supervisor-py#customizing-handoff-tools)和[群体](https://github.com/langchain-ai/langgraph-swarm-py#customizing-handoff-tools)文档了解如何定制移交机制。