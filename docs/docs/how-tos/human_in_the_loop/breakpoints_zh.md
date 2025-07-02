# 设置断点

有两种设置断点的方式：

1. 在节点执行**前**或**后**通过**编译时**或**运行时**设置断点。我们称之为[**静态断点**](#static-breakpoints)。
2. 在节点**内部**使用`NodeInterrupt`异常。我们称之为[**动态断点**](#dynamic-breakpoints)。

要使用断点功能，您需要：

1. 指定一个[**检查点存储器**](../../concepts/persistence.md#checkpoints)来保存每一步的图状态。
2. **设置断点**来指定执行暂停的位置。
3. 使用[**线程ID**](../../concepts/persistence.md#threads)运行图，在断点处暂停执行。
4. 通过`invoke`/`ainvoke`/`stream`/`astream`传入`None`作为输入参数来**继续执行**。

!!! tip
    有关断点的概念性概述，请参阅[断点](../../concepts/breakpoints.md)。

## 静态断点

静态断点会在节点执行前或执行后触发。您可以通过在编译时或运行时指定`interrupt_before`和`interrupt_after`来设置静态断点。

静态断点特别适用于调试场景，比如您想逐步执行图操作，或者在特定节点处暂停图执行。

=== "编译时设置"

    ```python
    # highlight-next-line
    graph = graph_builder.compile( # (1)!
        # highlight-next-line
        interrupt_before=["node_a"], # (2)!
        # highlight-next-line
        interrupt_after=["node_b", "node_c"], # (3)!
        checkpointer=checkpointer, # (4)!
    )

    config = {
        "configurable": {
            "thread_id": "some_thread"
        }
    }

    # 运行图直到断点处
    graph.invoke(inputs, config=thread_config) # (5)!

    # 继续执行图
    graph.invoke(None, config=thread_config) # (6)!
    ```

    1. 断点在`compile`时设置。
    2. `interrupt_before`指定在节点执行前暂停的节点。
    3. `interrupt_after`指定在节点执行后暂停的节点。
    4. 必须配置检查点存储器才能启用断点功能。
    5. 运行图直到第一个断点。
    6. 传入`None`作为输入继续执行，直到下一个断点。

=== "运行时设置"

    ```python
    # highlight-next-line
    graph.invoke( # (1)!
        inputs, 
        # highlight-next-line
        interrupt_before=["node_a"], # (2)!
        # highlight-next-line
        interrupt_after=["node_b", "node_c"] # (3)!
        config={
            "configurable": {"thread_id": "some_thread"}
        }, 
    )

    config = {
        "configurable": {
            "thread_id": "some_thread"
        }
    }

    # 运行图直到断点处
    graph.invoke(inputs, config=config) # (4)!

    # 继续执行图
    graph.invoke(None, config=config) # (5)!
    ```

    1. 调用`graph.invoke`时传入`interrupt_before`和`interrupt_after`参数。这是运行时配置，每次调用都可以更改。
    2. `interrupt_before`指定在节点执行前暂停的节点。
    3. `interrupt_after`指定在节点执行后暂停的节点。
    4. 运行图直到第一个断点。
    5. 传入`None`作为输入继续执行，直到下一个断点。

    !!! note
        无法在运行时为**子图**设置静态断点。如果要对子图设置断点，必须在编译时设置。

??? example "静态断点设置示例"

    ```python
    from IPython.display import Image, display
    from typing_extensions import TypedDict
    
    from langgraph.checkpoint.memory import InMemorySaver 
    from langgraph.graph import StateGraph, START, END
    
    
    class State(TypedDict):
        input: str
    
    
    def step_1(state):
        print("---Step 1---")
        pass
    
    
    def step_2(state):
        print("---Step 2---")
        pass
    
    
    def step_3(state):
        print("---Step 3---")
        pass
    
    
    builder = StateGraph(State)
    builder.add_node("step_1", step_1)
    builder.add_node("step_2", step_2)
    builder.add_node("step_3", step_3)
    builder.add_edge(START, "step_1")
    builder.add_edge("step_1", "step_2")
    builder.add_edge("step_2", "step_3")
    builder.add_edge("step_3", END)
    
    # 设置检查点存储器
    checkpointer = InMemorySaver() # (1)!
    
    graph = builder.compile(
        checkpointer=checkpointer, # (2)!
        interrupt_before=["step_3"] # (3)!
    )
    
    # 可视化
    display(Image(graph.get_graph().draw_mermaid_png()))
    
    
    # 初始输入
    initial_input = {"input": "hello world"}
    
    # 线程配置
    thread = {"configurable": {"thread_id": "1"}}
    
    # 运行图直到第一个中断点
    for event in graph.stream(initial_input, thread, stream_mode="values"):
        print(event)
        
    # 此时图会在断点处暂停
    # 可以获取当前图状态
    print(graph.get_state(config))
    
    # 传入`None`继续执行
    for event in graph.stream(None, thread, stream_mode="values"):
        print(event)
    ```

## 动态断点

当需要根据条件在节点内部中断图执行时，可以使用动态断点。

```python
from langgraph.errors import NodeInterrupt

def step_2(state: State) -> State:
    # highlight-next-line
    if len(state["input"]) > 5:
        # highlight-next-line
        raise NodeInterrupt( # (1)!
            f"接收到的输入长度超过5个字符: {state['foo']}"
        )
    return state
```

1. 根据条件抛出NodeInterrupt异常。在这个例子中，当`input`属性长度超过5个字符时创建动态断点。

<details class="example"><summary>动态断点使用示例</summary>

```python
from typing_extensions import TypedDict
from IPython.display import Image, display

from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import MemorySaver
from langgraph.errors import NodeInterrupt


class State(TypedDict):
    input: str


def step_1(state: State) -> State:
    print("---Step 1---")
    return state


def step_2(state: State) -> State:
    # 当输入长度超过5个字符时抛出NodeInterrupt
    if len(state["input"]) > 5:
        raise NodeInterrupt(
            f"接收到的输入长度超过5个字符: {state['input']}"
        )
    print("---Step 2---")
    return state


def step_3(state: State) -> State:
    print("---Step 3---")
    return state


builder = StateGraph(State)
builder.add_node("step_1", step_1)
builder.add_node("step_2", step_2)
builder.add_node("step_3", step_3)
builder.add_edge(START, "step_1")
builder.add_edge("step_1", "step_2")
builder.add_edge("step_2", "step_3")
builder.add_edge("step_3", END)

# 设置存储器
memory = MemorySaver()

# 编译图
graph = builder.compile(checkpointer=memory)

# 可视化
display(Image(graph.get_graph().draw_mermaid_png()))
```

首先运行输入长度≤5字符的情况。这会忽略我们定义的断点条件，正常执行完图操作。

```python
initial_input = {"input": "hello"}
thread_config = {"configurable": {"thread_id": "1"}}

for event in graph.stream(initial_input, thread_config, stream_mode="values"):
    print(event)
```

检查图状态，可以看到没有待执行任务，图已执行完毕。

```python
state = graph.get_state(thread_config)
print(state.next)
print(state.tasks)
```

然后运行输入长度>5字符的情况。这会在`step_2`节点触发动态断点。

```python
initial_input = {"input": "hello world"}
thread_config = {"configurable": {"thread_id": "2"}}

# 运行图直到中断点
for event in graph.stream(initial_input, thread_config, stream_mode="values"):
    print(event)
```

可以看到图在`step_2`处停止。检查状态可以看到待执行节点(`step_2`)、中断节点(`step_2`)和中断详情。

```python
state = graph.get_state(thread_config)
print(state.next)
print(state.tasks)
```

如果尝试从断点继续执行，由于输入和图状态未改变，会再次中断。

```python
# 注意：与常规断点一样，传入None继续执行
for event in graph.stream(None, thread_config, stream_mode="values"):
    print(event)
```

```python
state = graph.get_state(thread_config)
print(state.next)
print(state.tasks)
```

</details>

## 子图断点设置

为子图设置断点的方法：

* 在**编译**子图时定义[静态断点](#static-breakpoints)
* 定义[动态断点](#dynamic-breakpoints)

<details class="example"><summary>子图断点设置示例</summary>

```python
from typing_extensions import TypedDict

from langgraph.graph import START, StateGraph
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.types import interrupt


class State(TypedDict):
    foo: str


def subgraph_node_1(state: State):
    return {"foo": state["foo"]}


subgraph_builder = StateGraph(State)
subgraph_builder.add_node(subgraph_node_1)
subgraph_builder.add_edge(START, "subgraph_node_1")

subgraph = subgraph_builder.compile(interrupt_before=["subgraph_node_1"])

builder = StateGraph(State)
builder.add_node("node_1", subgraph)  # 直接将子图作为一个节点
builder.add_edge(START, "node_1")

checkpointer = InMemorySaver()
graph = builder.compile(checkpointer=checkpointer)

config = {"configurable": {"thread_id": "1"}}

graph.invoke({"foo": ""}, config)

# 获取包含子图状态的完整状态
print(graph.get_state(config, subgraphs=True).tasks[0].state)

# 继续执行子图
graph.invoke(None, config)
```

</details>