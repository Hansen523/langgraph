# 流式API

[LangGraph SDK](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/python_sdk_ref/)允许您从LangGraph API服务器[流式传输输出](../../concepts/streaming.md)。

!!! 注意

    LangGraph SDK和LangGraph Server是[LangGraph平台](../../concepts/langgraph_platform.md)的一部分。

## 基本用法

基本用法示例：

=== "Python"

    ```python
    from langgraph_sdk import get_client
    client = get_client(url=<DEPLOYMENT_URL>, api_key=<API_KEY>)

    # 使用名为"agent"的部署图
    assistant_id = "agent"

    # 创建线程
    thread = await client.threads.create()
    thread_id = thread["thread_id"]

    # 创建流式运行
    # highlight-next-line
    async for chunk in client.runs.stream(
        thread_id,
        assistant_id,
        input=inputs,
        stream_mode="updates"
    ):
        print(chunk.data)
    ```

=== "JavaScript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";
    const client = new Client({ apiUrl: <DEPLOYMENT_URL>, apiKey: <API_KEY> });

    // 使用名为"agent"的部署图
    const assistantID = "agent";

    // 创建线程
    const thread = await client.threads.create();
    const threadID = thread["thread_id"];

    // 创建流式运行
    # highlight-next-line
    const streamResponse = client.runs.stream(
      threadID,
      assistantID,
      {
        input,
        streamMode: "updates"
      }
    );
    for await (const chunk of streamResponse) {
      console.log(chunk.data);
    }
    ```

### 支持的流模式

| 模式                             | 描述                                                                                                                                                                         | LangGraph库方法                                                                                 |
|----------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| [`values`](#stream-graph-state)  | 在每个[超级步骤](../../concepts/low_level.md#graphs)后流式传输完整的图状态。                                                                                            | `.stream()` / `.astream()` 配合 [`stream_mode="values"`](../../how-tos/streaming.md#stream-graph-state)  |
| [`updates`](#stream-graph-state) | 流式传输图每一步的状态更新。如果在同一步骤中进行了多次更新（例如运行多个节点），这些更新将分别流式传输。 | `.stream()` / `.astream()` 配合 [`stream_mode="updates"`](../../how-tos/streaming.md#stream-graph-state) |
| [`messages-tuple`](#messages)    | 流式传输调用LLM的图节点的LLM令牌和元数据（对聊天应用有用）。                                                                                 | `.stream()` / `.astream()` 配合 [`stream_mode="messages"`](../../how-tos/streaming.md#messages)          |
| [`debug`](#debug)                | 在图的执行过程中流式传输尽可能多的信息。                                                                                                      | `.stream()` / `.astream()` 配合 [`stream_mode="debug"`](../../how-tos/streaming.md#stream-graph-state)   |
| [`custom`](#stream-custom-data)  | 从图内部流式传输自定义数据                                                                                                                                          | `.stream()` / `.astream()` 配合 [`stream_mode="custom"`](../../how-tos/streaming.md#stream-custom-data)  |
| [`events`](#stream-events)       | 流式传输所有事件（包括图的状态）；主要适用于迁移大型LCEL应用时有用。                                                                                 | `.astream_events()`                                                                                      |

## 流式图状态

使用流模式`updates`和`values`来流式传输图执行过程中的状态。

* `updates`流式传输图每一步后的**状态更新**。
* `values`流式传输图每一步后的**完整状态值**。

## 子图

要在流式输出中包含[子图](../../concepts/subgraphs.md)的输出，可以在父图的`.stream()`方法中设置`subgraphs=True`。这将流式传输来自父图和任何子图的输出。

## 调试{#debug}

使用`debug`流模式在整个图执行过程中流式传输尽可能多的信息。流式输出包括节点名称以及完整状态。

## LLM令牌{#messages}

使用`messages-tuple`流模式从图的任何部分（包括节点、工具、子图或任务）**逐个令牌**流式传输大型语言模型（LLM）的输出。

## 流式自定义数据

要发送**自定义用户定义的数据**：

## 流式事件

流式传输所有事件，包括图的状态：

## 无状态运行

如果您不想在[检查点器](../../concepts/persistence.md)数据库中**持久化**流式运行的输出，可以在不创建线程的情况下创建无状态运行：

## 加入并流式传输

LangGraph平台允许您加入一个活跃的[后台运行](../how-tos/background_run.md)并从中流式传输输出。为此，您可以使用[LangGraph SDK的](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/python_sdk_ref/) `client.runs.join_stream`方法：

!!! 警告 "输出未缓冲"

    当您使用`.join_stream`时，输出不会缓冲，因此在加入之前产生的任何输出都不会被接收。

## API参考

有关API使用和实现，请参考[API参考](../reference/api/api_ref.html#tag/thread-runs/POST/threads/{thread_id}/runs/stream)。