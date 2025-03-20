# 如何同时配置多种流模式

!!! info "前提条件"
    * [流模式](../../concepts/streaming.md)

本指南介绍如何同时配置多种流模式。

## 设置

首先，让我们设置客户端和线程：

=== "Python"

    ```python
    from langgraph_sdk import get_client

    client = get_client(url=<DEPLOYMENT_URL>)
    # 使用名为 "agent" 的已部署图
    assistant_id = "agent"
    # 创建线程
    thread = await client.threads.create()
    print(thread)
    ```

=== "Javascript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";

    const client = new Client({ apiUrl: <DEPLOYMENT_URL> });
    // 使用名为 "agent" 的已部署图
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

输出：

    {
        'thread_id': 'bfc68029-1f7b-400f-beab-6f9032a52da4',
        'created_at': '2024-06-24T21:30:07.980789+00:00',
        'updated_at': '2024-06-24T21:30:07.980789+00:00',
        'metadata': {},
        'status': 'idle',
        'config': {},
        'values': None
    }

## 使用多种流模式运行图

在为运行配置多种流模式时，每种模式都会生成相应的响应。在以下示例中，请注意将模式列表（`messages`、`events`、`debug`）传递给 `stream_mode` 参数，并且响应包含 `events`、`debug`、`messages/complete`、`messages/metadata` 和 `messages/partial` 事件类型。

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

    # 使用多种流模式流式传输事件
    async for chunk in client.runs.stream(
        thread_id=thread["thread_id"],
        assistant_id=assistant_id,
        input=input,
        stream_mode=["messages", "events", "debug"],
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

    // 使用多种流模式流式传输事件
    const streamResponse = client.runs.stream(
      thread["thread_id"],
      assistantID,
      {
        input,
        streamMode: ["messages", "events", "debug"]
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
         \"messages\",
         \"events\",
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

输出：

    Receiving new event of type: metadata...
    {'run_id': '1ef32717-bc30-6cf2-8a26-33f63567bc25'}
    
    
    
    Receiving new event of type: events...
    {'event': 'on_chain_start', 'data': {'input': {'messages': [{'role': 'human', 'content': "What's the weather in SF?"}]}}, 'name': 'LangGraph', 'tags': [], 'run_id': '1ef32717-bc30-6cf2-8a26-33f63567bc25', 'metadata': {'created_by': 'system', 'run_id': '1ef32717-bc30-6cf2-8a26-33f63567bc25', 'user_id': '', 'graph_id': 'agent', 'thread_id': 'bfc68029-1f7b-400f-beab-6f9032a52da4', 'assistant_id': 'fe096781-5601-53d2-b2f6-0d3403f7e9ca'}, 'parent_ids': []}
    
    
    
    Receiving new event of type: debug...
    {'type': 'checkpoint', 'timestamp': '2024-06-24T21:34:06.116009+00:00', 'step': -1, 'payload': {'config': {'tags': [], 'metadata': {'created_by': 'system', 'run_id': '1ef32717-bc30-6cf2-8a26-33f63567bc25', 'user_id': '', 'graph_id': 'agent', 'thread_id': 'bfc68029-1f7b-400f-beab-6f9032a52da4', 'assistant_id': 'fe096781-5601-53d2-b2f6-0d3403f7e9ca'}, 'callbacks': [None], 'recursion_limit': 25, 'configurable': {'run_id': '1ef32717-bc30-6cf2-8a26-33f63567bc25', 'user_id': '', 'graph_id': 'agent', 'thread_id': 'bfc68029-1f7b-400f-beab-6f9032a52da4', 'thread_ts': '1ef32717-bc7c-6daa-bfff-6b9027c1a50e', 'assistant_id': 'fe096781-5601-53d2-b2f6-0d3403f7e9ca'}, 'run_id': '1ef32717-bc30-6cf2-8a26-33f63567bc25'}, 'values': {'messages': []}, 'metadata': {'source': 'input', 'step': -1, 'writes': {'messages': [{'role': 'human', 'content': "What's the weather in SF?"}]}}}}
    
    
    
    Receiving new event of type: events...
    {'event': 'on_chain_stream', 'run_id': '1ef32717-bc30-6cf2-8a26-33f63567bc25', 'name': 'LangGraph', 'tags': [], 'metadata': {'created_by