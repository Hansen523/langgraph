# 功能API

## 概述

**功能API** 允许您将LangGraph的关键功能——[持久化](./persistence.md)、[记忆](./memory.md)、[人机交互](./human_in_the_loop.md)和[流式处理](./streaming.md)——添加到您的应用程序中，同时只需对现有代码进行最小改动。

它旨在将这些功能集成到可能使用标准语言原语进行分支和控制流的现有代码中，例如`if`语句、`for`循环和函数调用。与许多需要将代码重构为显式管道或DAG的数据编排框架不同，功能API允许您在不强制执行严格执行模型的情况下集成这些功能。

功能API使用两个关键构建块：

- **`@entrypoint`** – 将函数标记为工作流的起点，封装逻辑并管理执行流，包括处理长时间运行的任务和中断。
- **`@task`** – 表示一个离散的工作单元，例如API调用或数据处理步骤，可以在入口点内异步执行。任务返回一个类似未来的对象，可以同步等待或解析。

这为构建具有状态管理和流式处理的工作流提供了一个最小的抽象。

!!! 提示

    对于更喜欢声明式方法的用户，LangGraph的[图API](./low_level.md)允许您使用图范式定义工作流。两个API共享相同的底层运行时，因此您可以在同一个应用程序中一起使用它们。
    请参阅[功能API vs. 图API](#functional-api-vs-graph-api)部分以比较这两种范式。

## 示例

下面我们演示一个简单的应用程序，该应用程序撰写一篇文章并[中断](human_in_the_loop.md)以请求人工审核。

```python
from langgraph.func import entrypoint, task
from langgraph.types import interrupt

@task
def write_essay(topic: str) -> str:
    """撰写关于给定主题的文章。"""
    time.sleep(1) # 长时间运行任务的占位符。
    return f"关于主题的文章: {topic}"

@entrypoint(checkpointer=MemorySaver())
def workflow(topic: str) -> dict:
    """一个简单的工作流，撰写文章并请求审核。"""
    essay = write_essay("cat").result()
    is_approved = interrupt({
        # 提供给中断的任何json可序列化负载。
        # 当从工作流流式传输数据时，它将在客户端作为中断显示。
        "essay": essay, # 我们想要审核的文章。
        # 我们可以添加任何我们需要的额外信息。
        # 例如，引入一个名为“action”的键，并提供一些指令。
        "action": "请批准/拒绝文章",
    })

    return {
        "essay": essay, # 生成的文章
        "is_approved": is_approved, # 来自HIL的响应
    }
```

??? 示例 "详细解释"

    此工作流将撰写一篇关于“cat”主题的文章，然后暂停以获取人工审核。工作流可以无限期中断，直到提供审核。

    当工作流恢复时，它会从头开始执行，但由于`write_essay`任务的结果已经保存，任务结果将从检查点加载，而不是重新计算。

    ```python
    import time
    import uuid

    from langgraph.func import entrypoint, task
    from langgraph.types import interrupt
    from langgraph.checkpoint.memory import MemorySaver

    @task
    def write_essay(topic: str) -> str:
        """撰写关于给定主题的文章。"""
        time.sleep(1) # 这是长时间运行任务的占位符。
        return f"关于主题的文章: {topic}"

    @entrypoint(checkpointer=MemorySaver())
    def workflow(topic: str) -> dict:
        """一个简单的工作流，撰写文章并请求审核。"""
        essay = write_essay("cat").result()
        is_approved = interrupt({
            # 提供给中断的任何json可序列化负载。
            # 当从工作流流式传输数据时，它将在客户端作为中断显示。
            "essay": essay, # 我们想要审核的文章。
            # 我们可以添加任何我们需要的额外信息。
            # 例如，引入一个名为“action”的键，并提供一些指令。
            "action": "请批准/拒绝文章",
        })

        return {
            "essay": essay, # 生成的文章
            "is_approved": is_approved, # 来自HIL的响应
        }

    thread_id = str(uuid.uuid4())

    config = {
        "configurable": {
            "thread_id": thread_id
        }
    }

    for item in workflow.stream("cat", config):
        print(item)
    ```

    ```pycon
    {'write_essay': '关于主题的文章: cat'}
    {'__interrupt__': (Interrupt(value={'essay': '关于主题的文章: cat', 'action': '请批准/拒绝文章'}, resumable=True, ns=['workflow:f7b8508b-21c0-8b4c-5958-4e8de74d2684'], when='during'),)}
    ```

    文章已撰写并准备审核。一旦提供审核，我们可以恢复工作流：

    ```python
    from langgraph.types import Command

    # 从用户获取审核（例如，通过UI）
    # 在这种情况下，我们使用布尔值，但这可以是任何json可序列化的值。
    human_review = True

    for item in workflow.stream(Command(resume=human_review), config):
        print(item)
    ```

    ```pycon
    {'workflow': {'essay': '关于主题的文章: cat', 'is_approved': False}}
    ```

    工作流已完成，审核已添加到文章中。

## 入口点

[`@entrypoint`][langgraph.func.entrypoint] 装饰器可用于从函数创建工作流。它封装工作流逻辑并管理执行流，包括处理*长时间运行的任务*和[中断](./low_level.md#interrupt)。

### 定义

**入口点** 通过使用`@entrypoint`装饰器装饰一个函数来定义。

该函数**必须接受一个位置参数**，该参数作为工作流输入。如果需要传递多个数据，请使用字典作为第一个参数的输入类型。

使用`entrypoint`装饰函数会生成一个[`Pregel`][langgraph.pregel.Pregel.stream]实例，该实例有助于管理工作流的执行（例如，处理流式传输、恢复和检查点）。

您通常希望将**检查点器**传递给`@entrypoint`装饰器，以启用持久化并使用**人机交互**等功能。

=== "同步"

    ```python
    from langgraph.func import entrypoint

    @entrypoint(checkpointer=checkpointer)
    def my_workflow(some_input: dict) -> int:
        # 一些可能涉及长时间运行任务（如API调用）的逻辑，
        # 并且可能会因人机交互而中断。
        ...
        return result
    ```

=== "异步"

    ```python
    from langgraph.func import entrypoint

    @entrypoint(checkpointer=checkpointer)
    async def my_workflow(some_input: dict) -> int:
        # 一些可能涉及长时间运行任务（如API调用）的逻辑，
        # 并且可能会因人机交互而中断
        ...
        return result 
    ```

!!! 重要 "序列化"

    **入口点**的**输入**和**输出**必须是JSON可序列化的，以支持检查点。请参阅[序列化](#serialization)部分以获取更多详细信息。

### 可注入参数

在声明`entrypoint`时，您可以请求访问在运行时自动注入的额外参数。这些参数包括：


| 参数    | 描述                                                                                                                                       |
|--------------|---------------------------------------------------------------------------------------------------------------------------------------------------|
| **previous** | 访问与给定线程的上一个`checkpoint`关联的状态。参见[状态管理](#state-management)。                   |
| **store**    | [BaseStore][langgraph.store.base.BaseStore]的实例。对[长期记忆](#long-term-memory)有用。                                     |
| **writer**   | 用于流式传输自定义数据，将自定义数据写入`custom`流。对[流式传输自定义数据](#streaming-custom-data)有用。               |
| **config**   | 用于访问运行时配置。参见[RunnableConfig](https://python.langchain.com/docs/concepts/runnables/#runnableconfig)以获取信息。 |

!!! 重要

    使用适当的名称和类型注释声明参数。

??? 示例 "请求可注入参数"

    ```python
    from langchain_core.runnables import RunnableConfig
    from langgraph.func import entrypoint
    from langgraph.store.base import BaseStore
    from langgraph.store.memory import InMemoryStore

    in_memory_store = InMemoryStore(...)  # InMemoryStore的实例，用于长期记忆

    @entrypoint(
        checkpointer=checkpointer,  # 指定检查点器
        store=in_memory_store  # 指定存储
    )  
    def my_workflow(
        some_input: dict,  # 输入（例如，通过`invoke`传递）
        *,
        previous: Any = None, # 用于短期记忆
        store: BaseStore,  # 用于长期记忆
        writer: StreamWriter,  # 用于流式传输自定义数据
        config: RunnableConfig  # 用于访问传递给入口点的配置
    ) -> ...:
    ```

### 执行

使用[`@entrypoint`](#entrypoint)会生成一个[`Pregel`][langgraph.pregel.Pregel.stream]对象，可以使用`invoke`、`ainvoke`、`stream`和`astream`方法执行。

=== "Invoke"

    ```python
    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }
    my_workflow.invoke(some_input, config)  # 同步等待结果
    ```

=== "Async Invoke"

    ```python
    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }
    await my_workflow.ainvoke(some_input, config)  # 异步等待结果
    ```

=== "Stream"
    
    ```python
    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    for chunk in my_workflow.stream(some_input, config):
        print(chunk)
    ```

=== "Async Stream"

    ```python
    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    async for chunk in my_workflow.astream(some_input, config):
        print(chunk)
    ```

### 恢复

在[中断][langgraph.types.interrupt]后恢复执行可以通过将**resume**值传递给[Command][langgraph.types.Command]原语来完成。

=== "Invoke"

    ```python
    from langgraph.types import Command

    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }
    
    my_workflow.invoke(Command(resume=some_resume_value), config)
    ```

=== "Async Invoke"

    ```python
    from langgraph.types import Command

    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }
    
    await my_workflow.ainvoke(Command(resume=some_resume_value), config)
    ```

=== "Stream"

    ```python
    from langgraph.types import Command

    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }
    
    for chunk in my_workflow.stream(Command(resume=some_resume_value), config):
        print(chunk)
    ```

=== "Async Stream"

    ```python
    from langgraph.types import Command

    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    async for chunk in my_workflow.astream(Command(resume=some_resume_value), config):
        print(chunk)
    ```

**在错误后恢复**


要在错误后恢复，使用`None`和相同的**线程id**（配置）运行`entrypoint`。

这假设底层**错误**已解决，并且可以成功继续执行。

=== "Invoke"

    ```python

    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }
    
    my_workflow.invoke(None, config)
    ```

=== "Async Invoke"

    ```python

    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }
    
    await my_workflow.ainvoke(None, config)
    ```

=== "Stream"

    ```python

    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }
    
    for chunk in my_workflow.stream(None, config):
        print(chunk)
    ```

=== "Async Stream"

    ```python

    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    async for chunk in my_workflow.astream(None, config):
        print(chunk)
    ```

### 状态管理

当`entrypoint`与`checkpointer`一起定义时，它会在相同**线程id**的连续调用之间存储信息到[检查点](persistence.md#checkpoints)中。

这允许使用`previous`参数访问前一次调用的状态。

默认情况下，`previous`参数是前一次调用的返回值。

```python
@entrypoint(checkpointer=checkpointer)
def my_workflow(number: int, *, previous: Any = None) -> int:
    previous = previous or 0
    return number + previous

config = {
    "configurable": {
        "thread_id": "some_thread_id"
    }
}

my_workflow.invoke(1, config)  # 1 (previous was None)
my_workflow.invoke(2, config)  # 3 (previous was 1 from the previous invocation)
```

#### `entrypoint.final`

[entrypoint.final][langgraph.func.entrypoint.final] 是一个特殊的原语，可以从入口点返回，并允许**解耦**保存在检查点中的值与入口点的**返回值**。

第一个值是入口点的返回值，第二个值是将保存在检查点中的值。类型注释为`entrypoint.final[return_type, save_type]`。

```python
@entrypoint(checkpointer=checkpointer)
def my_workflow(number: int, *, previous: Any = None) -> entrypoint.final[int, int]:
    previous = previous or 0
    # 这将返回前一个值给调用者，保存
    # 2 * number到检查点，将在下一次调用中用于`previous`参数。
    return entrypoint.final(value=previous, save=2 * number)

config = {
    "configurable": {
        "thread_id": "1"
    }
}

my_workflow.invoke(3, config)  # 0 (previous was None)
my_workflow.invoke(1, config)  # 6 (previous was 3 * 2 from the previous invocation)
```

## 任务

**任务** 表示一个离散的工作单元，例如API调用或数据处理步骤。它有两个关键特征：

* **异步执行**：任务设计为异步执行，允许多个操作并发运行而不阻塞。
* **检查点**：任务结果保存到检查点，允许从最后保存的状态恢复工作流。（参见[持久化](persistence.md)以获取更多详细信息）。

### 定义

任务使用`@task`装饰器定义，该装饰器包装一个常规的Python函数。

```python
from langgraph.func import task

@task()
def slow_computation(input_value):
    # 模拟长时间运行的操作
    ...
    return result
```

!!! 重要 "序列化"

    **任务**的**输出**必须是JSON可序列化的，以支持检查点。

### 执行

**任务** 只能从**入口点**、另一个**任务**或[状态图节点](./low_level.md#nodes)中调用。

任务*不能*直接从主应用程序代码中调用。

当您调用**任务**时，它会立即返回一个未来对象。未来是一个占位符，表示稍后可用的结果。

要获取**任务**的结果，您可以同步等待（使用`result()`）或异步等待（使用`await`）。


=== "同步调用"

    ```python
    @entrypoint(checkpointer=checkpointer)
    def my_workflow(some_input: int) -> int:
        future = slow_computation(some_input)
        return future.result()  # 同步等待结果
    ```

=== "异步调用"

    ```python
    @entrypoint(checkpointer=checkpointer)
    async def my_workflow(some_input: int) -> int:
        return await slow_computation(some_input)  # 异步等待结果
    ```

## 何时使用任务

**任务** 在以下场景中非常有用：

- **检查点**：当您需要将长时间运行操作的结果保存到检查点时，以便在恢复工作流时不需要重新计算。
- **人机交互**：如果您正在构建一个需要人工干预的工作流，您必须使用**任务**来封装任何随机性（例如，API调用），以确保工作流可以正确恢复。请参阅[确定性](#determinism)部分以获取更多详细信息。
- **并行执行**：对于I/O密集型任务，**任务**支持并行执行，允许多个操作并发运行而不阻塞（例如，调用多个API）。
- **可观察性**：将操作包装在**任务**中提供了一种跟踪工作流进度并使用[LangSmith](https://docs.smith.langchain.com/)监控单个操作执行的方法。
- **可重试的工作**：当工作需要重试以处理失败或不一致时，**任务**提供了一种封装和管理重试逻辑的方法。
 
## 序列化

LangGraph中的序列化有两个关键方面：

1. `@entrypoint`的输入和输出必须是JSON可序列化的。
2. `@task`的输出必须是JSON可序列化的。

这些要求对于启用检查点和工作流恢复是必要的。使用Python原语
如字典、列表、字符串、数字和布尔值，以确保您的输入和输出是可序列化的。

序列化确保工作流状态（例如任务结果和中间值）可以可靠地保存和恢复。这对于启用人机交互、容错和并行执行至关重要。

提供不可序列化的输入或输出将在工作流配置为使用检查点时导致运行时错误。

## 确定性

要使用**人机交互**等功能，任何随机性都应封装在**任务**中。这保证了当执行暂停（例如，人机交互）然后恢复时，它将遵循相同的*步骤序列*，即使