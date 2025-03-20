# LangGraph的运行时（Pregel）

[Pregel][langgraph.pregel.Pregel] 实现了LangGraph的运行时，管理LangGraph应用程序的执行。

编译一个[StateGraph][langgraph.graph.StateGraph]或创建一个[入口点][langgraph.func.entrypoint]会生成一个[Pregel][langgraph.pregel.Pregel]实例，可以通过输入来调用。

本指南从高层次解释了运行时，并提供了直接使用Pregel实现应用程序的说明。

> **注意:** [Pregel][langgraph.pregel.Pregel]运行时以[Google的Pregel算法](https://research.google/pubs/pub37252/)命名，该算法描述了使用图进行大规模并行计算的有效方法。

## 概述

在LangGraph中，Pregel将[**actor模型**](https://en.wikipedia.org/wiki/Actor_model)和**通道**结合到一个应用程序中。**Actors**从通道读取数据并将数据写入通道。Pregel按照**Pregel算法**/**Bulk Synchronous Parallel**模型将应用程序的执行组织为多个步骤。

每个步骤由三个阶段组成：

- **计划**：确定在此步骤中要执行的**actors**。例如，在第一步中，选择订阅特殊**input**通道的**actors**；在后续步骤中，选择订阅上一步中更新的通道的**actors**。
- **执行**：并行执行所有选定的**actors**，直到所有完成，或其中一个失败，或达到超时。在此阶段，通道更新对actors不可见，直到下一步。
- **更新**：使用此步骤中**actors**写入的值更新通道。

重复执行，直到没有**actors**被选中执行，或达到最大步骤数。

## Actors

一个**actor**是一个[PregelNode][langgraph.pregel.read.PregelNode]。它订阅通道，从中读取数据，并将数据写入其中。可以将其视为Pregel算法中的**actor**。[PregelNodes][langgraph.pregel.read.PregelNode]实现了LangChain的Runnable接口。

## 通道

通道用于在actors（PregelNodes）之间进行通信。每个通道都有一个值类型、一个更新类型和一个更新函数——该函数接收一系列更新并修改存储的值。通道可用于将数据从一个链发送到另一个链，或用于将数据从一个链发送到未来的步骤中的自身。LangGraph提供了许多内置通道：

### 基本通道：LastValue和Topic

- [LastValue][langgraph.channels.LastValue]：默认通道，存储发送到通道的最后一个值，适用于输入和输出值，或用于将数据从一个步骤发送到下一个步骤。
- [Topic][langgraph.channels.Topic]：可配置的PubSub主题，适用于在**actors**之间发送多个值，或用于累积输出。可以配置为去重或在多个步骤中累积值。

### 高级通道：Context和BinaryOperatorAggregate

- `Context`：暴露上下文管理器的值，管理其生命周期。适用于访问需要设置和/或清理的外部资源；例如，`client = Context(httpx.Client)`。
- [BinaryOperatorAggregate][langgraph.channels.BinaryOperatorAggregate]：存储一个持久值，通过将二元运算符应用于当前值和发送到通道的每个更新来更新，适用于在多个步骤中计算聚合；例如，`total = BinaryOperatorAggregate(int, operator.add)`

## 示例

虽然大多数用户将通过[StateGraph][langgraph.graph.StateGraph] API或[入口点][langgraph.func.entrypoint]装饰器与Pregel交互，但也可以直接与Pregel交互。

以下是几个不同的示例，以帮助您理解Pregel API。

=== "单节点"

    ```python

    from langgraph.channels import EphemeralValue
    from langgraph.pregel import Pregel, Channel 

    node1 = (
        Channel.subscribe_to("a")
        | (lambda x: x + x)
        | Channel.write_to("b")
    )

    app = Pregel(
        nodes={"node1": node1},
        channels={
            "a": EphemeralValue(str),
            "b": EphemeralValue(str),
        },
        input_channels=["a"],
        output_channels=["b"],
    )

    app.invoke({"a": "foo"})
    ```

    ```con
    {'b': 'foofoo'}
    ```

=== "多节点"

    ```python
    from langgraph.channels import LastValue, EphemeralValue
    from langgraph.pregel import Pregel, Channel 

    node1 = (
        Channel.subscribe_to("a")
        | (lambda x: x + x)
        | Channel.write_to("b")
    )

    node2 = (
        Channel.subscribe_to("b")
        | (lambda x: x + x)
        | Channel.write_to("c")
    )


    app = Pregel(
        nodes={"node1": node1, "node2": node2},
        channels={
            "a": EphemeralValue(str),
            "b": LastValue(str),
            "c": EphemeralValue(str),
        },
        input_channels=["a"],
        output_channels=["b", "c"],
    )

    app.invoke({"a": "foo"})
    ```

    ```con
    {'b': 'foofoo', 'c': 'foofoofoofoo'}
    ```

=== "Topic"

    ```python
    from langgraph.channels import EphemeralValue, Topic
    from langgraph.pregel import Pregel, Channel 

    node1 = (
        Channel.subscribe_to("a")
        | (lambda x: x + x)
        | {
            "b": Channel.write_to("b"),
            "c": Channel.write_to("c")
        }
    )

    node2 = (
        Channel.subscribe_to("b")
        | (lambda x: x + x)
        | {
            "c": Channel.write_to("c"),
        }
    )

    app = Pregel(
        nodes={"node1": node1, "node2": node2},
        channels={
            "a": EphemeralValue(str),
            "b": EphemeralValue(str),
            "c": Topic(str, accumulate=True),
        },
        input_channels=["a"],
        output_channels=["c"],
    )

    app.invoke({"a": "foo"})
    ```

    ```pycon
    {'c': ['foofoo', 'foofoofoofoo']}
    ```

=== "BinaryOperatorAggregate"

    此示例演示了如何使用BinaryOperatorAggregate通道实现一个reducer。

    ```python
    from langgraph.channels import EphemeralValue, BinaryOperatorAggregate
    from langgraph.pregel import Pregel, Channel


    node1 = (
        Channel.subscribe_to("a")
        | (lambda x: x + x)
        | {
            "b": Channel.write_to("b"),
            "c": Channel.write_to("c")
        }
    )

    node2 = (
        Channel.subscribe_to("b")
        | (lambda x: x + x)
        | {
            "c": Channel.write_to("c"),
        }
    )

    def reducer(current, update):
        if current:
            return current + " | " + "update"
        else:
            return update

    app = Pregel(
        nodes={"node1": node1, "node2": node2},
        channels={
            "a": EphemeralValue(str),
            "b": EphemeralValue(str),
            "c": BinaryOperatorAggregate(str, operator=reducer),
        },
        input_channels=["a"],
        output_channels=["c"],
    )

    app.invoke({"a": "foo"})
    ```
    
    
=== "循环"

    此示例演示了如何通过让一个链写入它订阅的通道来在图中引入循环。执行将继续，直到向通道写入None值。

    ```python
    from langgraph.channels import EphemeralValue
    from langgraph.pregel import Pregel, Channel, ChannelWrite, ChannelWriteEntry

    example_node = (
        Channel.subscribe_to("value")
        | (lambda x: x + x if len(x) < 10 else None)
        | ChannelWrite(writes=[ChannelWriteEntry(channel="value", skip_none=True)])
    )

    app = Pregel(
        nodes={"example_node": example_node},
        channels={
            "value": EphemeralValue(str),
        },
        input_channels=["value"],
        output_channels=["value"],
    )

    app.invoke({"value": "a"})
    ```

    ```pycon
    {'value': 'aaaaaaaaaaaaaaaa'}
    ```

## 高级API

LangGraph提供了两个高级API来创建Pregel应用程序：[StateGraph（图API）](./low_level.md)和[功能API](functional_api.md)。


=== "StateGraph（图API）"

    [StateGraph（图API）][langgraph.graph.StateGraph]是一个更高级的抽象，简化了Pregel应用程序的创建。它允许您定义节点和边的图。当您编译图时，StateGraph API会自动为您创建Pregel应用程序。

    ```python
    from typing import TypedDict, Optional

    from langgraph.constants import START
    from langgraph.graph import StateGraph

    class Essay(TypedDict):
        topic: str
        content: Optional[str]
        score: Optional[float]

    def write_essay(essay: Essay):
        return {
            "content": f"Essay about {essay['topic']}",
        }

    def score_essay(essay: Essay):
        return {
            "score": 10
        }

    builder = StateGraph(Essay)
    builder.add_node(write_essay)
    builder.add_node(score_essay)
    builder.add_edge(START, "write_essay")

    # 编译图。 
    # 这将返回一个Pregel实例。
    graph = builder.compile()
    ```

    编译后的Pregel实例将与一组节点和通道相关联。您可以通过打印它们来检查节点和通道。

    ```python
    print(graph.nodes)
    ```

    您将看到类似以下内容：

    ```pycon 
    {'__start__': <langgraph.pregel.read.PregelNode at 0x7d05e3ba1810>,
     'write_essay': <langgraph.pregel.read.PregelNode at 0x7d05e3ba14d0>,
     'score_essay': <langgraph.pregel.read.PregelNode at 0x7d05e3ba1710>}
     ```

    ```python
    print(graph.channels)
    ```

    您应该看到类似以下内容

    ```pycon
    {'topic': <langgraph.channels.last_value.LastValue at 0x7d05e3294d80>,
     'content': <langgraph.channels.last_value.LastValue at 0x7d05e3295040>,
     'score': <langgraph.channels.last_value.LastValue at 0x7d05e3295980>,
     '__start__': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e3297e00>,
     'write_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e32960c0>,
     'score_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e2d8ab80>,
     'branch:__start__:__self__:write_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e32941c0>,
     'branch:__start__:__self__:score_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e2d88800>,
     'branch:write_essay:__self__:write_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e3295ec0>,
     'branch:write_essay:__self__:score_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e2d8ac00>,
     'branch:score_essay:__self__:write_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e2d89700>,
     'branch:score_essay:__self__:score_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e2d8b400>,
     'start:write_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e2d8b280>}
    ```

=== "功能API"

    在[功能API](functional_api.md)中，您可以使用[`entrypoint`][langgraph.func.entrypoint]来创建Pregel应用程序。`entrypoint`装饰器允许您定义一个接收输入并返回输出的函数。 

    ```python
    from typing import TypedDict, Optional

    from langgraph.checkpoint.memory import InMemorySaver
    from langgraph.f