# 审查工具调用

人机交互（HIL）对于[代理系统](https://langchain-ai.github.io/langgraph/concepts/agentic_concepts/#human-in-the-loop)至关重要。一个常见的模式是在某些工具调用之后添加一个人机交互步骤。这些工具调用通常会导致函数调用或保存某些信息。例如：

- 一个执行SQL的工具调用，随后将由工具运行
- 一个生成摘要的工具调用，随后将保存到图的状态中

请注意，使用工具调用是常见的，**无论是否实际调用工具**。

通常，您可能希望在此处进行以下几种不同的交互：

1. 批准工具调用并继续
2. 手动修改工具调用，然后继续
3. 提供自然语言反馈，然后将其传递回代理，而不是继续

我们可以使用[断点](https://langchain-ai.github.io/langgraph/how-tos/human_in_the_loop/breakpoints/)在LangGraph中实现这一点：断点允许我们在特定步骤之前中断图执行。在这个断点处，我们可以手动更新图状态，采取上述三种选项之一。

## 设置

我们不会展示我们托管的图的完整代码，但如果您想查看，可以在这里[查看](../../how-tos/human_in_the_loop/review-tool-calls.ipynb#simple-usage)。一旦这个图被托管，我们就可以调用它并等待用户输入。

### SDK初始化

首先，我们需要设置我们的客户端，以便与我们托管的图进行通信：


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

## 无需审查的示例

让我们看一个不需要审查的示例（因为没有调用工具）

=== "Python"

    ```python
    input = { 'messages':[{ "role":"user", "content":"hi!" }] }

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
    const input = { "messages": [{ "role": "user", "content": "hi!" }] };

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
       \"input\": {\"messages\": [{\\"role\\": \\"human\\", \\"content\\": \\"hi!\\"}]},
       \"stream_mode\": [
         \"updates\"
       ],
       \"interrupt_before\": [\"action\"]
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

    {'messages': [{'content': 'hi!', 'additional_kwargs': {}, 'response_metadata': {}, 'type': 'human', 'name': None, 'id': '39c51f14-2d5c-4690-883a-d940854b1845', 'example': False}]}
    {'messages': [{'content': 'hi!', 'additional_kwargs': {}, 'response_metadata': {}, 'type': 'human', 'name': None, 'id': '39c51f14-2d5c-4690-883a-d940854b1845', 'example': False}, {'content': [{'text': "Hello! Welcome. How can I assist you today? Is there anything specific you'd like to know or any information you're looking for?", 'type': 'text', 'index': 0}], 'additional_kwargs': {}, 'response_metadata': {'stop_reason': 'end_turn', 'stop_sequence': None}, 'type': 'ai', 'name': None, 'id': 'run-d65e07fb-43ff-4d98-ab6b-6316191b9c8b', 'example': False, 'tool_calls': [], 'invalid_tool_calls': [], 'usage_metadata': {'input_tokens': 355, 'output_tokens': 31, 'total_tokens': 386}}]}


如果我们检查状态，可以看到它已经完成

=== "Python"

    ```python
    state = await client.threads.get_state(thread["thread_id"])

    print(state['next'])
    ```

=== "Javascript"

    ```js
    const state = await client.threads.getState(thread["thread_id"]);

    console.log(state.next);
    ```

=== "CURL"

    ```bash
    curl --request GET \
        --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/state | jq -c '.next'
    ```

输出：

    []

## 批准工具的示例

现在让我们看看如何批准一个工具调用。请注意，我们不需要在我们的流调用中传递中断，因为图（定义在这里[here](../../how-tos/human_in_the_loop/review-tool-calls.ipynb#simple-usage)）已经在`human_review_node`之前编译了一个中断。

=== "Python"

    ```python
    input = {"messages": [{"role": "user", "content": "what's the weather in sf?"}]}

    async for chunk in client.runs.stream(
        thread["thread_id"],
        assistant_id,
        input=input,
    ):
        if chunk.data and chunk.event != "metadata": 
            print(chunk.data)
    ```

=== "Javascript"

    ```js
    const input = { "messages": [{ "role": "user", "content": "what's the weather in sf?" }] };

    const streamResponse = client.runs.stream(
      thread["thread_id"],
      assistantId,
      {
        input: input,
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
       \"input\": {\"messages\": [{\\"role\\": \\"human\\", \\"content\\": \\"what's the weather in sf?\\"}]}
     }" | \
     sed 's/\r$//' | \
     awk '
     /^event:/ {
         if (数据内容 != "" && 事件类型 != "metadata") {
             打印 数据内容 "\n"
         }
         替换(/^event: /, "", $0)
         事件类型 = $0
         数据内容 = ""
     }
     /^data:/ {
         替换(/^data: /, "", $0)
         数据内容 = $0
     }
     结束 {
         如果 (数据内容 != "" && 事件类型 != "metadata") {
             打印 数据内容 "\n"
         }
     }
     '
    ```

输出：

    {'messages': [{'content': "what's the weather in sf?", 'additional_kwargs': {}, 'response_metadata': {}, 'type': 'human', 'name': None, 'id': '54e19d6e-89fa-44fb-b92c-12e7dd4ddf08', 'example': False}]}
    {'messages': [{'content': "what's the weather in sf?", 'additional_kwargs': {}, 'response_metadata': {}, 'type': 'human', 'name': None, 'id': '54e19d6e-89fa-44fb-b92c-12e7dd4ddf08', 'example': False}, {'content': [{'text': "Certainly! I can help you check the weather in San Francisco. To get this information, I'll use the weather search function. Let me do that for you right away.", 'type': 'text', 'index': 0}, {'id': 'toolu_015yrR3GMDXe6X8m2p9CsEDN', 'input': {}, 'name': 'weather_search', 'type': 'tool_use', 'index': 1, 'partial_json': '{"city": "San Francisco"}'}], 'additional_kwargs': {}, 'response_metadata': {'stop_reason': 'tool_use', 'stop_sequence': None}, 'type': 'ai', 'name': None, 'id': 'run-45a6b6c3-ac69-42a4-8957-d982203d6392', 'example': False, 'tool_calls': [{'name': 'weather_search', 'args': {'city': 'San Francisco'}, 'id': 'toolu_015yrR3GMDXe6X8m2p9CsEDN', 'type': 'tool_call'}], 'invalid_tool_calls': [], 'usage_metadata': {'input_tokens': 360, 'output_tokens': 90, 'total_tokens': 450}}]}


如果我们现在检查，可以看到它正在等待人工审查：

=== "Python"

    ```python
    state = await client.threads.get_state(thread["thread_id"])

    print(state['next'])
    ```

=== "Javascript"

    ```js
    const state = await client.threads.getState(thread["thread_id"]);

    console.log(state.next);
    ```

=== "CURL"

    ```bash
    curl --request GET \
        --url <DELPOYMENT_URL>/threads/<THREAD_ID>/state | jq -c '.next'
    ```

输出：

    ['human_review_node']

要批准工具调用，我们可以继续执行线程而不进行任何编辑。为此，我们只需创建一个没有输入的新运行。

=== "Python"

    ```python
    async for chunk in client.runs.stream(
        thread["thread_id"],
        assistant_id,
        input=None,
        stream_mode="values",
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
        streamMode: "values",
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
       \"assistant_id\": \"agent\"
     }" | \
     sed 's/\r$//' | \
     awk '
     /^event:/ {
         if (数据内容 != "" && 事件类型 != "metadata") {
             打印 数据内容 "\n"
         }
         替换(/^event: /, "", $0)
         事件类型 = $0
         数据内容 = ""
     }
     /^data:/ {
         替换(/^data: /, "", $0)
         数据内容 = $0
     }
     结束 {
         如果 (数据内容 != "" && 事件类型 != "metadata") {
             打印 数据内容 "\n"
         }
     }
     '
    ```

输出：

    {'messages': [{'content': "what's the weather in sf?", 'additional_kwargs': {}, 'response_metadata': {}, 'type': 'human', 'name': None, 'id': '54e19d6e-89fa-44fb-b92c-12e7dd4ddf08', 'example': False}, {'content': [{'text': "Certainly! I can help you check the weather in San Francisco. To get this information, I'll use the weather search function. Let me do that for you right away.", 'type': 'text', 'index': 0}, {'id': 'toolu_015yrR3GMDXe6X8m2p9CsEDN', 'input': {}, 'name': 'weather_search', 'type': 'tool_use', 'index': 1, 'partial_json': '{"city": "San Francisco"}'}], 'additional_kwargs': {}, 'response_metadata': {'stop_reason': 'tool_use', 'stop_sequence': None}, 'type': 'ai', 'name': None, 'id': 'run-45a6b6c3-ac69-42a4-8957-d982203d6392', 'example': False, 'tool_calls': [{'name': 'weather_search', 'args': {'city': 'San Francisco'}, 'id': 'toolu_015yrR3GMDXe6X8m2p9CsEDN', 'type': 'tool_call'}], 'invalid_tool_calls': [], 'usage_metadata': {'input_tokens': 360, 'output_tokens': 90, 'total_tokens': 450}}, {'content': 'Sunny!', 'additional_kwargs': {}, 'response_metadata': {}, 'type': 'tool', 'name': 'weather_search', 'id': '826cd0f2-9cc6-46f0-b7df-daa6a05d13d2', 'tool_call_id': 'toolu_015yrR3GMDXe6X8m2p9CsEDN', 'artifact': None, 'status': 'success'}]}
    {'messages': [{'content': "what's the weather in sf?", 'additional_kwargs': {}, 'response_metadata': {}, 'type': 'human', 'name': None, 'id': '54e19d6e-89fa-44fb-b92c-12e7dd4ddf08', 'example': False}, {'content': [{'text': "Certainly! I can help you check the weather in San Francisco. To get this information, I'll use the weather search function. Let me do that for you right away.", 'type': 'text', 'index': 0}, {'id': 'toolu_015yrR3GMDXe6X8m2p9CsEDN', 'input': {}, 'name': 'weather_search', 'type': 'tool_use', 'index': 1, 'partial_json': '{"city": "San Francisco"}'}], 'additional_kwargs': {}, 'response_metadata': {'stop_reason': 'tool_use', 'stop_sequence': None}, 'type': 'ai', 'name': None, 'id': 'run-45a6b6c3-ac69-42a4-8957-d982203d6392', 'example': False, 'tool_calls': [{'name': 'weather_search', 'args': {'city': 'San Francisco'}, 'id': 'toolu_015yrR3GMDXe6X8m2p9CsEDN', 'type': 'tool_call'}], 'invalid_tool_calls': [], 'usage_metadata': {'input_tokens': 360, 'output_tokens': 90, 'total_tokens': 450}}, {'content': 'Sunny!', 'additional_kwargs': {}, 'response_metadata': {}, 'type': 'tool', 'name': 'weather_search', 'id': '826cd0f2-9cc6-46f0-b7df-daa6a05d13d2', 'tool_call_id': 'toolu_015yrR3GMDXe6X8m2p9CsEDN', 'artifact': None, 'status': 'success'}, {'content': [{'text': "\n\nGreat news! The weather in San Francisco is sunny today. It's a beautiful day in the city by the bay. Is there anything else you'd like to know about the weather or any other information I can help you with?", 'type': 'text', 'index': 0}], 'additional_kwargs': {}, 'response_metadata': {'stop_reason': 'end_turn', 'stop_sequence': None}, 'type': 'ai', 'name': None, 'id': 'run-5d5fd0f1-a939-447e-801a-9aaa812322d3', 'example': False, 'tool_calls': [], 'invalid_tool_calls': [], 'usage_metadata': {'input_tokens': 464, 'output_tokens': 50, 'total_tokens': 514}}]}

## 编辑工具调用

现在假设我们想要编辑工具调用。例如，更改一些参数（甚至更改调用的工具！），然后执行该工具。

=== "Python"

    ```python
    input = {"messages": [{"role": "user", "content": "what's the weather in sf?"}]}

    async for chunk in client.runs.stream(
        thread["thread_id"],
        assistant_id,
        input=input,
        stream_mode="values",
    ):
        if chunk.data and chunk.event != "metadata": 
            print(chunk.data)
    ```

=== "Javascript"

    ```js
   