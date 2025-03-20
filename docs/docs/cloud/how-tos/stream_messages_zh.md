# 如何从您的图中流式传输消息

!!! info "先决条件"
    * [流式传输](../../concepts/streaming.md)

本指南将介绍如何从您的图中流式传输消息。使用 `stream_mode="messages-tuple"`，图中节点内任何聊天模型调用生成的消息（即单个LLM令牌）将被流式传输回来。

## 设置

首先，让我们设置客户端和线程：

=== "Python"

    ```python
    from langgraph_sdk import get_client

    client = get_client(url=<DEPLOYMENT_URL>)
    # 使用部署名称为 "agent" 的图
    assistant_id = "agent"
    # 创建线程
    thread = await client.threads.create()
    print(thread)
    ```

=== "Javascript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";

    const client = new Client({ apiUrl: <DEPLOYMENT_URL> });
    // 使用部署名称为 "agent" 的图
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
        'thread_id': 'e1431c95-e241-4d1d-a252-27eceb1e5c86',
        'created_at': '2024-06-21T15:48:59.808924+00:00',
        'updated_at': '2024-06-21T15:48:59.808924+00:00',
        'metadata': {},
        'status': 'idle',
        'config': {},
        'values': None
    }

## 在消息模式下流式传输图

现在，我们可以以元组 `(message, metadata)` 的形式流式传输节点内生成的任何消息的LLM令牌。元数据包含附加信息，可用于将流式输出过滤到特定节点或LLM。

=== "Python"

    ```python
    input = {"messages": [{"role": "user", "content": "what's the weather in sf"}]}
    config = {"configurable": {"model_name": "openai"}}

    async for chunk in client.runs.stream(
        thread["thread_id"],
        assistant_id=assistant_id,
        input=input,
        config=config,
        stream_mode="messages-tuple",
    ):
        print(f"Receiving new event of type: {chunk.event}...")
        print(chunk.data)
        print("\n\n")
    ```

=== "Javascript"

    ```js
    const input = {
      messages: [
        {
          role: "human",
          content: "What's the weather in sf",
        }
      ]
    };
    const config = { configurable: { model_name: "openai" } };

    const streamResponse = client.runs.stream(
      thread["thread_id"],
      assistantID,
      {
        input,
        config,
        streamMode: "messages-tuple"
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
       \"input\": {\"messages\": [{\"role\": \"human\", \"content\": \"what's the weather in la\"}]},
       \"stream_mode\": [
         \"messages-tuple\"
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

    接收到新事件类型：metadata...
    {"run_id": "1ef971e0-9a84-6154-9047-247b4ce89c4d", "attempt": 1}

    ...

    接收到新事件类型：messages...
    [
      {
        "type": "AIMessageChunk",
        "tool_calls": [
          {
            "name": "tavily_search_results_json",
            "args": {
              "query": "weat"
            },
            "id": "toolu_0114XKXdNtHQEa3ozmY1uDdM",
            "type": "tool_call"
          }
        ],
        ...
      },
      {
        "graph_id": "agent",
        "langgraph_node": "agent",
        ...
      }
    ]



    接收到新事件类型：messages...
    [
      {
        "type": "AIMessageChunk",
        "tool_calls": [
          {
            "name": "tavily_search_results_json",
            "args": {
              "query": "her in san "
            },
            "id": "toolu_0114XKXdNtHQEa3ozmY1uDdM",
            "type": "tool_call"
          }
        ],
        ...
      },
      {
        "graph_id": "agent",
        "langgraph_node": "agent",
        ...
      }
    ]

    ...

    接收到新事件类型：messages...
    [
      {
        "type": "AIMessageChunk",
        "tool_calls": [
          {
            "name": "tavily_search_results_json",
            "args": {
              "query": "francisco"
            },
            "id": "toolu_0114XKXdNtHQEa3ozmY1uDdM",
            "type": "tool_call"
          }
        ],
        ...
      },
      {
        "graph_id": "agent",
        "langgraph_node": "agent",
        ...
      }
    ]

    ...

    接收到新事件类型：messages...
    [
      {
        "content": "[{\"url\": \"https://www.weatherapi.com/\", \"content\": \"{'location': {'name': 'San Francisco', 'region': 'California', 'country': 'United States of America', 'lat': 37.775, 'lon': -122.4183, 'tz_id': 'America/Los_Angeles', 'localtime_epoch': 1730475777, 'localtime': '2024-11-01 08:42'}, 'current': {'last_updated_epoch': 1730475000, 'last_updated': '2024-11-01 08:30', 'temp_c': 11.1, 'temp_f': 52.0, 'is_day': 1, 'condition': {'text': 'Partly cloudy', 'icon': '//cdn.weatherapi.com/weather/64x64/day/116.png', 'code': 1003}, 'wind_mph': 2.2, 'wind_kph': 3.6, 'wind_degree': 192, 'wind_dir': 'SSW', 'pressure_mb': 1018.0, 'pressure_in': 30.07, 'precip_mm': 0.0, 'precip_in': 0.0, 'humidity': 89, 'cloud': 75, 'feelslike_c': 11.5, 'feelslike_f': 52.6, 'windchill_c': 10.0, 'windchill_f': 50.1, 'heatindex_c': 10.4, 'heatindex_f': 50.7, 'dewpoint_c': 9.1, 'dewpoint_f': 48.5, 'vis_km': 16.0, 'vis_miles': 9.0, 'uv': 3.0, 'gust_mph': 6.7, 'gust_kph': 10.8}}\"}]",
        "type": "tool",
        "tool_call_id": "toolu_0114XKXdNtHQEa3ozmY1uDdM",
        ...
      },
      {
        "graph_id": "agent",
        "langgraph_node": "action",
        ...
      }
    ]

    ...

    接收到新事件类型：messages...
    [
      {
        "content": [
          {
            "text": "\n\nThe search",
            "type": "text",
            "index": 0
          }
        ],
        "type": "AIMessageChunk",
        ...
      },
      {
        "graph_id": "agent",
        "langgraph_node": "agent",
        ...
      }
    ]



    接收到新事件类型：messages...
    [
      {
        "content": [
          {
            "text": " results provide",
            "type": "text",
            "index": 0
          }
        ],
        "type": "AIMessageChunk",
        ...
      },
      {
        "graph_id": "agent",
        "langgraph_node": "agent",
        ...
      }
    ]



    接收到新事件类型：messages...
    [
      {
        "content": [
          {
            "text": " the current weather conditions",
            "type": "text",
            "index": 0
          }
        ],
        "type": "AIMessageChunk",
        ...
      },
      {
        "graph_id": "agent",
        "langgraph_node": "agent",
        ...
      }
    ]



    接收到新事件类型：messages...
    [
      {
        "content": [
          {
            "text": " in San Francisco.",
            "type": "text",
            "index": 0
          }
        ],
        "type": "AIMessageChunk",
        ...
      },
      {
        "graph_id": "agent",
        "langgraph_node": "agent",
        ...
      }
    ]

    ...