# 复制线程

你可能希望复制（即“分叉”）一个现有的线程，以保留现有线程的历史记录，并创建不影响原始线程的独立运行。本指南展示了如何做到这一点。

## 设置

此代码假设你已经有一个要复制的线程。你可以在这里阅读什么是线程[这里](../../concepts/langgraph_server.md#threads)，并学习如何在线程上流式运行[这些操作指南](../../how-tos/index.md#streaming_1)。

### SDK 初始化

首先，我们需要设置客户端，以便与我们托管的图进行通信：

=== "Python"

    ```python
    from langgraph_sdk import get_client
    client = get_client(url="<DEPLOYMENT_URL>")
    assistant_id = "agent"
    thread = await client.threads.create()
    ```

=== "Javascript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";

    const client = new Client({ apiUrl: "<DEPLOYMENT_URL>" });
    const assistantId = "agent";
    const thread = await client.threads.create();
    ```

=== "CURL"

    ```bash
    curl --request POST \
      --url <DEPLOYMENT_URL>/threads \
      --header 'Content-Type: application/json' \
      --data '{
        "metadata": {}
      }'
    ```

## 复制线程

以下代码假设你要复制的线程已经存在。

复制线程将创建一个具有与现有线程相同历史记录的新线程，然后允许你继续执行运行。

### 创建副本

=== "Python"

    ```python
    copied_thread = await client.threads.copy(<THREAD_ID>)
    ```

=== "Javascript"

    ```js
    let copiedThread = await client.threads.copy(<THREAD_ID>);
    ```

=== "CURL"

    ```bash
    curl --request POST --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/copy \
    --header 'Content-Type: application/json'
    ```

### 验证副本

我们可以验证之前线程的历史记录是否正确复制：

=== "Python"

    ```python
    def remove_thread_id(d):
      if 'metadata' in d and 'thread_id' in d['metadata']:
          del d['metadata']['thread_id']
      return d

    original_thread_history = list(map(remove_thread_id,await client.threads.get_history(<THREAD_ID>)))
    copied_thread_history = list(map(remove_thread_id,await client.threads.get_history(copied_thread['thread_id'])))

    # 比较两个历史记录
    assert original_thread_history == copied_thread_history
    # 如果我们到达这里，断言通过！
    print("The histories are the same.")
    ```

=== "Javascript"

    ```js
    function removeThreadId(d) {
      if (d.metadata && d.metadata.thread_id) {
        delete d.metadata.thread_id;
      }
      return d;
    }

    // 假设 `client.threads.getHistory(threadId)` 是一个返回字典列表的异步函数
    async function compareThreadHistories(threadId, copiedThreadId) {
      const originalThreadHistory = (await client.threads.getHistory(threadId)).map(removeThreadId);
      const copiedThreadHistory = (await client.threads.getHistory(copiedThreadId)).map(removeThreadId);

      // 比较两个历史记录
      console.assert(JSON.stringify(originalThreadHistory) === JSON.stringify(copiedThreadHistory));
      // 如果我们到达这里，断言通过！
      console.log("The histories are the same.");
    }

    // 示例用法
    compareThreadHistories(<THREAD_ID>, copiedThread.thread_id);
    ```

=== "CURL"

    ```bash
    if diff <(
        curl --request GET --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/history | jq -S 'map(del(.metadata.thread_id))'
    ) <(
        curl --request GET --url <DEPLOYMENT_URL>/threads/<COPIED_THREAD_ID>/history | jq -S 'map(del(.metadata.thread_id))'
    ) >/dev/null; then
        echo "The histories are the same."
    else
        echo "The histories are different."
    fi
    ```

输出：

    The histories are the same.