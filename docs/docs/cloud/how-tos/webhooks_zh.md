# 使用Webhooks

在使用LangGraph Cloud时，您可能希望使用webhooks在API调用完成后接收更新。Webhooks对于在运行处理完成后触发服务中的操作非常有用。要实现这一点，您需要暴露一个可以接受`POST`请求的端点，并在API请求中将此端点作为`webhook`参数传递。

目前，SDK并未提供内置支持来定义webhook端点，但您可以通过API请求手动指定它们。

## 支持的端点

以下API端点接受`webhook`参数：

| 操作 | HTTP方法 | 端点 |
|-----------|------------|----------|
| 创建运行 | `POST` | `/thread/{thread_id}/runs` |
| 创建线程定时任务 | `POST` | `/thread/{thread_id}/runs/crons` |
| 流式运行 | `POST` | `/thread/{thread_id}/runs/stream` |
| 等待运行 | `POST` | `/thread/{thread_id}/runs/wait` |
| 创建定时任务 | `POST` | `/runs/crons` |
| 无状态流式运行 | `POST` | `/runs/stream` |
| 无状态等待运行 | `POST` | `/runs/wait` |

在本指南中，我们将展示如何在流式运行后触发webhook。

## 设置您的助手和线程

在进行API调用之前，请先设置您的助手和线程。

=== "Python"
```python
from langgraph_sdk import get_client

client = get_client(url=<DEPLOYMENT_URL>)
assistant_id = "agent"
thread = await client.threads.create()
print(thread)
```

=== "JavaScript"
```js
import { Client } from "@langchain/langgraph-sdk";

const client = new Client({ apiUrl: <DEPLOYMENT_URL> });
const assistantID = "agent";
const thread = await client.threads.create();
console.log(thread);
```

=== "CURL"
```bash
curl --request POST \
    --url <DEPLOYMENT_URL>/assistants/search \
    --header 'Content-Type: application/json' \
    --data '{ "limit": 10, "offset": 0 }' | jq -c 'map(select(.config == null or .config == {})) | .[0]' && \
curl --request POST \
    --url <DEPLOYMENT_URL>/threads \
    --header 'Content-Type: application/json' \
    --data '{}'
```

### 示例响应
```json
{
    "thread_id": "9dde5490-2b67-47c8-aa14-4bfec88af217",
    "created_at": "2024-08-30T23:07:38.242730+00:00",
    "updated_at": "2024-08-30T23:07:38.242730+00:00",
    "metadata": {},
    "status": "idle",
    "config": {},
    "values": null
}
```

## 在图形运行中使用Webhook

要使用webhook，请在API请求中指定`webhook`参数。当运行完成时，LangGraph Cloud会向指定的webhook URL发送`POST`请求。

例如，如果您的服务器在`https://my-server.app/my-webhook-endpoint`监听webhook事件，请在请求中包含此URL：

=== "Python"
```python
input = { "messages": [{ "role": "user", "content": "Hello!" }] }

async for chunk in client.runs.stream(
    thread_id=thread["thread_id"],
    assistant_id=assistant_id,
    input=input,
    stream_mode="events",
    webhook="https://my-server.app/my-webhook-endpoint"
):
    pass
```

=== "JavaScript"
```js
const input = { messages: [{ role: "human", content: "Hello!" }] };

const streamResponse = client.runs.stream(
  thread["thread_id"],
  assistantID,
  {
    input: input,
    webhook: "https://my-server.app/my-webhook-endpoint"
  }
);

for await (const chunk of streamResponse) {
  // 处理流输出
}
```

=== "CURL"
```bash
curl --request POST \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/stream \
    --header 'Content-Type: application/json' \
    --data '{
        "assistant_id": <ASSISTANT_ID>,
        "input": {"messages": [{"role": "user", "content": "Hello!"}]},
        "webhook": "https://my-server.app/my-webhook-endpoint"
    }'
```

## Webhook负载

LangGraph Cloud以[Run](../../concepts/langgraph_server.md/#runs)的格式发送webhook通知。有关详细信息，请参阅[API参考](https://langchain-ai.github.io/langgraph/cloud/reference/api/api_ref.html#model/run)。请求负载包括运行输入、配置和其他元数据，这些信息位于`kwargs`字段中。

## 保护Webhooks

为确保只有经过授权的请求才能访问您的webhook端点，请考虑将安全令牌作为查询参数添加：

```
https://my-server.app/my-webhook-endpoint?token=YOUR_SECRET_TOKEN
```

您的服务器应在处理请求之前提取并验证此令牌。

## 测试Webhooks

您可以使用以下在线服务测试您的webhook：

- **[Beeceptor](https://beeceptor.com/)** – 快速创建测试端点并检查传入的webhook负载。
- **[Webhook.site](https://webhook.site/)** – 实时查看、调试和记录传入的webhook请求。

这些工具可以帮助您验证LangGraph Cloud是否正确触发并向您的服务发送webhooks。

---

通过遵循这些步骤，您可以将webhooks集成到LangGraph Cloud工作流中，根据完成的运行自动执行操作。