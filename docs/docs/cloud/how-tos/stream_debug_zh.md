# 如何流式传输调试事件

!!! info "前提条件"
    * [流式传输](../../concepts/streaming.md)

本指南介绍了如何从您的图中流式传输调试事件（`stream_mode="debug"`）。流式传输调试事件会产生包含 `type` 和 `timestamp` 键的响应。调试事件对应于图执行中的不同步骤，有三种不同类型的步骤会流式传输回给您：

- `checkpoint`：这些事件会在图保存其状态时流式传输，这发生在每个超级步骤之后。了解更多关于检查点的信息[这里](https://langchain-ai.github.io/langgraph/concepts/low_level/#checkpointer)
- `task`：这些事件会在每个超级步骤之前流式传输，并包含有关单个任务的信息。每个超级步骤通过执行一系列任务来工作，其中每个任务都限定在特定的节点和输入范围内。下面我们将详细讨论这些任务的格式。
- `task_result`：在每个 `task` 事件之后，您会看到一个相应的 `task_result` 事件，顾名思义，它包含有关在超级步骤中执行的任务结果的信息。滚动以了解这些事件的确切结构。

## 设置

首先让我们设置我们的客户端和线程：

=== "Python"
    
    ```python
    from langgraph_sdk import get_client

    client = get_client(url=<DEPLOYMENT_URL>)
    # 使用名称为 "agent" 的图
    assistant_id = "agent"
    # 创建线程
    thread = await client.threads.create()
    print(thread)
    ```

=== "Javascript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";

    const client = new Client({ apiUrl: <DEPLOYMENT_URL> });
    // 使用名称为 "agent" 的图
    const assistantID = "agent";
    // 创建线程
    const thread = await client.threads.create();
    console.log(thread);
    ```

=== "CURL"

    ```bash
    curl --request POST \
      --url <DEPLOYMENT_URL>/threads \
      --header 'Content-Type: application/json' \
      --data '{}'
    ```


输出:

    {
        'thread_id': 'd0cbe9ad-f11c-443a-9f6f-dca0ae5a0dd3',
        'created_at': '2024-06-21T22:10:27.696862+00:00',
        'updated_at': '2024-06-21T22:10:27.696862+00:00',
        'metadata': {},
        'status': 'idle',
        'config': {},
        'values': None
    }

## 以调试模式流式传输图

=== "Python"

    ```python
    # 创建输入
    input = {
        "messages": [
            {
                "role": "user",
                "content": "What's the weather in SF?",
            }
        ]
    }

    # 流式调试
    async for chunk in client.runs.stream(
        thread_id=thread["thread_id"],
        assistant_id=assistant_id,
        input=input,
        stream_mode="debug",
    ):
        print(f"Receiving new event of type: {chunk.event}...")
        print(chunk.data)
        print("\n\n")
    ```

=== "Javascript"

    ```js
    // 创建输入
    const input = {
      messages: [
        {
          role: "human",
          content: "What's the weather in SF?",
        }
      ]
    };

    // 流式调试
    const streamResponse = client.runs.stream(
      thread["thread_id"],
      assistantID,
      {
        input,
        streamMode: "debug"
      }
    );

    for await (const chunk of streamResponse) {
      console.log(`Receiving new event of type: ${chunk.event}...`);
      console.log(chunk.data);
      console.log("\n\n");
    }
    ```

=== "CURL"

    ```bash
    curl --request POST \
     --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/stream \
     --header 'Content-Type: application/json' \
     --data "{
       \"assistant_id\": \"agent\",
       \"input\": {\"messages\": [{\"role\": \"human\", \"content\": \"What's the weather in SF?\"}]},
       \"stream_mode\": [
         \"debug\"
       ]
     }" | \
     sed 's/\r$//' | \
     awk '
     /^event:/ {
         if (data_content != "") {
             print data_content "\n"
         }
         sub(/^event: /, "Receiving event of type: ", $0)
         printf "%s...\n", $0
         data_content = ""
     }
     /^data:/ {
         sub(/^data: /, "", $0)
         data_content = $0
     }
     END {
         if (data_content != "") {
             print data_content "\n"
         }
     }
     ' 
    ```


输出:

    Receiving new event of type: metadata...
    {'run_id': '1ef65938-d7c7-68db-b786-011aa1cb3cd2'}



    Receiving new event of type: debug...
    {'type': 'checkpoint', 'timestamp': '2024-08-28T23:16:28.134680+00:00', 'step': -1, 'payload': {'config': {'tags': [], 'metadata': {'created_by': 'system', 'run_id': '1ef65938-d7c7-68db-b786-011aa1cb3cd2', 'user_id': '', 'graph_id': 'agent', 'thread_id': 'be4fd54d-ff22-4e9e-8876-d5cccc0e8048', 'assistant_id': 'fe096781-5601-53d2-b2f6-0d3403f7e9ca'}, 'callbacks': [None], 'recursion_limit': 25, 'configurable': {'run_id': '1ef65938-d7c7-68db-b786-011aa1cb3cd2', 'user_id': '', 'graph_id': 'agent', 'thread_id': 'be4fd54d-ff22-4e9e-8876-d5cccc0e8048', 'assistant_id': 'fe096781-5601-53d2-b2f6-0d3403f7e9ca', 'checkpoint_id': '1ef65938-d8f3-6b25-bfff-30a8ed6460bd', 'checkpoint_ns': ''}, 'run_id': '1ef65938-d7c7-68db-b786-011aa1cb3cd2'}, 'values': {'messages': [], 'search_results': []}, 'metadata': {'source': 'input', 'writes': {'messages': [{'role': 'human', 'content': "What's the weather in SF?"}]}, 'step': -1}, 'next': ['__start__'], 'tasks': [{'id': 'b40d2c90-dc1e-52db-82d6-08751b769c55', 'name': '__start__', 'interrupts': []}]}}



    Receiving new event of type: debug...
    {'type': 'checkpoint', 'timestamp': '2024-08-28T23:16:28.139821+00:00', 'step': 0, 'payload': {'config': {'tags': [], 'metadata': {'created_by