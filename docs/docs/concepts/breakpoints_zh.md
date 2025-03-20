# 断点

断点在特定点暂停图的执行，并允许逐步执行。断点由LangGraph的[**持久层**](./persistence.md)提供支持，该层在每个图步骤后保存状态。断点也可以用于启用[**人机交互**](./human_in_the_loop.md)工作流，但我们建议为此目的使用[`interrupt`函数](./human_in_the_loop.md#interrupt)。

## 要求

要使用断点，您需要：

1. [**指定一个检查点**](persistence.md#checkpoints)以在每个步骤后保存图状态。
2. [**设置断点**](#setting-breakpoints)以指定执行应在何处暂停。
3. **使用[**线程ID**](./persistence.md#threads)运行图**以在断点处暂停执行。
4. **使用`invoke`/`ainvoke`/`stream`/`astream`恢复执行**（请参阅[**`Command`原语**](./human_in_the_loop.md#the-command-primitive)）。

## 设置断点

您可以在两个地方设置断点：

1. **在节点执行之前**或**之后**通过在**编译时**或**运行时**设置断点。我们称之为[**静态断点**](#static-breakpoints)。
2. **在节点内部**使用[`NodeInterrupt`异常](#nodeinterrupt-exception)。
 
### 静态断点

静态断点在节点执行**之前**或**之后**触发。您可以通过在**“编译”时**或**运行时**指定`interrupt_before`和`interrupt_after`来设置静态断点。

=== "编译时"

    ```python
    graph = graph_builder.compile(
        interrupt_before=["node_a"], 
        interrupt_after=["node_b", "node_c"],
        checkpointer=..., # 指定一个检查点
    )

    thread_config = {
        "configurable": {
            "thread_id": "some_thread"
        }
    }

    # 运行图直到断点
    graph.invoke(inputs, config=thread_config)

    # 根据用户输入可选地更新图状态
    graph.update_state(update, config=thread_config)

    # 恢复图
    graph.invoke(None, config=thread_config)
    ```

=== "运行时"

    ```python
    graph.invoke(
        inputs, 
        config={"configurable": {"thread_id": "some_thread"}}, 
        interrupt_before=["node_a"], 
        interrupt_after=["node_b", "node_c"]
    )

    thread_config = {
        "configurable": {
            "thread_id": "some_thread"
        }
    }

    # 运行图直到断点
    graph.invoke(inputs, config=thread_config)

    # 根据用户输入可选地更新图状态
    graph.update_state(update, config=thread_config)

    # 恢复图
    graph.invoke(None, config=thread_config)
    ```

    !!! 注意

        您不能在运行时为**子图**设置静态断点。
        如果您有一个子图，必须在编译时设置断点。

静态断点在调试时特别有用，如果您想逐步执行图的一个节点或想在图执行的特定节点处暂停。

### `NodeInterrupt`异常

我们建议您[**使用`interrupt`函数代替**][langgraph.types.interrupt]`NodeInterrupt`异常，如果您正在尝试实现[人机交互](./human_in_the_loop.md)工作流。`interrupt`函数更易于使用且更灵活。

??? node "`NodeInterrupt`异常"

    开发人员可以定义一些*条件*，必须在满足这些条件时触发断点。这种*动态断点*的概念在开发人员希望在*特定条件*下暂停图时非常有用。这使用了一个`NodeInterrupt`，这是一种特殊类型的异常，可以根据某些条件从节点内部抛出。例如，我们可以定义一个在`input`长度超过5个字符时触发的动态断点。

    ```python
    def my_node(state: State) -> State:
        if len(state['input']) > 5:
            raise NodeInterrupt(f"Received input that is longer than 5 characters: {state['input']}")

        return state
    ```


    假设我们运行图时输入触发了动态断点，然后尝试通过传递`None`作为输入来恢复图执行。

    ```python
    # 在遇到动态断点后，尝试继续图执行而不更改状态 
    for event in graph.stream(None, thread_config, stream_mode="values"):
        print(event)
    ```

    图将再次*中断*，因为此节点将使用相同的图状态*重新运行*。我们需要更改图状态，使得触发动态断点的条件不再满足。因此，我们可以简单地将图状态编辑为满足动态断点条件的输入（< 5个字符）并重新运行节点。

    ```python 
    # 更新状态以通过动态断点
    graph.update_state(config=thread_config, values={"input": "foo"})
    for event in graph.stream(None, thread_config, stream_mode="values"):
        print(event)
    ```

    或者，如果我们想保留当前输入并跳过执行检查的节点（`my_node`）怎么办？为此，我们可以简单地执行图更新，使用`as_node="my_node"`并传递`None`作为值。这不会更新图状态，但会以`my_node`运行更新，从而有效地跳过节点并绕过动态断点。

    ```python
    # 此更新将完全跳过节点`my_node`
    graph.update_state(config=thread_config, values=None, as_node="my_node")
    for event in graph.stream(None, thread_config, stream_mode="values"):
        print(event)
    ```

## 其他资源 📚

- [**概念指南：持久化**](persistence.md)：阅读持久化指南以获取更多关于持久化的上下文。
- [**概念指南：人机交互**](human_in_the_loop.md)：阅读人机交互指南以获取更多关于使用断点将人类反馈集成到LangGraph应用程序中的上下文。
- [**如何查看和更新过去的图状态**](../how-tos/human_in_the_loop/time-travel.ipynb)：逐步说明如何处理图状态，演示**重放**和**分支**操作。