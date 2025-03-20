# 如何从先前状态重放和分支

使用 LangGraph Cloud，您可以返回到任何先前的状态，并重新运行图表以重现测试期间发现的问题，或者从先前状态中以不同的方式分支。在本指南中，我们将展示一个快速示例，说明如何重放过去的状态以及如何从先前状态分支。

## 设置

我们不会展示我们托管的图表的完整代码，但如果您想查看，可以点击[这里](../../how-tos/human_in_the_loop/time-travel.ipynb#build-the-agent)。一旦这个图表被托管，我们就可以调用它并等待用户输入。

### SDK 初始化

首先，我们需要设置我们的客户端，以便与我们托管的图表进行通信：

=== "Python"

    ```python
    from langgraph_sdk import get_client
    client = get_client(url=<DEPLOYMENT_URL>)
    # 使用名为 "agent" 的图表
    assistant_id = "agent"
    thread = await client.threads.create()
    ```

=== "Javascript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";

    const client = new Client({ apiUrl: <DEPLOYMENT_URL> });
    // 使用名为 "agent" 的图表
    const assistantId = "agent";
    const thread = await client.threads.create();
    ```

=== "CURL"

    ```bash
    curl --request POST \
      --url <DEPLOYMENT_URL>/threads \
      --header 'Content-Type: application/json' \
      --data '{}'
    ```

## 重放状态

### 初始调用

在重放状态之前，我们需要创建一些状态以供重放！为此，让我们用一个简单的消息调用我们的图表：

=== "Python"

    ```python
    input = {"messages": [{"role": "user", "content": "Please search the weather in SF"}]}

    async for chunk in client.runs.stream(
        thread["thread_id"],
        assistant_id,
        input=input,
        stream_mode="updates",
    ):
        if chunk.data and chunk.event != "metadata": 
            print(chunk.data)
    ```

=== "Javascript"

    ```js
    const input = { "messages": [{ "role": "user", "content": "Please search the weather in SF" }] }

    const streamResponse = client.runs.stream(
      thread["thread_id"],
      assistantId,
      {
        input: input,
        streamMode: "updates",
      }
    );
    for await (const chunk of streamResponse) {
      if (chunk.data && chunk.event !== "metadata") {
        console.log(chunk.data);
      }
    }
    ```

=== "CURL"

    ```bash
    curl --request POST \
     --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/stream \
     --header 'Content-Type: application/json' \
     --data "{
       \"assistant_id\": \"agent\",
       \"input\": {\"messages\": [{\"role\": \"human\", \"content\": \"Please search the weather in SF\"}]},
       \"stream_mode\": [
         \"updates\"
       ]
     }" | \
     sed 's/\r$//' | \
     awk '
     /^event:/ {
         if (data_content != "" && event_type != "metadata") {
             print data_content "\n"
         }
         sub(/^event: /, "", $0)
         event_type = $0
         data_content = ""
     }
     /^data:/ {
         sub(/^data: /, "", $0)
         data_content = $0
     }
     END {
         if (data_content != "" && event_type != "metadata") {
             print data_content "\n"
         }
     }
     '
    ```
    
输出:

    {'agent': {'messages': [{'content': [{'text': "Certainly! I'll use the search function to look up the current weather in San Francisco for you. Let me do that now.", 'type': 'text'}, {'id': 'toolu_011vroKUtWU7SBdrngpgpFMn', 'input': {'query': 'current weather in San Francisco'}, 'name': 'search', 'type': 'tool_use'}], 'additional_kwargs': {}, 'response_metadata': {}, 'type': 'ai', 'name': None, 'id': 'run-ee639877-d97d-40f8-96dc-d0d1ae22d203', 'example': False, 'tool_calls': [{'name': 'search', 'args': {'query': 'current weather in San Francisco'}, 'id': 'toolu_011vroKUtWU7SBdrngpgpFMn'}], 'invalid_tool_calls': [], 'usage_metadata': None}]}}
    {'action': {'messages': [{'content': '["I looked up: current weather in San Francisco. Result: It\'s sunny in San Francisco, but you better look out if you\'re a Gemini 😈."]', 'additional_kwargs': {}, 'response_metadata': {}, 'type': 'tool', 'name': 'search', 'id': '7bad0e72-5ebe-4b08-9b8a-b99b0fe22fb7', 'tool_call_id': 'toolu_011vroKUtWU7SBdrngpgpFMn'}]}}
    {'agent': {'messages': [{'content': "Based on the search results, I can provide you with information about the current weather in San Francisco:\n\nThe weather in San Francisco is currently sunny. This is great news for outdoor activities and enjoying the city's beautiful sights.\n\nIt's worth noting that the search result included an unusual comment about Geminis, which isn't typically part of a weather report. This might be due to the search engine including some astrological information or a joke in its results. However, for the purpose of answering your question about the weather, we can focus on the fact that it's sunny in San Francisco.\n\nIf you need any more specific information about the weather in San Francisco, such as temperature, wind speed, or forecast for the coming days, please let me know, and I'd be happy to search for that information for you.", 'additional_kwargs': {}, 'response_metadata': {}, 'type': 'ai', 'name': None, 'id': 'run-dbac539a-33c8-4f0c-9e20-91f318371e7c', 'example': False, 'tool_calls': [], 'invalid_tool_calls': [], 'usage_metadata': None}]}}


现在让我们获取我们的状态列表，并从第三个状态（在工具调用之前）调用：


=== "Python"

    ```python
    states = await client.threads.get_history(thread['thread_id'])

    # 我们可以通过检查 'next' 属性来确认这个状态是正确的，并看到它是工具调用节点
    state_to_replay = states[2]
    print(state_to_replay['next'])
    ```

=== "Javascript"

    ```js
    const states = await client.threads.getHistory(thread['thread_id']);

    // 我们可以通过检查 'next' 属性来确认这个状态是正确的，并看到它是工具调用节点
    const stateToReplay = states[2];
    console.log(stateToReplay['next']);
    ```

=== "CURL"

    ```bash
    curl --request GET --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/history | jq -r '.[2].next'
    ```

输出:

    ['action']



要从一个状态重新运行，我们首先需要向线程状态发出一个空更新。然后我们需要传入生成的 `checkpoint_id`，如下所示：

=== "Python"

    ```python
    state_to_replay = states[2]
    updated_config = await client.threads.update_state(
        thread["thread_id"],
        {"messages": []},
        checkpoint_id=state_to_replay["checkpoint_id"]
    )
    async for chunk in client.runs.stream(
        thread["thread_id"],
        assistant_id, # graph_id
        input=None,
        stream_mode="updates",
        checkpoint_id=updated_config["checkpoint_id"]
    ):
        if chunk.data and chunk.event != "metadata": 
            print(chunk.data)
    ```

=== "Javascript"

    ```js
    const stateToReplay = states[2];
    const config = await client.threads.updateState(thread["thread_id"], { values: {"messages": [] }, checkpointId: stateToReplay["checkpoint_id"] });
    const streamResponse = client.runs.stream(
      thread["thread_id"],
      assistantId,
      {
        input: null,
        streamMode: "updates",
        checkpointId: config["checkpoint_id"]
      }
    );
    for await (const chunk of streamResponse) {
      if (chunk.data && chunk.event !== "metadata") {
        console.log(chunk.data);
      }
    }
    ```

=== "CURL"

    ```bash
    curl --request GET --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/history | jq -c '
        .[2] as $state_to_replay |
        {
            values: { messages: .[2].values.messages[-1] },
            checkpoint_id: $state_to_replay.checkpoint_id
        }' | \
    curl --request POST \
        --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/state \
        --header 'Content-Type: application/json' \
        --data @- | jq .checkpoint_id | \
    curl --request POST \
     --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/stream \
     --header 'Content-Type: application/json' \
     --data "{
       \"assistant_id\": \"agent\",
       \"checkpoint_id\": \"$1\",
       \"stream_mode\": [
         \"updates\"
       ]
     }" | \
     sed 's/\r$//' | \
     awk '
     /^event:/ {
         if (data_content != "" && event_type != "metadata") {
             print data_content "\n"
         }
         sub(/^event: /, "", $0)
         event_type = $0
         data_content = ""
     }
     /^data:/ {
         sub(/^data: /, "", $0)
         data_content = $0
     }
     END {
         if (data_content != "" && event_type != "metadata") {
             print data_content "\n"
         }
     }
     '
    ```

输出:

    {'action': {'messages': [{'content': '["I looked up: current weather in San Francisco. Result: It\'s sunny in San Francisco, but you better look out if you\'re a Gemini 😈."]', 'additional_kwargs': {}, 'response_metadata': {}, 'type': 'tool', 'name': 'search', 'id': 'eba650e5-400e-4938-8508-f878dcbcc532', 'tool_call_id': 'toolu_011vroKUtWU7SBdrngpgpFMn'}]}}
    {'agent': {'messages': [{'content': "Based on the search results, I can provide you with information about the current weather in San Francisco:\n\nThe weather in San Francisco is currently sunny. This is great news if you're planning any outdoor activities or simply want to enjoy a pleasant day in the city.\n\nIt's worth noting that the search result included an unusual comment about Geminis, which doesn't seem directly related to the weather. This appears to be a playful or humorous addition to the weather report, possibly from the source where this information was obtained.\n\nIs there anything else you'd like to know about the weather in San Francisco or any other information you need?", 'additional_kwargs': {}, 'response_metadata': {}, 'type': 'ai', 'name': None, 'id': 'run-bc6dca3f-a1e2-4f59-a69b-fe0515a348bb', 'example': False, 'tool_calls': [], 'invalid_tool_calls': [], 'usage_metadata': None}]}}


正如我们所看到的，图表从工具节点重新启动，输入与我们原始图表运行相同。

## 从先前状态分支

使用 LangGraph 的检查点功能，您不仅可以重放过去的状态，还可以从先前的位置分支，让代理探索替代轨迹，或者让用户“版本控制”工作流中的更改。

让我们展示如何做到这一点，以编辑特定时间点的状态。让我们更新状态以更改工具的输入

=== "Python"

    ```python
    # 现在让我们获取状态中的最后一条消息
    # 这是我们要更新的工具调用
    last_message = state_to_replay['values']['messages'][-1]

    # 现在让我们更新该工具调用的参数
    last_message['tool_calls'][0]['args'] = {'query': 'current weather in SF'}

    config = await client.threads.update_state(thread['thread_id'],{"messages":[last_message]},checkpoint_id=state_to_replay['checkpoint_id'])
    ```

=== "Javascript"

    ```js
    // 现在让我们获取状态中的最后一条消息
    // 这是我们要更新的工具调用
    let lastMessage = stateToReplay['values']['messages'][-1];

    // 现在让我们更新该工具调用的参数
    lastMessage['tool_calls'][0]['args'] = { 'query': 'current weather in SF' };

    const config = await client.threads.updateState(thread['thread_id'], { values: { "messages": [lastMessage] }, checkpointId: stateToReplay['checkpoint_id'] });
    ```

=== "CURL"

    ```bash
    curl -s --request GET --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/history | \
    jq -c '
        .[2] as $state_to_replay |
        .[2].values.messages[-1].tool_calls[0].args.query = "current weather in SF" |
        {
            values: { messages: .[2].values.messages[-1] },
            checkpoint_id: $state_to_replay.checkpoint_id
        }' | \
    curl --request POST \
        --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/state \
        --header 'Content-Type: application/json' \
        --data @-
    ```

现在我们可以使用这个新配置重新运行我们的图表，从 `new_state` 开始，这是我们 `state_to_replay` 的一个分支：

=== "Python"

    ```python
    async for chunk in client.runs.stream(
        thread["thread_id"],
        assistant_id,
        input=None,
        stream_mode="updates",
        checkpoint_id=config['checkpoint_id']
    ):
        if chunk.data and chunk.event != "metadata": 
            print(chunk.data)
    ```

=== "Javascript"

    ```js
    const streamResponse = client.runs.stream(
      thread["thread_id"],
      assistantId,
      {
        input: null,
        streamMode: "updates",
        checkpointId: config['checkpoint_id'],
      }
    );
    for await (const chunk of streamResponse) {
      if (chunk.data && chunk.event !== "metadata") {
        console.log(chunk.data);
      }
    }
    ```

=== "CURL"

    ```bash
    curl -s --request GET --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/state | \
    jq -c '.checkpoint_id' | \
    curl --request POST \
     --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/stream \
     --header 'Content-Type: application/json' \
     --data "{
       \"assistant_id\": \"agent\",
       \"checkpoint_id\": \"$1\",
       \"stream_mode\": [
         \"updates\"
       ]
     }" | \
     sed 's/\r$//' | \
     awk '
     /^event:/ {
         if (data_content != "" && event_type != "metadata") {
             print data_content "\n"
         }
         sub(/^event: /, "", $0)
         event_type = $0
         data_content = ""
     }
     /^data:/ {
         sub(/^data: /, "", $0)
         data_content = $0
     }
     END {
         if (data_content != "" && event_type != "metadata") {
             print data_content "\n"
         }
     }
     '
    ```

输出:


    {'action': {'messages': [{'content': '["I looked up: current weather in SF. Result: It\'s sunny in San Francisco, but you better look out if you\'re a Gemini 😈."]', 'additional_kwargs': {}, 'response_metadata': {}, 'type': 'tool', 'name': 'search', 'id': '2baf9941-4fda-4081-9f87-d76795d289f1', 'tool_call_id': 'toolu_011vroKUtWU7SBdrngpgpFMn'}]}}
    {'agent': {'messages': [{'content': "Based on the search results, I can provide you with information about the current weather in San Francisco (SF):\n\nThe weather in San Francisco is currently sunny. This means it's a clear day with plenty of sunshine. \n\nIt's worth noting that the specific temperature wasn't provided in the search result, but sunny weather in San Francisco typically means comfortable temperatures. San Francisco is known for its mild climate, so even on sunny days, it's often not too hot.\n\nThe search result also included a playful reference to astrological signs, mentioning Gemini. However, this is likely just a joke or part of the search engine's presentation and not related to the actual weather conditions.\n\nIs there any specific information about the weather in San Francisco you'd like to know more about? I'd be happy to perform another search if you need details on temperature, wind conditions, or the forecast for the coming days.", 'additional_kwargs': {}, 'response_metadata': {}, 'type': 'ai', 'name': None, 'id': 'run-a83de52d-ed18-4402-9384-75c462485743', 'example': False, 'tool_calls': [], 'invalid_tool_calls': [], 'usage_metadata': None}]}}


正如我们所看到的，搜索查询从 San Francisco 更改为 SF，正如我们所希望的那样！