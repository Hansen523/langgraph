# 如何使用RemoteGraph与部署交互

!!! info "前提条件"
    - [LangGraph平台](../concepts/langgraph_platform.md)
    - [LangGraph服务器](../concepts/langgraph_server.md)

`RemoteGraph`是一个接口，允许您像与常规本地定义的LangGraph图（例如`CompiledGraph`）一样与LangGraph平台部署进行交互。本指南将向您展示如何初始化`RemoteGraph`并与之交互。

## 初始化图

初始化`RemoteGraph`时，必须始终指定：

- `name`：您想与之交互的图的名称。这与您在部署的`langgraph.json`配置文件中使用的图名称相同。
- `api_key`：有效的LangSmith API密钥。可以设置为环境变量（`LANGSMITH_API_KEY`）或通过`api_key`参数直接传递。如果`LangGraphClient` / `SyncLangGraphClient`是用`api_key`参数初始化的，API密钥也可以通过`client` / `sync_client`参数提供。

此外，您必须提供以下之一：

- `url`：您想与之交互的部署的URL。如果传递`url`参数，将使用提供的URL、标头（如果提供）和默认配置值（例如超时等）创建同步和异步客户端。
- `client`：用于与部署异步交互的`LangGraphClient`实例（例如使用`.astream()`、`.ainvoke()`、`.aget_state()`、`.aupdate_state()`等）
- `sync_client`：用于与部署同步交互的`SyncLangGraphClient`实例（例如使用`.stream()`、`.invoke()`、`.get_state()`、`.update_state()`等）

!!! 注意

    如果您同时传递`client`或`sync_client`以及`url`参数，它们将优先于`url`参数。如果未提供`client` / `sync_client` / `url`参数中的任何一个，`RemoteGraph`将在运行时引发`ValueError`。

### 使用URL

=== "Python"

    ```python
    from langgraph.pregel.remote import RemoteGraph

    url = <DEPLOYMENT_URL>
    graph_name = "agent"
    remote_graph = RemoteGraph(graph_name, url=url)
    ```

=== "JavaScript"

    ```ts
    import { RemoteGraph } from "@langchain/langgraph/remote";

    const url = `<DEPLOYMENT_URL>`;
    const graphName = "agent";
    const remoteGraph = new RemoteGraph({ graphId: graphName, url });
    ```

### 使用客户端

=== "Python"

    ```python
    from langgraph_sdk import get_client, get_sync_client
    from langgraph.pregel.remote import RemoteGraph

    url = <DEPLOYMENT_URL>
    graph_name = "agent"
    client = get_client(url=url)
    sync_client = get_sync_client(url=url)
    remote_graph = RemoteGraph(graph_name, client=client, sync_client=sync_client)
    ```

=== "JavaScript"

    ```ts
    import { Client } from "@langchain/langgraph-sdk";
    import { RemoteGraph } from "@langchain/langgraph/remote";

    const client = new Client({ apiUrl: `<DEPLOYMENT_URL>` });
    const graphName = "agent";
    const remoteGraph = new RemoteGraph({ graphId: graphName, client });
    ```

## 调用图

由于`RemoteGraph`是一个实现了与`CompiledGraph`相同方法的`Runnable`，您可以像通常与编译图交互一样与之交互，即调用`.invoke()`、`.stream()`、`.get_state()`、`.update_state()`等（以及它们的异步对应方法）。

### 异步

!!! 注意

    要异步使用图，必须在初始化`RemoteGraph`时提供`url`或`client`。

=== "Python"

    ```python
    # 调用图
    result = await remote_graph.ainvoke({
        "messages": [{"role": "user", "content": "what's the weather in sf"}]
    })

    # 流式传输图的输出
    async for chunk in remote_graph.astream({
        "messages": [{"role": "user", "content": "what's the weather in la"}]
    }):
        print(chunk)
    ```

=== "JavaScript"

    ```ts
    // 调用图
    const result = await remoteGraph.invoke({
        messages: [{role: "user", content: "what's the weather in sf"}]
    })

    // 流式传输图的输出
    for await (const chunk of await remoteGraph.stream({
        messages: [{role: "user", content: "what's the weather in la"}]
    })):
        console.log(chunk)
    ```

### 同步

