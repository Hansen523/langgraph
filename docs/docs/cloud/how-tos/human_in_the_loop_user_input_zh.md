# 如何等待用户输入

人机交互的主要模式之一是等待用户输入。一个关键用例是向用户提出澄清问题。实现这一点的一种方法是直接转到`END`节点并退出图。然后，任何用户响应都会作为图的新调用返回。这基本上就是创建一个聊天机器人架构。

这样做的问题是很难回到图中的特定点继续执行。通常情况下，代理正在执行某个过程的中间阶段，只需要一点用户输入。虽然可以设计图，使其具有`conditional_entry_point`，将用户消息路由回正确的位置，但这并不具有很好的扩展性（因为这本质上涉及一个可以几乎在任何地方结束的路由函数）。

另一种方法是设置一个专门用于获取用户输入的节点。这在笔记本环境中很容易实现——只需在节点中放置一个`input()`调用。但这并不完全适合生产环境。

幸运的是，LangGraph使得可以在生产环境中实现类似的功能。基本思路是：

- 设置一个表示用户输入的节点。这可以有特定的入边/出边（根据需要）。该节点中实际上不应包含任何逻辑。
- 在该节点之前添加一个断点。这将在该节点执行之前停止图（这很好，因为其中没有任何实际逻辑）。
- 使用`.update_state`来更新图的状态。传入你获得的任何用户响应。这里的关键是使用`as_node`参数，**将该更新应用为该节点的操作**。这将使得下次恢复执行时，从该节点刚刚执行后的状态继续，而不是从头开始。

## 设置

