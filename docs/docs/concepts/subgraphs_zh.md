# 子图

子图是一种作为[节点](./low_level.md#nodes)被嵌入到另一个[图](./low_level.md#graphs)中的图——这是LangGraph中封装概念的体现。子图允许您构建由多个组件组成的复杂系统，而这些组件本身也是图。

![子图](./img/subgraph.png)

使用子图的典型场景包括：
- 构建[多智能体系统](./multi_agent.md)
- 需要在多个图中复用一组节点时
- 不同团队需要独立开发图的不同部分时，可以将每个部分定义为子图。只要遵守子图接口（输入输出模式），父图可以在不了解子图细节的情况下构建

引入子图时的核心问题是父子图之间的通信机制，即图执行过程中如何传递[状态](./low_level.md#state)。主要有两种情况：

* 父子图在状态[模式](./low_level.md#state)中具有**共享状态键**。此时可以直接[将子图作为节点加入父图](../how-tos/subgraph.ipynb#shared-state-schemas)

    ```python
    from langgraph.graph import StateGraph, MessagesState, START

    # 子图

    def call_model(state: MessagesState):
        response = model.invoke(state["messages"])
        return {"messages": response}

    subgraph_builder = StateGraph(State)
    subgraph_builder.add_node(call_model)
    ...
    # highlight-next-line
    subgraph = subgraph_builder.compile()

    # 父图

    builder = StateGraph(State)
    # highlight-next-line
    builder.add_node("subgraph_node", subgraph)
    builder.add_edge(START, "subgraph_node")
    graph = builder.compile()
    ...
    graph.invoke({"messages": [{"role": "user", "content": "hi!"}]})
    ```

* 父子图具有**不同模式**（状态[模式](./low_level.md#state)无共享键）。此时需要[通过父图节点调用子图](../how-tos/subgraph.ipynb#different-state-schemas)，适用于父子图状态模式不同且需要转换状态的情况：

    ```python
    from typing_extensions import TypedDict, Annotated
    from langchain_core.messages import AnyMessage
    from langgraph.graph import StateGraph, MessagesState, START
    from langgraph.graph.message import add_messages

    class SubgraphMessagesState(TypedDict):
        # highlight-next-line
        subgraph_messages: Annotated[list[AnyMessage], add_messages]

    # 子图

    # highlight-next-line
    def call_model(state: SubgraphMessagesState):
        response = model.invoke(state["subgraph_messages"])
        return {"subgraph_messages": response}

    subgraph_builder = StateGraph(SubgraphMessagesState)
    subgraph_builder.add_node("call_model_from_subgraph", call_model)
    subgraph_builder.add_edge(START, "call_model_from_subgraph")
    ...
    # highlight-next-line
    subgraph = subgraph_builder.compile()

    # 父图

    def call_subgraph(state: MessagesState):
        response = subgraph.invoke({"subgraph_messages": state["messages"]})
        return {"messages": response["subgraph_messages"]}

    builder = StateGraph(State)
    # highlight-next-line
    builder.add_node("subgraph_node", call_subgraph)
    builder.add_edge(START, "subgraph_node")
    graph = builder.compile()
    ...
    graph.invoke({"messages": [{"role": "user", "content": "hi!"}]})
    ```