!!! 注意

    要同步使用图，必须在初始化`RemoteGraph`时提供`url`或`sync_client`。

=== "Python"

    ```python
    # 调用图
    result = remote_graph.invoke({
        "messages": [{"role": "user", "content": "what's the weather in sf"}]
    })

    # 流式传输图的输出
    for chunk in remote_graph.stream({
        "messages": [{"role": "user", "content": "what's the weather in la"}]
    }):
        print(chunk)
    ```

## 线程级持久性

默认情况下，图的运行（即`.invoke()`或`.stream()`调用）是无状态的——图的检查点和最终状态不会被持久化。如果您想持久化图运行的输出（例如，以启用人在环功能），可以创建一个线程并通过`config`参数提供线程ID，就像与常规编译图一样：

=== "Python"

    ```python
    from langgraph_sdk import get_sync_client
    url = <DEPLOYMENT_URL>
    graph_name = "agent"
    sync_client = get_sync_client(url=url)
    remote_graph = RemoteGraph(graph_name, url=url)

    # 创建一个线程（或使用现有线程）
    thread = sync_client.threads.create()

    # 使用线程配置调用图
    config = {"configurable": {"thread_id": thread["thread_id"]}}
    result = remote_graph.invoke({
        "messages": [{"role": "user", "content": "what's the weather in sf"}]
    }, config=config)

    # 验证状态是否被持久化到线程
    thread_state = remote_graph.get_state(config)
    print(thread_state)
    ```

=== "JavaScript"

    ```ts
    import { Client } from "@langchain/langgraph-sdk";
    import { RemoteGraph } from "@langchain/langgraph/remote";

    const url = `<DEPLOYMENT_URL>`;
    const graphName = "agent";
    const client = new Client({ apiUrl: url });
    const remoteGraph = new RemoteGraph({ graphId: graphName, url });

    // 创建一个线程（或使用现有线程）
    const thread = await client.threads.create();

    // 使用线程配置调用图
    const config = { configurable: { thread_id: thread.thread_id }};
    const result = await remoteGraph.invoke({
      messages: [{ role: "user", content: "what's the weather in sf" }],
    }, config);

    // 验证状态是否被持久化到线程
    const threadState = await remoteGraph.getState(config);
    console.log(threadState);
    ```

## 用作子图

!!! 注意

    如果您需要对具有`RemoteGraph`子图节点的图使用`checkpointer`，请确保使用UUID作为线程ID。

由于`RemoteGraph`的行为与常规`CompiledGraph`相同，它也可以用作另一个图中的子图。例如：

=== "Python"

    ```python
    from langgraph_sdk import get_sync_client
    from langgraph.graph import StateGraph, MessagesState, START
    from typing import TypedDict

    url = <DEPLOYMENT_URL>
    graph_name = "agent"
    remote_graph = RemoteGraph(graph_name, url=url)

    # 定义父图
    builder = StateGraph(MessagesState)
    # 直接添加远程图作为节点
    builder.add_node("child", remote_graph)
    builder.add_edge(START, "child")
    graph = builder.compile()

    # 调用父图
    result = graph.invoke({
        "messages": [{"role": "user", "content": "what's the weather in sf"}]
    })
    print(result)

    # 流式传输父图和子图的输出
    for chunk in graph.stream({
        "messages": [{"role": "user", "content": "what's the weather in sf"}]
    }, subgraphs=True):
        print(chunk)
    ```

=== "JavaScript"

    ```ts
    import { MessagesAnnotation, StateGraph, START } from "@langchain/langgraph";
    import { RemoteGraph } from "@langchain/langgraph/remote";

    const url = `<DEPLOYMENT_URL>`;
    const graphName = "agent";
    const remoteGraph = new RemoteGraph({ graphId: graphName, url });

    // 定义父图并直接添加远程图作为节点
    const graph = new StateGraph(MessagesAnnotation)
      .addNode("child", remoteGraph)
      .addEdge(START, "child")
      .compile()

    // 调用父图
    const result = await graph.invoke({
      messages: [{ role: "user", content: "what's the weather in sf" }]
    });
    console.log(result);

    // 流式传输父图和子图的输出
    for await (const chunk of await graph.stream({
      messages: [{ role: "user", content: "what's the weather in la" }]
    }, { subgraphs: true })) {
      console.log(chunk);
    }
    ```