我们不会展示我们托管的图的完整代码，但你可以在这里查看[here](../../how-tos/human_in_the_loop/wait-user-input.ipynb#agent)。一旦这个图被托管，我们就可以调用它并等待用户输入。

### SDK初始化

首先，我们需要设置客户端，以便与托管的图进行通信：

=== "Python"

    ```python
    from langgraph_sdk import get_client
    client = get_client(url=<DEPLOYMENT_URL>)
    # 使用名为 "agent" 的图
    assistant_id = "agent"
    thread = await client.threads.create()
    ```

=== "Javascript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";

    const client = new Client({ apiUrl: <DEPLOYMENT_URL> });
    // 使用名为 "agent" 的图
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

## 等待用户输入

### 初始调用

现在，让我们通过中断`ask_human`节点来调用我们的图：

=== "Python"

    ```python
    input = {
        "messages": [
            {
                "role": "user",
                "content": "使用搜索工具询问用户他们所在的位置，然后查找那里的天气",
            }
        ]
    }

    async for chunk in client.runs.stream(
        thread["thread_id"],
        assistant_id,
        input=input,
        stream_mode="updates",
        interrupt_before=["ask_human"],
    ):
        if chunk.data and chunk.event != "metadata": 
            print(chunk.data)
    ```
=== "Javascript"

    ```js
    const input = {
      messages: [
        {
          role: "human",
          content: "使用搜索工具询问用户他们所在的位置，然后查找那里的天气"
        }
      ]
    };

    const streamResponse = client.runs.stream(
      thread["thread_id"],
      assistantId,
      {
        input: input,
        streamMode: "updates",
        interruptBefore: ["ask_human"]
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
       \"input\": {\"messages\": [{\"role\": \"human\", \"content\": \"使用搜索工具询问用户他们所在的位置，然后查找那里的天气\"}]},
       \"interrupt_before\": [\"ask_human\"],
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

    {'agent': {'messages': [{'content': [{'text': "当然！我将使用AskHuman函数询问用户的位置，然后使用搜索函数查找该位置的天气。让我们从询问用户的位置开始。", 'type': 'text'}, {'id': 'toolu_01RFahzYPvnPWTb2USk2RdKR', 'input': {'question': '您当前所在的位置是哪里？'}, 'name': 'AskHuman', 'type': 'tool_use'}], 'additional_kwargs': {}, 'response_metadata': {}, 'type': 'ai', 'name': None, 'id': 'run-a8422215-71d3-4093-afb4-9db141c94ddb', 'example': False, 'tool_calls': [{'name': 'AskHuman', 'args': {'question': '您当前所在的位置是哪里？'}, 'id': 'toolu_01RFahzYPvnPWTb2USk2RdKR'}], 'invalid_tool_calls': [], 'usage_metadata': None}]}}


### 将用户输入添加到状态

我们现在希望用用户的响应更新这个线程。然后我们可以启动另一个运行。

因为我们将此视为工具调用，所以我们需要将状态更新为工具调用的响应。为此，我们需要检查状态以获取工具调用的ID。


=== "Python"

    ```python
    state = await client.threads.get_state(thread['thread_id'])
    tool_call_id = state['values']['messages'][-1]['tool_calls'][0]['id']

    # 我们现在创建带有ID和我们想要的响应的工具调用
    tool_message = [{"tool_call_id": tool_call_id, "type": "tool", "content": "旧金山"}]

    await client.threads.update_state(thread['thread_id'], {"messages": tool_message}, as_node="ask_human")
    ```

=== "Javascript"

    ```js
    const state = await client.threads.getState(thread["thread_id"]);
    const toolCallId = state.values.messages[state.values.messages.length - 1].tool_calls[0].id;

    // 我们现在创建带有ID和我们想要的响应的工具调用
    const toolMessage = [
      {
        tool_call_id: toolCallId,
        type: "tool",
        content: "旧金山"
      }
    ];

    await client.threads.updateState(
      thread["thread_id"],
      { values: { messages: toolMessage } },
      { asNode: "ask_human" }
    );
    ```

=== "CURL"

    ```bash
    curl --request GET \
     --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/state \
     | jq -r '.values.messages[-1].tool_calls[0].id' \
     | sh -c '
         TOOL_CALL_ID="$1"
         
         # 构造JSON有效载荷
         JSON_PAYLOAD=$(printf "{\"messages\": [{\"tool_call_id\": \"%s\", \"type\": \"tool\", \"content\": \"旧金山\"}], \"as_node\": \"ask_human\"}" "$TOOL_CALL_ID")
         
         # 发送更新后的状态
         curl --request POST \
              --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/state \
              --header "Content-Type: application/json" \
              --data "${JSON_PAYLOAD}"
     ' _ 
    ```

输出:

    {'configurable': {'thread_id': 'a9f322ae-4ed1-41ec-942b-38cb3d342c3a',
    'checkpoint_ns': '',
    'checkpoint_id': '1ef58e97-a623-63dd-8002-39a9a9b20be3'}}


### 在接收到用户输入后调用

我们现在可以告诉代理继续。我们可以只传递None作为图的输入，因为不需要额外的输入：

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
        streamMode: "updates"
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

输出:

    {'agent': {'messages': [{'content': [{'text': "感谢您告诉我您在旧金山。现在，我将使用搜索功能查找旧金山的天气。", 'type': 'text'}, {'id': 'toolu_01K57ofmgG2wyJ8tYJjbq5k7', 'input': {'query': '旧金山当前天气'}, 'name': 'search', 'type': 'tool_use'}], 'additional_kwargs': {}, 'response_metadata': {}, 'type': 'ai', 'name': None, 'id': 'run-241baed7-db5e-44ce-ac3c-56431705c22b', 'example': False, 'tool_calls': [{'name': 'search', 'args': {'query': '旧金山当前天气'}, 'id': 'toolu_01K57ofmgG2wyJ8tYJjbq5k7'}], 'invalid_tool_calls': [], 'usage_metadata': None}]}}
    {'action': {'messages': [{'content': '["我查找了：旧金山当前天气。结果：旧金山阳光明媚，但如果你是双子座，你最好小心😈。"]', 'additional_kwargs': {}, 'response_metadata': {}, 'type': 'tool', 'name': 'search', 'id': '8b699b95-8546-4557-8e66-14ea71a15ed8', 'tool_call_id': 'toolu_01K57ofmgG2wyJ8tYJjbq5k7'}]}}
    {'agent': {'messages': [{'content': "根据搜索结果，我可以为您提供旧金山当前天气的信息：\n\n旧金山的天气目前是晴天。这是一个美好的日子！\n\n然而，我注意到搜索结果中包括了一个关于双子座星座的奇怪评论。这似乎是一个笑话或搜索引擎添加的无关信息。要获取准确和详细的天气信息，您可能需要查看旧金山的可靠天气服务或应用程序。\n\n您还想了解关于天气或旧金山的其他信息吗？", 'additional_kwargs': {}, 'response_metadata': {}, 'type': 'ai', 'name': None, 'id': 'run-b4d7309f-f849-46aa-b6ef-475bcabd2be9', 'example': False, 'tool_calls': [], 'invalid_tool_calls': [], 'usage_metadata': None}]}}