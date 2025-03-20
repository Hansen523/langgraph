# 如何流式传输图的状态更新

!!! info "前提条件"
    * [流式传输](../../concepts/streaming.md)

本指南介绍如何为图使用 `stream_mode="updates"`，这将流式传输每个节点执行后对图状态的更新。这与使用 `stream_mode="values"` 不同：它不会在每个超步流式传输状态的整个值，而是仅流式传输在该超步中对状态进行更新的每个节点的更新。

## 设置

首先，让我们设置客户端和线程：

=== "Python"

    ```python
    from langgraph_sdk import get_client

    client = get_client(url=<DEPLOYMENT_URL>)
    # 使用名为 "agent" 的图
    assistant_id = "agent"
    # 创建线程
    thread = await client.threads.create()
    print(thread)
    ```

=== "Javascript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";

    const client = new Client({ apiUrl: <DEPLOYMENT_URL> });
    // 使用名为 "agent" 的图
    const assistantID = "agent";
    # 创建线程
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
      'thread_id': '979e3c89-a702-4882-87c2-7a59a250ce16',
      'created_at': '2024-06-21T15:22:07.453100+00:00',
      'updated_at': '2024-06-21T15:22:07.453100+00:00',
      'metadata': {},
      'status': 'idle',
      'config': {},
      'values': None 
    }

## 以更新模式流式传输图

现在我们可以通过更新进行流式传输，它会在每个节点执行后输出对状态的更新：


=== "Python"

    ```python
    input = {
        "messages": [
            {
                "role": "user",
                "content": "what's the weather in la"
            }
        ]
    }
    async for chunk in client.runs.stream(
        thread["thread_id"],
        assistant_id,
        input=input,
        stream_mode="updates",
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
          content: "What's the weather in la"
        }
      ]
    };

    const streamResponse = client.runs.stream(
      thread["thread_id"],
      assistantID,
      {
        input,
        streamMode: "updates"
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
       \"input\": {\"messages\": [{\"role\": \"human\", \"content\": \"What's the weather in la\"}]},
       \"stream_mode\": [
         \"updates\"
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
    {"run_id": "cfc96c16-ed9a-44bd-b5bb-c30e3c0725f0"}



    Receiving new event of type: updates...
    {
      "agent": {
        "messages": [
          {
            "type": "ai",
            "tool_calls": [
              {
                "name": "tavily_search_results_json",
                "args": {
                  "query": "weather in los angeles"
                },
                "id": "toolu_0148tMmDK51iLQfG1yaNwRHM"
              }
            ],
            ...
          }
        ]
      }
    }



    Receiving new event of type: updates...
    {
      "action": {
        "messages": [
          {
            "content": [
              {
                "url": "https://www.weatherapi.com/",
                "content": "{\"location\": {\"name\": \"Los Angeles\", \"region\": \"California\", \"country\": \"United States of America\", \"lat\": 34.05, \"lon\": -118.24, \"tz_id\": \"America/Los_Angeles\", \"localtime_epoch\": 1716062239, \"localtime\": \"2024-05-18 12:57\"}, \"current\": {\"last_updated_epoch\": 1716061500, \"last_updated\": \"2024-05-18 12:45\", \"temp_c\": 18.9, \"temp_f\": 66.0, \"is_day\": 1, \"condition\": {\"text\": \"Overcast\", \"icon\": \"//cdn.weatherapi.com/weather/64x64/day/122.png\", \"code\": 1009}, \"wind_mph\": 2.2, \"wind_kph\": 3.6, \"wind_degree\": 10, \"wind_dir\": \"N\", \"pressure_mb\": 1017.0, \"pressure_in\": 30.02, \"precip_mm\": 0.0, \"precip_in\": 0.0, \"humidity\": 65, \"cloud\": 100, \"feelslike_c\": 18.9, \"feelslike_f\": 66.0, \"vis_km\": 16.0, \"vis_miles\": 9.0, \"uv\": 6.0, \"gust_mph\": 7.5, \"gust_kph\": 12.0}}"
              }
            ],
            "type": "tool",
            "name": "tavily_search_results_json",
            "tool_call_id": "toolu_0148tMmDK51iLQfG1yaNwRHM",
            ...
          }
        ]
      }
    }



    Receiving new event of type: updates...
    {
      "agent": {
        "messages": [
          {
            "content": "The weather in Los Angeles is currently overcast with a temperature of around 66°F (18.9°C). There are light winds from the north at around 2-3 mph. The humidity is 65% and visibility is good at 9 miles. Overall, mild spring weather conditions in LA.",
            "type": "ai",
            ...
          }
        ]
      }
    }



    Receiving new event of type: end...
    None