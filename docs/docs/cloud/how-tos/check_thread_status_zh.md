# 检查你的线程状态

## 设置

首先，我们可以使用你托管图形的URL来设置我们的客户端：

### SDK初始化

首先，我们需要设置我们的客户端，以便与我们托管的图形进行通信：

=== "Python"

    ```python
    from langgraph_sdk import get_client
    client = get_client(url=<DEPLOYMENT_URL>)
    # 使用名称为"agent"的图形
    assistant_id = "agent"
    thread = await client.threads.create()
    ```

=== "Javascript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";

    const client = new Client({ apiUrl: <DEPLOYMENT_URL> });
    // 使用名称为"agent"的图形
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

## 查找空闲线程

我们可以使用以下命令查找空闲的线程，这意味着在该线程上执行的所有运行都已结束：

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

输出：

    [{'thread_id': 'cacf79bb-4248-4d01-aabc-938dbd60ed2c',
    'created_at': '2024-08-14T17:36:38.921660+00:00',
    'updated_at': '2024-08-14T17:36:38.921660+00:00',
    'metadata': {'graph_id': 'agent'},
    'status': 'idle',
    'config': {'configurable': {}}}]


## 查找中断的线程

我们可以使用以下命令查找在运行过程中被中断的线程，这可能意味着在运行结束之前发生了错误，或者达到了人工干预的断点，运行正在等待继续：

=== "Python"

    ```python
    print(await client.threads.search(status="interrupted",limit=1))
    ```

=== "Javascript"

    ```js
    console.log(await client.threads.search({ status: "interrupted", limit: 1 }));
    ```

=== "CURL"

    ```bash
    curl --request POST \  
    --url <DEPLOYMENT_URL>/threads/search \
    --header 'Content-Type: application/json' \
    --data '{"status": "interrupted", "limit": 1}'
    ```

输出：

    [{'thread_id': '0d282b22-bbd5-4d95-9c61-04dcc2e302a5',
    'created_at': '2024-08-14T17:41:50.235455+00:00',
    'updated_at': '2024-08-14T17:41:50.235455+00:00',
    'metadata': {'graph_id': 'agent'},
    'status': 'interrupted',
    'config': {'configurable': {}}}]
    
## 查找忙碌的线程

我们可以使用以下命令查找忙碌的线程，这意味着它们当前正在处理运行的执行：

=== "Python"

    ```python
    print(await client.threads.search(status="busy",limit=1))
    ```

=== "Javascript"

    ```js
    console.log(await client.threads.search({ status: "busy", limit: 1 }));
    ```

=== "CURL"

    ```bash
    curl --request POST \  
    --url <DEPLOYMENT_URL>/threads/search \
    --header 'Content-Type: application/json' \
    --data '{"status": "busy", "limit": 1}'
    ```

输出：

    [{'thread_id': '0d282b22-bbd5-4d95-9c61-04dcc2e302a5',
    'created_at': '2024-08-14T17:41:50.235455+00:00',
    'updated_at': '2024-08-14T17:41:50.235455+00:00',
    'metadata': {'graph_id': 'agent'},
    'status': 'busy',
    'config': {'configurable': {}}}]

## 查找特定线程

你可能还想检查特定线程的状态，你可以通过以下几种方式来实现：

### 通过ID查找

你可以使用`get`函数来查找特定线程的状态，只要你保存了ID：

=== "Python"

    ```python
    print((await client.threads.get(<THREAD_ID>))['status'])
    ```

=== "Javascript"

    ```js
    console.log((await client.threads.get(<THREAD_ID>)).status);
    ```

=== "CURL"

    ```bash
    curl --request GET \ 
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID> \
    --header 'Content-Type: application/json' | jq -r '.status'
    ```

输出：

    'idle'

### 通过元数据查找

线程的搜索端点还允许你根据元数据进行过滤，如果你使用元数据来标记线程以保持其组织性，这将非常有用：

=== "Python"

    ```python
    print((await client.threads.search(metadata={"foo":"bar"},limit=1))[0]['status'])
    ```

=== "Javascript"

    ```js
    console.log((await client.threads.search({ metadata: { "foo": "bar" }, limit: 1 }))[0].status);
    ```

=== "CURL"

    ```bash
    curl --request POST \  
    --url <DEPLOYMENT_URL>/threads/search \
    --header 'Content-Type: application/json' \
    --data '{"metadata": {"foo":"bar"}, "limit": 1}' | jq -r '.[0].status'
    ```

输出：

    'idle'