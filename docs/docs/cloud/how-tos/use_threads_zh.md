# 使用线程

在本指南中，我们将展示如何创建、查看和检查[线程](../../concepts/persistence.md#threads)。

## 创建线程

要运行您的图并保持状态持久化，首先需要创建一个线程。

### 空线程

使用[LangGraph SDK](../../concepts/sdk.md)的`create`方法创建新线程。更多信息参见[Python](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/python_sdk_ref/#langgraph_sdk.client.ThreadsClient.create)和[JS](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/js_ts_sdk_ref/#create_3) SDK参考文档。

=== "Python"

    ```python
    from langgraph_sdk import get_client

    client = get_client(url=<DEPLOYMENT_URL>)
    thread = await client.threads.create()

    print(thread)
    ```

=== "Javascript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";

    const client = new Client({ apiUrl: <DEPLOYMENT_URL> });
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
      "thread_id": "123e4567-e89b-12d3-a456-426614174000",
      "created_at": "2025-05-12T14:04:08.268Z",
      "updated_at": "2025-05-12T14:04:08.268Z",
      "metadata": {},
      "status": "idle",
      "values": {}
    }

### 复制线程

如果您已有一个线程并希望复制其状态，可以使用`copy`方法。这将创建一个独立线程，其历史记录与操作时的原始线程相同。更多信息参见[Python](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/python_sdk_ref/#langgraph_sdk.client.ThreadsClient.copy)和[JS](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/js_ts_sdk_ref/#copy) SDK参考文档。

=== "Python"

    ```python
    copied_thread = await client.threads.copy(<THREAD_ID>)
    ```

=== "Javascript"

    ```js
    const copiedThread = await client.threads.copy(<THREAD_ID>);
    ```

=== "CURL"

    ```bash
    curl --request POST --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/copy \
    --header 'Content-Type: application/json'
    ```

### 预填充状态

您还可以通过向`create`方法提供`supersteps`列表来创建具有任意预定义状态的线程。`supersteps`描述了一系列状态更新。

=== "Python"

    ```python
    from langgraph_sdk import get_client

    client = get_client(url=<DEPLOYMENT_URL>)
    thread = await client.threads.create(
      graph_id="agent",
      supersteps=[
        {
          updates: [
            {
              values: {},
              as_node: '__input__',
            },
          ],
        },
        {
          updates: [
            {
              values: {
                messages: [
                  {
                    type: 'human',
                    content: 'hello',
                  },
                ],
              },
              as_node: '__start__',
            },
          ],
        },
        {
          updates: [
            {
              values: {
                messages: [
                  {
                    content: 'Hello! How can I assist you today?',
                    type: 'ai',
                  },
                ],
              },
              as_node: 'call_model',
            },
          ],
        },
      ])

    print(thread)
    ```

=== "Javascript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";

    const client = new Client({ apiUrl: <DEPLOYMENT_URL> });
    const thread = await client.threads.create({
        graphId: 'agent',
        supersteps: [
        {
          updates: [
            {
              values: {},
              asNode: '__input__',
            },
          ],
        },
        {
          updates: [
            {
              values: {
                messages: [
                  {
                    type: 'human',
                    content: 'hello',
                  },
                ],
              },
              asNode: '__start__',
            },
          ],
        },
        {
          updates: [
            {
              values: {
                messages: [
                  {
                    content: 'Hello! How can I assist you today?',
                    type: 'ai',
                  },
                ],
              },
              asNode: 'call_model',
            },
          ],
        },
      ],
    });

    console.log(thread);
    ```

=== "CURL"

    ```bash
    curl --request POST \
        --url <DEPLOYMENT_URL>/threads \
        --header 'Content-Type: application/json' \
        --data '{"metadata":{"graph_id":"agent"},"supersteps":[{"updates":[{"values":{},"as_node":"__input__"}]},{"updates":[{"values":{"messages":[{"type":"human","content":"hello"}]},"as_node":"__start__"}]},{"updates":[{"values":{"messages":[{"content":"Hello\u0021 How can I assist you today?","type":"ai"}]},"as_node":"call_model"}]}]}'
    ```

输出:

    {
        "thread_id": "f15d70a1-27d4-4793-a897-de5609920b7d",
        "created_at": "2025-05-12T15:37:08.935038+00:00",
        "updated_at": "2025-05-12T15:37:08.935046+00:00",
        "metadata": {"graph_id": "agent"},
        "status": "idle",
        "config": {},
        "values": {
            "messages": [
                {
                    "content": "hello",
                    "additional_kwargs": {},
                    "response_metadata": {},
                    "type": "human",
                    "name": null,
                    "id": "8701f3be-959c-4b7c-852f-c2160699b4ab",
                    "example": false
                },
                {
                    "content": "Hello! How can I assist you today?",
                    "additional_kwargs": {},
                    "response_metadata": {},
                    "type": "ai",
                    "name": null,
                    "id": "4d8ea561-7ca1-409a-99f7-6b67af3e1aa3",
                    "example": false,
                    "tool_calls": [],
                    "invalid_tool_calls": [],
                    "usage_metadata": null
                }
            ]
        }
    }

## 列出线程

### LangGraph SDK

使用[LangGraph SDK](../../concepts/sdk.md)的`search`方法列出线程。这将列出符合筛选条件的应用程序线程。更多信息参见[Python](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/python_sdk_ref/#langgraph_sdk.client.ThreadsClient.search)和[JS](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/js_ts_sdk_ref/#search_2) SDK参考文档。

#### 按线程状态筛选

使用`status`字段按状态筛选线程。支持的值包括`idle`、`busy`、`interrupted`和`error`。

=== "Python"

    ```python
    print(await client.threads.search(status="idle",limit=1))
    ```

=== "Javascript"

    ```js
    console.log(await client.threads.search({ status: "idle", limit: 1 }));
    ```

=== "CURL"

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads/search \
    --header 'Content-Type: application/json' \
    --data '{"status": "idle", "limit": 1}'
    ```

输出:

    [
      {
        'thread_id': 'cacf79bb-4248-4d01-aabc-938dbd60ed2c',
        'created_at': '2024-08-14T17:36:38.921660+00:00',
        'updated_at': '2024-08-14T17:36:38.921660+00:00',
        'metadata': {'graph_id': 'agent'},
        'status': 'idle',
        'config': {'configurable': {}}
      }
    ]

#### 按元数据筛选

=== "Python"

    ```python
    print((await client.threads.search(metadata={"graph_id":"agent"},limit=1)))
    ```

=== "Javascript"

    ```js
    console.log((await client.threads.search({ metadata: { "graph_id": "agent" }, limit: 1 })));
    ```

=== "CURL"

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads/search \
    --header 'Content-Type: application/json' \
    --data '{"metadata": {"graph_id":"agent"}, "limit": 1}'
    ```

输出:

    [
      {
        'thread_id': 'cacf79bb-4248-4d01-aabc-938dbd60ed2c',
        'created_at': '2024-08-14T17:36:38.921660+00:00',
        'updated_at': '2024-08-14T17:36:38.921660+00:00',
        'metadata': {'graph_id': 'agent'},
        'status': 'idle',
        'config': {'configurable': {}}
      }
    ]

#### 排序

SDK还支持通过`thread_id`、`status`、`created_at`和`updated_at`进行排序，使用`sort_by`和`sort_order`参数。

### LangGraph平台UI

您也可以通过LangGraph平台UI查看部署中的线程。

在您的部署中选择"Threads"选项卡，这将加载部署中的所有线程表格。

要按线程状态筛选，请选择顶部栏中的状态。要按支持的属性排序，请单击所需列的箭头图标。

## 检查线程

### LangGraph SDK

#### 获取线程

使用`get`方法查看特定线程：

=== "Python"

    ```python
    print((await client.threads.get(<THREAD_ID>)))
    ```

=== "Javascript"

    ```js
    console.log((await client.threads.get(<THREAD_ID>)));
    ```

=== "CURL"

    ```bash
    curl --request GET \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID> \
    --header 'Content-Type: application/json'
    ```

输出:

    {
      'thread_id': 'cacf79bb-4248-4d01-aabc-938dbd60ed2c',
      'created_at': '2024-08-14T17:36:38.921660+00:00',
      'updated_at': '2024-08-14T17:36:38.921660+00:00',
      'metadata': {'graph_id': 'agent'},
      'status': 'idle',
      'config': {'configurable': {}}
    }

#### 检查线程状态

使用`get_state`方法查看线程的当前状态：

=== "Python"

    ```python
    print((await client.threads.get_state(<THREAD_ID>)))
    ```

=== "Javascript"

    ```js
    console.log((await client.threads.getState(<THREAD_ID>)));
    ```

=== "CURL"

    ```bash
    curl --request GET \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/state \
    --header 'Content-Type: application/json'
    ```

输出:

    {
        "values": {
            "messages": [
                {
                    "content": "hello",
                    "additional_kwargs": {},
                    "response_metadata": {},
                    "type": "human",
                    "name": null,
                    "id": "8701f3be-959c-4b7c-852f-c2160699b4ab",
                    "example": false
                },
                {
                    "content": "Hello! How can I assist you today?",
                    "additional_kwargs": {},
                    "response_metadata": {},
                    "type": "ai",
                    "name": null,
                    "id": "4d8ea561-7ca1-409a-99f7-6b67af3e1aa3",
                    "example": false,
                    "tool_calls": [],
                    "invalid_tool_calls": [],
                    "usage_metadata": null
                }
            ]
        },
        "next": [],
        "tasks": [],
        "metadata": {
            "thread_id": "f15d70a1-27d4-4793-a897-de5609920b7d",
            "checkpoint_id": "1f02f46f-7308-616c-8000-1b158a9a6955",
            "graph_id": "agent_with_quite_a_long_name",
            "source": "update",
            "step": 1,
            "writes": {
                "call_model": {
                    "messages": [
                        {
                            "content": "Hello! How can I assist you today?",
                            "type": "ai"
                        }
                    ]
                }
            },
            "parents": {}
        },
        "created_at": "2025-05-12T15:37:09.008055+00:00",
        "checkpoint": {
            "checkpoint_id": "1f02f46f-733f-6b58-8001-ea90dcabb1bd",
            "thread_id": "f15d70a1-27d4-4793-a897-de5609920b7d",
            "checkpoint_ns": ""
        },
        "parent_checkpoint": {
            "checkpoint_id": "1f02f46f-7308-616c-8000-1b158a9a6955",
            "thread_id": "f15d70a1-27d4-4793-a897-de5609920b7d",
            "checkpoint_ns": ""
        },
        "checkpoint_id": "1f02f46f-733f-6b58-8001-ea90dcabb1bd",
        "parent_checkpoint_id": "1f02f46f-7308-616c-8000-1b158a9a6955"
    }

可选地，要查看线程在特定检查点的状态，只需传入检查点ID（或整个检查点对象）：

=== "Python"

    ```python
    thread_state = await client.threads.get_state(
      thread_id=<THREAD_ID>
      checkpoint_id=<CHECKPOINT_ID>
    )
    ```

=== "Javascript"

    ```js
    const threadState = await client.threads.getState(<THREAD_ID>, <CHECKPOINT_ID>);
    ```

=== "CURL"

    ```bash
    curl --request GET \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/state/<CHECKPOINT_ID> \
    --header 'Content-Type: application/json'
    ```

#### 检查完整线程历史

使用`get_history`方法查看线程的历史记录。这将返回线程经历的所有状态的列表。

### LangGraph平台UI

您也可以通过LangGraph平台UI查看线程。

在您的部署中选择"Threads"选项卡，这将加载部署中的所有线程表格。

选择线程以检查其当前状态。要在[LangGraph Studio](../../concepts//langgraph_studio.md)中查看其完整历史记录并进行调试，请打开线程。