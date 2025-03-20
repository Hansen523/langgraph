# 如何创建具有配置的代理

LangGraph API 的一个好处是它允许您创建具有不同配置的代理。
这在您希望以下情况时非常有用：

- 将认知架构定义为 LangGraph
- 让该 LangGraph 在某些属性上可配置（例如，系统消息或使用的 LLM）
- 让用户创建具有任意配置的代理，保存它们，并在将来使用它们

在本指南中，我们将展示如何为我们内置的默认代理实现这一点。

如果您查看我们定义的代理，您可以看到在 `call_model` 节点中，我们基于某些配置创建了模型。该节点如下所示：

=== "Python"

    ```python
    def call_model(state, config):
        messages = state["messages"]
        model_name = config.get('configurable', {}).get("model_name", "anthropic")
        model = _get_model(model_name)
        response = model.invoke(messages)
        # 我们返回一个列表，因为这将被添加到现有列表中
        return {"messages": [response]}
    ```

=== "Javascript"

    ```js
    function callModel(state: State, config: RunnableConfig) {
      const messages = state.messages;
      const modelName = config.configurable?.model_name ?? "anthropic";
      const model = _getModel(modelName);
      const response = model.invoke(messages);
      // 我们返回一个列表，因为这将被添加到现有列表中
      return { messages: [response] };
    }
    ```

我们正在配置中查找 `model_name` 参数（如果未找到，则默认为 `anthropic`）。这意味着默认情况下我们使用 Anthropic 作为我们的模型提供商。在本示例中，我们将看到一个如何创建配置为使用 OpenAI 的代理的示例。

首先，让我们设置我们的客户端和线程：

=== "Python"

    ```python
    from langgraph_sdk import get_client

    client = get_client(url=<DEPLOYMENT_URL>)
    # 选择一个未配置的助手
    assistants = await client.assistants.search()
    assistant = [a for a in assistants if not a["config"]][0]
    ```

=== "Javascript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";

    const client = new Client({ apiUrl: <DEPLOYMENT_URL> });
    // 选择一个未配置的助手
    const assistants = await client.assistants.search();
    const assistant = assistants.find(a => !a.config);
    ```

=== "CURL"

    ```bash
    curl --request POST \
        --url <DEPLOYMENT_URL>/assistants/search \
        --header 'Content-Type: application/json' \
        --data '{
            "limit": 10,
            "offset": 0
        }' | jq -c 'map(select(.config == null or .config == {})) | .[0]'
    ```

我们现在可以调用 `.get_schemas` 来获取与此图相关的模式：

=== "Python"

    ```python
    schemas = await client.assistants.get_schemas(
        assistant_id=assistant["assistant_id"]
    )
    # 有多种类型的模式
    # 我们可以获取 `config_schema` 来查看可配置参数
    print(schemas["config_schema"])
    ```

=== "Javascript"

    ```js
    const schemas = await client.assistants.getSchemas(
      assistant["assistant_id"]
    );
    // 有多种类型的模式
    // 我们可以获取 `config_schema` 来查看可配置参数
    console.log(schemas.config_schema);
    ```

=== "CURL"

    ```bash
    curl --request GET \
        --url <DEPLOYMENT_URL>/assistants/<ASSISTANT_ID>/schemas | jq -r '.config_schema'
    ```

输出：

    {
        'model_name': 
            {
                'title': 'Model Name',
                'enum': ['anthropic', 'openai'],
                'type': 'string'
            }
    }

现在我们可以使用配置初始化一个助手：

=== "Python"

    ```python
    openai_assistant = await client.assistants.create(
        # "agent" 是我们部署的图的名称
        "agent", config={"configurable": {"model_name": "openai"}}
    )

    print(openai_assistant)
    ```

=== "Javascript"

    ```js
    let openAIAssistant = await client.assistants.create(
      // "agent" 是我们部署的图的名称
      "agent", { "configurable": { "model_name": "openai" } }
    );

    console.log(openAIAssistant);
    ```

=== "CURL"

    ```bash
    curl --request POST \
        --url <DEPLOYMENT_URL>/assistants \
        --header 'Content-Type: application/json' \
        --data '{"graph_id":"agent","config":{"configurable":{"model_name":"open_ai"}}}'
    ```

输出：

    {
        "assistant_id": "62e209ca-9154-432a-b9e9-2d75c7a9219b",
        "graph_id": "agent",
        "created_at": "2024-08-31T03:09:10.230718+00:00",
        "updated_at": "2024-08-31T03:09:10.230718+00:00",
        "config": {
            "configurable": {
                "model_name": "open_ai"
            }
        },
        "metadata": {}
    }

我们可以验证配置确实生效：

=== "Python"

    ```python
    thread = await client.threads.create()
    input = {"messages": [{"role": "user", "content": "who made you?"}]}
    async for event in client.runs.stream(
        thread["thread_id"],
        openai_assistant["assistant_id"],
        input=input,
        stream_mode="updates",
    ):
        print(f"Receiving event of type: {event.event}")
        print(event.data)
        print("\n\n")
    ```

=== "Javascript"

    ```js
    const thread = await client.threads.create();
    let input = { "messages": [{ "role": "user", "content": "who made you?" }] };

    const streamResponse = client.runs.stream(
      thread["thread_id"],
      openAIAssistant["assistant_id"],
      {
        input,
        streamMode: "updates"
      }
    );

    for await (const event of streamResponse) {
      console.log(`Receiving event of type: ${event.event}`);
      console.log(event.data);
      console.log("\n\n");
    }
    ```

=== "CURL"

    ```bash
    thread_id=$(curl --request POST \
        --url <DEPLOYMENT_URL>/threads \
        --header 'Content-Type: application/json' \
        --data '{}' | jq -r '.thread_id') && \
    curl --request POST \
        --url "<DEPLOYMENT_URL>/threads/${thread_id}/runs/stream" \
        --header 'Content-Type: application/json' \
        --data '{
            "assistant_id": <OPENAI_ASSISTANT_ID>,
            "input": {
                "messages": [
                    {
                        "role": "user",
                        "content": "who made you?"
                    }
                ]
            },
            "stream_mode": [
                "updates"
            ]
        }' | \
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
                print data_content "\n\n"
            }
        }
    '
    ```

输出：

    Receiving event of type: metadata
    {'run_id': '1ef6746e-5893-67b1-978a-0f1cd4060e16'}



    Receiving event of type: updates
    {'agent': {'messages': [{'content': 'I was created by OpenAI, a research organization focused on developing and advancing artificial intelligence technology.', 'additional_kwargs': {}, 'response_metadata': {'finish_reason': 'stop', 'model_name': 'gpt-4o-2024-05-13', 'system_fingerprint': 'fp_157b3831f5'}, 'type': 'ai', 'name': None, 'id': 'run-e1a6b25c-8416-41f2-9981-f9cfe043f414', 'example': False, 'tool_calls': [], 'invalid_tool_calls': [], 'usage_metadata': None}]}}

