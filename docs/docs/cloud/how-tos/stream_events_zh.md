# 如何流式传输事件

!!! info "前提条件"
    * [流式传输](../../concepts/streaming.md#流式传输-llm-令牌和事件-astream_events)

本指南介绍了如何从你的图中流式传输事件 (`stream_mode="events"`)。根据你的 LangGraph 应用程序的用例和用户体验，你的应用程序可能会以不同的方式处理事件类型。

## 设置

=== "Python"

    ```python
    from langgraph_sdk import get_client

    client = get_client(url=<DEPLOYMENT_URL>)
    # 使用名称为 "agent" 部署的图
    assistant_id = "agent"
    # 创建线程
    thread = await client.threads.create()
    print(thread)
    ```

=== "Javascript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";

    const client = new Client({ apiUrl: <DEPLOYMENT_URL> });
    // 使用名称为 "agent" 部署的图
    const assistantID = "agent";
    // 创建线程
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
        'thread_id': '3f4c64e0-f792-4a5e-aa07-a4404e06e0bd',
        'created_at': '2024-06-24T22:16:29.301522+00:00',
        'updated_at': '2024-06-24T22:16