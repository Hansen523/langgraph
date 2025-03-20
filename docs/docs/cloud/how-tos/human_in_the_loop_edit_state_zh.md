# 如何编辑已部署图的状态

在创建LangGraph代理时，通常需要添加一个人工干预组件。这在赋予它们访问工具时非常有用。在这些情况下，您可能希望在继续之前编辑图的状态（例如，编辑正在调用的工具或调用方式）。

这可以通过几种方式实现，但主要支持的方式是在节点执行之前添加“中断”。这会在该节点处中断执行。然后，您可以使用`update_state`更新状态，并从该点继续执行。

## 设置

我们不会展示我们托管的图的完整代码，但如果您想看，可以在这里查看[here](../../how-tos/human_in_the_loop/edit-graph-state.ipynb#agent)。一旦托管了这个图，我们就可以调用它并等待用户输入。

### SDK初始化

首先，我们需要设置客户端，以便与我们托管的图进行通信：


=== "Python"

    ```python
    from langgraph_sdk import get_client
    client = get_client(url=<DEPLOYMENT_URL>)
    # 使用名为"agent"的图
    assistant_id = "agent"
    thread = await client.threads.create()
    ```

=== "Javascript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";

    const client = new Client({ apiUrl: <DEPLOYMENT_URL> });
    // 使用名为"agent"的图
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

## 编辑状态

### 初始调用

现在让我们调用我们的图，确保在`action`节点之前中断。

=== "Python"

    ```python
    input = { 'messages':[{ "role":"user", "content":"search for weather in SF" }] }

    async for chunk in client.runs.stream(
        thread["thread_id"],
        assistant_id,
        input=input,
        stream_mode="updates",
        interrupt_before=["action"],
    ):
        if chunk.data and chunk.event != "metadata": 
            print(chunk.data)
    ```

=== "Javascript"

    ```js
    const input = { messages: [{ role: "human", content: "search for weather in SF" }] };

    const streamResponse = client.runs.stream(
      thread["thread_id"],
      assistantId,
      {
        input: input,
        streamMode: "updates",
        interruptBefore: ["action"],
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
       \"input\": {\"messages\": [{\"role\": \"human\", \"content\": \"search for weather in SF\"}]},
       \"interrupt_before\": [\"action\"],
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

输出：

    {'agent': {'messages': [{'content': [{'text': "Certainly! I'll search for the current weather in San Francisco for you using the search function. Here's how I'll do that:", 'type': 'text'}, {'id': 'toolu_01KEJMBFozSiZoS4mAcPZeqQ', 'input': {'query': 'current weather in San Francisco'}, 'name': 'search', 'type': 'tool_use'}], 'additional_kwargs': {}, 'response_metadata': {}, 'type': 'ai', 'name': None, 'id': 'run-6dbb0167-f8f6-4e2a-ab68-229b2d1fbb64', 'example': False, 'tool_calls': [{'name': 'search', 'args': {'query': 'current weather in San Francisco'}, 'id': 'toolu_01KEJMBFozSiZoS4mAcPZeqQ'}], 'invalid_tool_calls': [], 'usage_metadata': None}]}}


### 编辑状态

现在，假设我们实际上是想搜索Sidi Frej（另一个以SF为缩写的城市）的天气。我们可以编辑状态以正确反映这一点：


=== "Python"

    ```python
    # 首先，获取当前状态
    current_state = await client.threads.get_state(thread['thread_id'])

    # 现在获取状态中的最后一条消息
    # 这是我们要更新的带有工具调用的消息
    last_message = current_state['values']['messages'][-1]

    # 现在更新该工具调用的参数
    last_message['tool_calls'][0]['args'] = {'query': 'current weather in Sidi Frej'}

    # 现在调用`update_state`，将这条消息传递到`messages`键中
    # 这将像任何其他状态更新一样被处理
    # 它将被传递到`messages`键的reducer函数中
    # 该reducer函数将使用消息的ID来更新它
    # 重要的是它必须具有正确的ID！否则它将被附加为新消息
    await client.threads.update_state(thread['thread_id'], {"messages": last_message})
    ```

=== "Javascript"

    ```js
    // 首先，获取当前状态
    const currentState = await client.threads.getState(thread["thread_id"]);

    // 现在获取状态中的最后一条消息
    // 这是我们要更新的带有工具调用的消息
    let lastMessage = currentState.values.messages.slice(-1)[0];

    // 现在更新该工具调用的参数
    lastMessage.tool_calls[0].args = { query: "current weather in Sidi Frej" };

    // 现在调用`update_state`，将这条消息传递到`messages`键中
    // 这将像任何其他状态更新一样被处理
    // 它将被传递到`messages`键的reducer函数中
    // 该reducer函数将使用消息的ID来更新它
    // 重要的是它必须具有正确的ID！否则它将被附加为新消息
    await client.threads.updateState(thread["thread_id"], { values: { messages: lastMessage } });
    ```

=== "CURL"

    ```bash
    curl --request GET --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/state | \                                                                                      
    jq '.values.messages[-1] | (.tool_calls[0].args = {"query": "current weather in Sidi Frej"})' | \
    curl --request POST \
      --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/state \
      --header 'Content-Type: application/json' \
      --data @-
    ```

输出：

    {'configurable': {'thread_id': '9c8f1a43-9dd8-4017-9271-2c53e57cf66a',
      'checkpoint_ns': '',
      'checkpoint_id': '1ef58e7e-3641-649f-8002-8b4305a64858'}}



### 恢复调用

现在我们可以恢复图运行，但使用更新后的状态：


=== "Python"

    ```python
    async for chunk in client.runs.stream(
        thread["thread_id"],
        assistant_id,
        input=None,
        stream_mode="updates",
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
       \"stream_mode\": [
         \"updates\"
       ]
     }"| \ 
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

