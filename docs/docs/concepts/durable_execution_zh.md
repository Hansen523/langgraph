# 持久化执行

**持久化执行**是一种技术，其中进程或工作流在关键点保存其进度，使其能够暂停并在稍后从暂停的地方继续执行。这在需要[人机交互](./human_in_the_loop.md)的场景中特别有用，用户可以在继续之前检查、验证或修改流程，并且在可能遇到中断或错误的长时任务中（例如，调用LLM时超时）也非常有用。通过保存已完成的工作，持久化执行使得流程可以在不重新处理先前步骤的情况下继续执行——即使在经过较长时间延迟后（例如，一周后）。

LangGraph内置的[持久化](./persistence.md)层为工作流提供了持久化执行功能，确保每个执行步骤的状态都被保存到持久化存储中。这种能力保证了如果工作流被中断——无论是由于系统故障还是[人机交互](./human_in_the_loop.md)——它都可以从最后记录的状态继续执行。

!!! 提示

    如果您正在使用带有检查点的LangGraph，那么您已经启用了持久化执行。您可以在任何点暂停和恢复工作流，即使在中断或故障之后。
    为了充分利用持久化执行，请确保您的工作流设计为[确定性](#determinism-and-consistent-replay)和[幂等性](#determinism-and-consistent-replay)，并将任何副作用或非确定性操作封装在[任务](./functional_api.md#task)中。您可以在[StateGraph（图API）](./low_level.md)和[功能API](./functional_api.md)中使用[任务](./functional_api.md#task)。

## 要求

要在LangGraph中利用持久化执行，您需要：

1. 通过指定将保存工作流进度的[检查点](./persistence.md#checkpointer-libraries)来启用工作流中的[持久化](./persistence.md)。
2. 在执行工作流时指定一个[线程标识符](./persistence.md#threads)。这将跟踪特定工作流实例的执行历史。
3. 将任何非确定性操作（例如，随机数生成）或带有副作用的操作（例如，文件写入、API调用）封装在[任务][langgraph.func.task]中，以确保当工作流恢复时，这些操作不会重复执行，而是从持久化层检索其结果。更多信息请参见[确定性和一致性重放](#determinism-and-consistent-replay)。

## 确定性和一致性重放

当您恢复工作流运行时，代码**不会**从执行停止的**同一行代码**恢复；相反，它将识别一个适当的[起始点](#starting-points-for-resuming-workflows)，从该点继续执行。这意味着工作流将从[起始点](#starting-points-for-resuming-workflows)开始重放所有步骤，直到达到停止点。

因此，当您为持久化执行编写工作流时，必须将任何非确定性操作（例如，随机数生成）和任何带有副作用的操作（例如，文件写入、API调用）封装在[任务](./functional_api.md#task)或[节点](./low_level.md#nodes)中。

为了确保您的工作流是确定性的并且可以一致地重放，请遵循以下指南：

- **避免重复工作**：如果[节点](./low_level.md#nodes)包含多个带有副作用的操作（例如，日志记录、文件写入或网络调用），请将每个操作封装在一个单独的**任务**中。这确保在工作流恢复时，操作不会重复执行，而是从持久化层检索其结果。
- **封装非确定性操作**：将可能产生非确定性结果的代码（例如，随机数生成）封装在**任务**或**节点**中。这确保在恢复时，工作流按照记录的确切步骤序列执行，并产生相同的结果。
- **使用幂等操作**：尽可能确保副作用操作（例如，API调用、文件写入）是幂等的。这意味着如果在工作流失败后重试操作，它将与第一次执行时产生相同的效果。这对于导致数据写入的操作尤为重要。如果**任务**开始但未能成功完成，工作流的恢复将重新运行**任务**，依赖记录的结果来保持一致性。使用幂等键或验证现有结果，以避免意外的重复，确保工作流执行顺畅且可预测。

有关需要避免的常见陷阱的示例，请参见功能API中的[常见陷阱](./functional_api.md#common-pitfalls)部分，其中展示了如何使用**任务**构建代码以避免这些问题。相同的原则适用于[StateGraph（图API）][langgraph.graph.state.StateGraph]。

## 在节点中使用任务

如果[节点](./low_level.md#nodes)包含多个操作，您可能会发现将每个操作转换为**任务**比将操作重构为单独的节点更容易。

=== "原始"

    ```python
    from typing import NotRequired
    from typing_extensions import TypedDict
    import uuid

    from langgraph.checkpoint.memory import MemorySaver
    from langgraph.graph import StateGraph, START, END
    import requests

    # 定义一个TypedDict来表示状态
    class State(TypedDict):
        url: str
        result: NotRequired[str]

    def call_api(state: State):
        """示例节点，进行API请求。"""
        # highlight-next-line
        result = requests.get(state['url']).text[:100]  # 副作用
        return {
            "result": result
        }

    # 创建一个StateGraph构建器，并为call_api函数添加一个节点
    builder = StateGraph(State)
    builder.add_node("call_api", call_api)

    # 将开始和结束节点连接到call_api节点
    builder.add_edge(START, "call_api")
    builder.add_edge("call_api", END)

    # 指定一个检查点
    checkpointer = MemorySaver()

    # 使用检查点编译图
    graph = builder.compile(checkpointer=checkpointer)

    # 定义一个带有线程ID的配置。
    thread_id = uuid.uuid4()
    config = {"configurable": {"thread_id": thread_id}}

    # 调用图
    graph.invoke({"url": "https://www.example.com"}, config)
    ```

=== "使用任务"

    ```python
    from typing import NotRequired
    from typing_extensions import TypedDict
    import uuid

    from langgraph.checkpoint.memory import MemorySaver
    from langgraph.func import task
    from langgraph.graph import StateGraph, START, END
    import requests

    # 定义一个TypedDict来表示状态
    class State(TypedDict):
        urls: list[str]
        result: NotRequired[list[str]]


    @task
    def _make_request(url: str):
        """进行请求。"""
        # highlight-next-line
        return requests.get(url).text[:100]

    def call_api(state: State):
        """示例节点，进行API请求。"""
        # highlight-next-line
        requests = [_make_request(url) for url in state['urls']]
        results = [request.result() for request in requests]
        return {
            "results": results
        }

    # 创建一个StateGraph构建器，并为call_api函数添加一个节点
    builder = StateGraph(State)
    builder.add_node("call_api", call_api)

    # 将开始和结束节点连接到call_api节点
    builder.add_edge(START, "call_api")
    builder.add_edge("call_api", END)

    # 指定一个检查点
    checkpointer = MemorySaver()

    # 使用检查点编译图
    graph = builder.compile(checkpointer=checkpointer)

    # 定义一个带有线程ID的配置。
    thread_id = uuid.uuid4()
    config = {"configurable": {"thread_id": thread_id}}

    # 调用图
    graph.invoke({"urls": ["https://www.example.com"]}, config)
    ```

## 恢复工作流

一旦在您的工作流中启用了持久化执行，您可以在以下场景中恢复执行：

- **暂停和恢复工作流**：使用[interrupt][langgraph.types.interrupt]函数在特定点暂停工作流，并使用[Command][langgraph.types.Command]原语使用更新后的状态恢复它。有关更多详细信息，请参见[**人机交互**](./human_in_the_loop.md)。
- **从故障中恢复**：在异常（例如，LLM提供商中断）后自动从最后一个成功的检查点恢复工作流。这涉及通过提供相同的线程标识符来执行工作流，并提供`None`作为输入值（参见功能API中的[此示例](./functional_api.md#resuming-after-an-error)）。

## 恢复工作流的起始点

* 如果您使用的是[StateGraph（图API）][langgraph.graph.state.StateGraph]，则起始点是执行停止的[节点](./low_level.md#nodes)的开始。
* 如果您在节点内进行子图调用，则起始点将是调用被中断的子图的**父节点**。
在子图内，起始点将是执行停止的特定[节点](./low_level.md#nodes)。
* 如果您使用的是功能API，则起始点是执行停止的[入口点](./functional_api.md#entrypoint)的开始。