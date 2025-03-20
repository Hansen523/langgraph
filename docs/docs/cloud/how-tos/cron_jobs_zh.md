# 定时任务

有时你不想基于用户交互来运行你的图，而是希望按计划调度你的图运行——例如，如果你希望你的图每周为你的团队编写并发送待办事项邮件。LangGraph Cloud 允许你通过使用 `Crons` 客户端来实现这一点，而无需编写自己的脚本。要调度一个图任务，你需要传递一个 [cron 表达式](https://crontab.cronhub.io/) 来告知客户端何时运行图。`Cron` 任务在后台运行，不会干扰图的正常调用。

## 设置

首先，让我们设置我们的 SDK 客户端、助手和线程：

=== "Python"

    ```python
    from langgraph_sdk import get_client

    client = get_client(url=<DEPLOYMENT_URL>)
    # 使用部署名称为 "agent" 的图
    assistant_id = "agent"
    # 创建线程
    thread = await client.threads.create()
    print(thread)
    ```

=== "Javascript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";

    const client = new Client({ apiUrl: <DEPLOYMENT_URL> });
    // 使用部署名称为 "agent" 的图
    const assistantId = "agent";
    // 创建线程
    const thread = await client.threads.create();
    console.log(thread);
    ```

=== "CURL"

    ```bash
    curl --request POST \
        --url <DEPLOYMENT_URL>/assistants/search \
        --header 'Content-Type: application/json' \
        --data '{
            "limit": 10,
            "offset": 0
        }' | jq -c 'map(select(.config == null or .config == {})) | .[0].graph_id' && \
    curl --request POST \
        --url <DEPLOYMENT_URL>/threads \
        --header 'Content-Type: application/json' \
        --data '{}'
    ```

输出：

    {
        'thread_id': '9dde5490-2b67-47c8-aa14-4bfec88af217', 
        'created_at': '2024-08-30T23:07:38.242730+00:00', 
        'updated_at': '2024-08-30T23:07:38.242730+00:00', 
        'metadata': {}, 
        'status': 'idle', 
        'config': {}, 
        'values': None
    }

## 线程上的定时任务

要创建与特定线程关联的定时任务，你可以编写：


=== "Python"

    ```python
    # 这调度了一个每天 15:27（下午 3:27）运行的任务
    cron_job = await client.crons.create_for_thread(
        thread["thread_id"],
        assistant_id,
        schedule="27 15 * * *",
        input={"messages": [{"role": "user", "content": "What time is it?"}]},
    )
    ```

=== "Javascript"

    ```js
    // 这调度了一个每天 15:27（下午 3:27）运行的任务
    const cronJob = await client.crons.create_for_thread(
      thread["thread_id"],
      assistantId,
      {
        schedule: "27 15 * * *",
        input: { messages: [{ role: "user", content: "What time is it?" }] }
      }
    );
    ```

=== "CURL"

    ```bash
    curl --request POST \
        --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/crons \
        --header 'Content-Type: application/json' \
        --data '{
            "assistant_id": <ASSISTANT_ID>,
        }'
    ```

请注意，删除不再有用的 `Cron` 任务 **非常** 重要。否则你可能会因为不必要的 API 调用而产生费用！你可以使用以下代码删除 `Cron` 任务：

=== "Python"

    ```python
    await client.crons.delete(cron_job["cron_id"])
    ```

=== "Javascript"

    ```js
    await client.crons.delete(cronJob["cron_id"]);
    ```

=== "CURL"

    ```bash
    curl --request DELETE \
        --url <DEPLOYMENT_URL>/runs/crons/<CRON_ID>
    ```

## 无状态定时任务

你也可以使用以下代码创建无状态的定时任务：

=== "Python"

    ```python
    # 这调度了一个每天 15:27（下午 3:27）运行的任务
    cron_job_stateless = await client.crons.create(
        assistant_id,
        schedule="27 15 * * *",
        input={"messages": [{"role": "user", "content": "What time is it?"}]},
    )
    ```

=== "Javascript"

    ```js
    // 这调度了一个每天 15:27（下午 3:27）运行的任务
    const cronJobStateless = await client.crons.create(
      assistantId,
      {
        schedule: "27 15 * * *",
        input: { messages: [{ role: "user", content: "What time is it?" }] }
      }
    );
    ```

=== "CURL"

    ```bash
    curl --request POST \
        --url <DEPLOYMENT_URL>/runs/crons \
        --header 'Content-Type: application/json' \
        --data '{
            "assistant_id": <ASSISTANT_ID>,
        }'
    ```

再次提醒，任务完成后记得删除它！

=== "Python"

    ```python
    await client.crons.delete(cron_job_stateless["cron_id"])
    ```

=== "Javascript"

    ```js
    await client.crons.delete(cronJobStateless["cron_id"]);
    ```

=== "CURL"

    ```bash
    curl --request DELETE \
        --url <DEPLOYMENT_URL>/runs/crons/<CRON_ID>
    ```