输出：

    {'action': {'messages': [{'content': '["I looked up: current weather in Sidi Frej. Result: It\'s sunny in San Francisco, but you better look out if you\'re a Gemini 😈."]', 'additional_kwargs': {}, 'response_metadata': {}, 'type': 'tool', 'name': 'search', 'id': '1161b8d1-bee4-4188-9be8-698aecb69f10', 'tool_call_id': 'toolu_01KEJMBFozSiZoS4mAcPZeqQ'}]}}
    {'agent': {'messages': [{'content': [{'text': 'I apologize for the confusion in my search query. It seems the search function interpreted "SF" as "Sidi Frej" instead of "San Francisco" as we intended. Let me search again with the full city name to get the correct information:', 'type': 'text'}, {'id': 'toolu_0111rrwgfAcmurHZn55qjqTR', 'input': {'query': 'current weather in San Francisco'}, 'name': 'search', 'type': 'tool_use'}], 'additional_kwargs': {}, 'response_metadata': {}, 'type': 'ai', 'name': None, 'id': 'run-b8c25779-cfb4-46fc-a421-48553551242f', 'example': False, 'tool_calls': [{'name': 'search', 'args': {'query': 'current weather in San Francisco'}, 'id': 'toolu_0111rrwgfAcmurHZn55qjqTR'}], 'invalid_tool_calls': [], 'usage_metadata': None}]}}
    {'action': {'messages': [{'content': '["I looked up: current weather in San Francisco. Result: It\'s sunny in San Francisco, but you better look out if you\'re a Gemini 😈."]', 'additional_kwargs': {}, 'response_metadata': {}, 'type': 'tool', 'name': 'search', 'id': '6bc632ae-5ee6-4d01-9532-79c524a2d443', 'tool_call_id': 'toolu_0111rrwgfAcmurHZn55qjqTR'}]}}
    {'agent': {'messages': [{'content': "Now, based on the search results, I can provide you with information about the current weather in San Francisco:\n\nThe weather in San Francisco is currently sunny. \n\nIt's worth noting that the search result included an unusual comment about Gemini, which doesn't seem directly related to the weather. This might be due to the search engine including some astrological information or a joke in its results. However, for the purpose of weather information, we can focus on the fact that it's sunny in San Francisco right now.\n\nIs there anything else you'd like to know about the weather in San Francisco or any other location?", 'additional_kwargs': {}, 'response_metadata': {}, 'type': 'ai', 'name': None, 'id': 'run-227a042b-dd97-476e-af32-76a3703af5d8', 'example': False, 'tool_calls': [], 'invalid_tool_calls': [], 'usage_metadata': None}]}}


如您所见，它现在查找Sidi Frej的当前天气（尽管我们的虚拟搜索节点仍然返回SF的结果，因为我们在这个示例中实际上并没有进行搜索，我们每次都返回相同的“It's sunny in San Francisco ...”结果）。