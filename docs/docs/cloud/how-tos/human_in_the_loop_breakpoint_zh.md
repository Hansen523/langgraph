# 如何添加断点

在创建 LangGraph 代理时，通常需要添加一个人工干预组件。这在为它们提供工具访问权限时非常有用。在这些情况下，您可能希望在执行之前手动批准某个操作。

这可以通过几种方式实现，但主要支持的方式是在执行节点之前添加“中断”。这会在该节点处中断执行。然后，您可以从该点恢复以继续执行。

## 设置

### 为您的图编写代码

在本指南中，我们使用一个简单的 ReAct 风格的托管图（您可以在[这里](../../how-tos/human_in_the_loop/breakpoints.ipynb)查看定义它的完整代码）。重要的是，有两个节点（一个名为 `agent`，用于调用 LLM，另一个名为 `action`，用于调用工具），以及从 `agent` 节点确定是调用 `action` 节点还是结束图运行的路由函数（`action` 节点在执行后总是调用 `agent` 节点）。

### SDK 初始化

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

## 添加断点

我们现在希望在调用工具之前在图中添加一个断点。我们可以通过添加 `interrupt_before=["action"]` 来实现这一点，这告诉我们在调用 `action` 节点之前中断。我们可以在编译图时或在启动运行时执行此操作。这里我们将在启动运行时执行此操作，如果您希望在编译时执行此操作，则需要编辑定义图的 Python 文件，并在调用 `.compile` 时添加 `interrupt_before` 参数。

首先，让我们通过 SDK 访问我们托管的 LangGraph 实例：

然后，现在让我们在工具节点之前编译它并添加断点：

=== "Python"

    ```python
    input = {"messages": [{"role": "user", "content": "what's the weather in sf"}]}
    async for chunk in client.runs.stream(
        thread["thread_id"],
        assistant_id,
        input=input,
        stream_mode="updates",
        interrupt_before=["action"],
    ):
        print(f"Receiving new event of type: {chunk.event}...")
        print(chunk.data)
        print("\n\n")
    ```
=== "Javascript"

    ```js
    const input = { messages: [{ role: "human", content: "what's the weather in sf" }] };

    const streamResponse = client.runs.stream(
      thread["thread_id"],
      assistantId,
      {
        input: input,
        streamMode: "updates",
        interruptBefore: ["action"]
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
       \"input\": {\"messages\": [{\"role\": \"human\", \"content\": \"what's the weather in sf\"}]},
       \"interrupt_before\": [\"action\"],
       \"stream_mode\": [
         \"messages\"
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
    {'run_id': '3b77ef83-687a-4840-8858-0371f91a92c3'}
    
    
    
    Receiving new event of type: data...
    {'agent': {'messages': [{'content': [{'id': 'toolu_01HwZqM1ptX6E15A5LAmyZTB', 'input': {'query': 'weather in san francisco'}, 'name': 'tavily_search_results_json', 'type': 'tool_use'}], 'additional_kwargs': {}, 'response_metadata': {}, 'type': 'ai', 'name': None, 'id': 'run-e5d17791-4d37-4ad2-815f-a0c4cba62585', 'example': False, 'tool_calls': [{'name': 'tavily_search_results_json', 'args': {'query': 'weather in san francisco'}, 'id': 'toolu_01HwZqM1ptX6E15A5LAmyZTB'}], 'invalid_tool_calls': []}]}}
    
    
    
    Receiving new event of type: end...
    None
    
    
    