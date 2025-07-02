# 使用时间旅行功能

要在LangGraph中使用[时间旅行](../../concepts/time-travel.md)功能：

1. [运行图](#1-运行图)：使用[`invoke`][langgraph.graph.state.CompiledStateGraph.invoke]或[`stream`][langgraph.graph.state.CompiledStateGraph.stream]方法传入初始输入运行图。
2. [识别现有线程中的检查点](#2-识别检查点)：使用[`get_state_history()`][langgraph.graph.state.CompiledStateGraph.get_state_history]方法获取特定`thread_id`的执行历史记录，并定位所需的`checkpoint_id`。  
   或者，在希望暂停执行的节点前设置[断点](../../concepts/breakpoints.md)。然后可以找到记录到该断点的最新检查点。
3. [更新图状态（可选）](#3-更新状态-可选)：使用[`update_state`][langgraph.graph.state.CompiledStateGraph.update_state]方法修改检查点处的图状态，并从替代状态恢复执行。
4. [从检查点恢复执行](#4-从检查点恢复执行)：使用`invoke`或`stream`方法，传入输入`None`和包含适当`thread_id`与`checkpoint_id`的配置。

!!! tip

    关于时间旅行的概念概述，请参阅[时间旅行](../../concepts/time-travel.md)。

## 在工作流中

这个示例构建了一个简单的LangGraph工作流，生成一个笑话主题并使用LLM编写笑话。它展示了如何运行图、检索过去的执行检查点、可选地修改状态，以及从选定的检查点恢复执行以探索替代结果。

### 设置

首先我们需要安装所需的包

```python
%%capture --no-stderr
%pip install --quiet -U langgraph langchain_anthropic
```

接下来，我们需要为Anthropic（我们将使用的LLM）设置API密钥

```python
import getpass
import os


def _set_env(var: str):
    if not os.environ.get(var):
        os.environ[var] = getpass.getpass(f"{var}: ")


_set_env("ANTHROPIC_API_KEY")
```

<div class="admonition tip">
    <p class="admonition-title">为LangGraph开发设置<a href="https://smith.langchain.com">LangSmith</a></p>
    <p style="padding-top: 5px;">
        注册LangSmith可以快速发现问题并提高LangGraph项目的性能。LangSmith让您使用跟踪数据来调试、测试和监控使用LangGraph构建的LLM应用——阅读更多关于如何开始的信息<a href="https://docs.smith.langchain.com">这里</a>。
    </p>
</div>

```python
import uuid

from typing_extensions import TypedDict, NotRequired
from langgraph.graph import StateGraph, START, END
from langchain.chat_models import init_chat_model
from langgraph.checkpoint.memory import InMemorySaver


class State(TypedDict):
    topic: NotRequired[str]
    joke: NotRequired[str]


llm = init_chat_model(
    "anthropic:claude-3-7-sonnet-latest",
    temperature=0,
)


def generate_topic(state: State):
    """LLM调用生成笑话主题"""
    msg = llm.invoke("Give me a funny topic for a joke")
    return {"topic": msg.content}


def write_joke(state: State):
    """LLM调用根据主题编写笑话"""
    msg = llm.invoke(f"Write a short joke about {state['topic']}")
    return {"joke": msg.content}


# 构建工作流
workflow = StateGraph(State)

# 添加节点
workflow.add_node("generate_topic", generate_topic)
workflow.add_node("write_joke", write_joke)

# 添加连接节点的边
workflow.add_edge(START, "generate_topic")
workflow.add_edge("generate_topic", "write_joke")
workflow.add_edge("write_joke", END)

# 编译
checkpointer = InMemorySaver()
graph = workflow.compile(checkpointer=checkpointer)
graph
```

### 1. 运行图

```python
config = {
    "configurable": {
        "thread_id": uuid.uuid4(),
    }
}
state = graph.invoke({}, config)

print(state["topic"])
print()
print(state["joke"])
```

**输出:**
```
How about "The Secret Life of Socks in the Dryer"? You know, exploring the mysterious phenomenon of how socks go into the laundry as pairs but come out as singles. Where do they go? Are they starting new lives elsewhere? Is there a sock paradise we don't know about? There's a lot of comedic potential in the everyday mystery that unites us all!

# The Secret Life of Socks in the Dryer

I finally discovered where all my missing socks go after the dryer. Turns out they're not missing at all—they've just eloped with someone else's socks from the laundromat to start new lives together.

My blue argyle is now living in Bermuda with a red polka dot, posting vacation photos on Sockstagram and sending me lint as alimony.
```

### 2. 识别检查点

```python
# 状态按逆时间顺序返回
states = list(graph.get_state_history(config))

for state in states:
    print(state.next)
    print(state.config["configurable"]["checkpoint_id"])
    print()
```

**输出:**
```
()
1f02ac4a-ec9f-6524-8002-8f7b0bbeed0e

('write_joke',)
1f02ac4a-ce2a-6494-8001-cb2e2d651227

('generate_topic',)
1f02ac4a-a4e0-630d-8000-b73c254ba748

('__start__',)
1f02ac4a-a4dd-665e-bfff-e6c8c44315d9
```

```python
# 这是倒数第二个状态（状态按时间顺序列出）
selected_state = states[1]
print(selected_state.next)
print(selected_state.values)
```

**输出:**
```
('write_joke',)
{'topic': 'How about "The Secret Life of Socks in the Dryer"? You know, exploring the mysterious phenomenon of how socks go into the laundry as pairs but come out as singles. Where do they go? Are they starting new lives elsewhere? Is there a sock paradise we don\\'t know about? There\\'s a lot of comedic potential in the everyday mystery that unites us all!'}
```

### 3. 更新状态（可选）

`update_state`将创建一个新的检查点。新检查点将与同一线程关联，但使用新的检查点ID。

```python
new_config = graph.update_state(selected_state.config, values={"topic": "chickens"})
print(new_config)
```

**输出:**
```
{'configurable': {'thread_id': 'c62e2e03-c27b-4cb6-8cea-ea9bfedae006', 'checkpoint_ns': '', 'checkpoint_id': '1f02ac4a-ecee-600b-8002-a1d21df32e4c'}}
```

### 4. 从检查点恢复执行

```python
graph.invoke(None, new_config)
```

**输出:**
```python
{'topic': 'chickens',
 'joke': 'Why did the chicken join a band?\n\nBecause it had excellent drumsticks!'}
```