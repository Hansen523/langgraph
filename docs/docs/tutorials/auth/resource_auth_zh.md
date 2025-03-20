# 使对话私密化（第二部分/三部分）

!!! note "这是我们的认证系列的第二部分："

    1. [基本认证](getting_started.md) - 控制谁可以访问你的机器人
    2. 资源授权（你在这里） - 让用户拥有私密对话
    3. [生产环境认证](add_auth_server.md) - 添加真实用户账户并使用OAuth2进行验证

在本教程中，我们将扩展我们的聊天机器人，使每个用户都能拥有自己的私密对话。我们将添加[资源级访问控制](../../concepts/auth.md#resource-level-access-control)，以便用户只能看到自己的对话线程。

![授权处理程序](./img/authorization.png)

???+ tip "占位符令牌"

    正如我们在[第一部分](getting_started.md)中所做的那样，在本节中，我们将使用一个硬编码的令牌进行说明。
    在掌握基础知识后，我们将在第三部分中实现一个“生产就绪”的认证方案。

## 理解资源授权

在上一教程中，我们控制了谁可以访问我们的机器人。但现在，任何经过认证的用户都可以看到其他人的对话！让我们通过添加[资源授权](../../concepts/auth.md#resource-authorization)来解决这个问题。

首先，确保你已经完成了[基本认证](getting_started.md)教程，并且你的安全机器人可以无错误地运行：

```bash
cd custom-auth
pip install -e .
langgraph dev --no-browser
```

> - 🚀 API: http://127.0.0.1:2024
> - 🎨 Studio UI: https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:2024
> - 📚 API文档: http://127.0.0.1:2024/docs

## 添加资源授权

回想一下，在上一教程中，[`Auth`](../../cloud/reference/sdk/python_sdk_ref.md#langgraph_sdk.auth.Auth)对象让我们注册了一个[认证函数](../../concepts/auth.md#authentication)，LangGraph平台使用它来验证传入请求中的承载令牌。现在我们将使用它来注册一个**授权**处理程序。

授权处理程序是在认证成功后运行的函数。这些处理程序可以向资源添加[元数据](../../concepts/auth.md#resource-metadata)（如资源的所有者），并过滤每个用户可以看到的内容。

让我们更新`src/security/auth.py`并添加一个在每个请求上运行的授权处理程序：

```python hl_lines="29-39" title="src/security/auth.py"
from langgraph_sdk import Auth

# 保留我们之前教程中的测试用户
VALID_TOKENS = {
    "user1-token": {"id": "user1", "name": "Alice"},
    "user2-token": {"id": "user2", "name": "Bob"},
}

auth = Auth()


@auth.authenticate
async def get_current_user(authorization: str | None) -> Auth.types.MinimalUserDict:
    """我们之前教程中的认证处理程序。"""
    assert authorization
    scheme, token = authorization.split()
    assert scheme.lower() == "bearer"

    if token not in VALID_TOKENS:
        raise Auth.exceptions.HTTPException(status_code=401, detail="Invalid token")

    user_data = VALID_TOKENS[token]
    return {
        "identity": user_data["id"],
    }


@auth.on
async def add_owner(
    ctx: Auth.types.AuthContext,  # 包含当前用户的信息
    value: dict,  # 正在创建/访问的资源
):
    """使资源对其创建者私有。"""
    # 示例：
    # ctx: AuthContext(
    #     permissions=[],
    #     user=ProxyUser(
    #         identity='user1',
    #         is_authenticated=True,
    #         display_name='user1'
    #     ),
    #     resource='threads',
    #     action='create_run'
    # )
    # value: 
    # {
    #     'thread_id': UUID('1e1b2733-303f-4dcd-9620-02d370287d72'),
    #     'assistant_id': UUID('fe096781-5601-53d2-b2f6-0d3403f7e9ca'),
    #     'run_id': UUID('1efbe268-1627-66d4-aa8d-b956b0f02a41'),
    #     'status': 'pending',
    #     'metadata': {},
    #     'prevent_insert_if_inflight': True,
    #     'multitask_strategy': 'reject',
    #     'if_not_exists': 'reject',
    #     'after_seconds': 0,
    #     'kwargs': {
    #         'input': {'messages': [{'role': 'user', 'content': 'Hello!'}]},
    #         'command': None,
    #         'config': {
    #             'configurable': {
    #                 'langgraph_auth_user': ... 你的用户对象...
    #                 'langgraph_auth_user_id': 'user1'
    #             }
    #         },
    #         'stream_mode': ['values'],
    #         'interrupt_before': None,
    #         'interrupt_after': None,
    #         'webhook': None,
    #         'feedback_keys': None,
    #         'temporary': False,
    #         'subgraphs': False
    #     }
    # }

    # 做两件事：
    # 1. 将用户的ID添加到资源的元数据中。每个LangGraph资源都有一个`metadata`字典，它与资源一起持久化。
    # 这个元数据在读取和更新操作中用于过滤
    # 2. 返回一个过滤器，让用户只能看到自己的资源
    filters = {"owner": ctx.user.identity}
    metadata = value.setdefault("metadata", {})
    metadata.update(filters)

    # 只让用户看到自己的资源
    return filters
```

处理程序接收两个参数：

1. `ctx` ([AuthContext](../../cloud/reference/sdk/python_sdk_ref.md#langgraph_sdk.auth.types.AuthContext)): 包含当前`user`的信息，用户的`permissions`，`resource`（"threads", "crons", "assistants"），以及正在执行的`action`（"create", "read", "update", "delete", "search", "create_run"）
2. `value` (`dict`): 正在创建或访问的数据。此字典的内容取决于正在访问的资源和操作。有关如何获得更严格范围的访问控制的信息，请参见下面的[添加范围授权处理程序](#scoped-authorization)。

请注意，我们的简单处理程序做了两件事：

1. 将用户的ID添加到资源的元数据中。
2. 返回一个元数据过滤器，以便用户只能看到他们拥有的资源。

## 测试私密对话

让我们测试我们的授权。如果我们设置正确，我们应该会看到所有✅消息。确保你的开发服务器正在运行（运行`langgraph dev`）：

```python
from langgraph_sdk import get_client

# 为两个用户创建客户端
alice = get_client(
    url="http://localhost:2024",
    headers={"Authorization": "Bearer user1-token"}
)

bob = get_client(
    url="http://localhost:2024",
    headers={"Authorization": "Bearer user2-token"}
)

# Alice创建一个助手
alice_assistant = await alice.assistants.create()
print(f"✅ Alice创建了助手: {alice_assistant['assistant_id']}")

# Alice创建一个线程并聊天
alice_thread = await alice.threads.create()
print(f"✅ Alice创建了线程: {alice_thread['thread_id']}")

await alice.runs.create(
    thread_id=alice_thread["thread_id"],
    assistant_id="agent",
    input={"messages": [{"role": "user", "content": "Hi, this is Alice's private chat"}]}
)

# Bob尝试访问Alice的线程
try:
    await bob.threads.get(alice_thread["thread_id"])
    print("❌ Bob不应该看到Alice的线程！")
except Exception as e:
    print("✅ Bob正确拒绝了访问:", e)

# Bob创建自己的线程
bob_thread = await bob.threads.create()
await bob.runs.create(
    thread_id=bob_thread["thread_id"],
    assistant_id="agent",
    input={"messages": [{"role": "user", "content": "Hi, this is Bob's private chat"}]}
)
print(f"✅ Bob创建了自己的线程: {bob_thread['thread_id']}")

# 列出线程 - 每个用户只能看到自己的
alice_threads = await alice.threads.search()
bob_threads = await bob.threads.search()
print(f"✅ Alice看到{len(alice_threads)}个线程")
print(f"✅ Bob看到{len(bob_threads)}个线程")
```

运行测试代码，你应该会看到如下输出：

```bash
✅ Alice创建了助手: fc50fb08-78da-45a9-93cc-1d3928a3fc37
✅ Alice创建了线程: 533179b7-05bc-4d48-b47a-a83cbdb5781d
✅ Bob正确拒绝了访问: Client error '404 Not Found' for url 'http://localhost:2024/threads/533179b7-05bc-4d48-b47a-a83cbdb5781d'
For more information check: https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/404
✅ Bob创建了自己的线程: 437c36ed-dd45-4a1e-b484-28ba6eca8819
✅ Alice看到1个线程
✅ Bob看到1个线程
```

这意味着：

1. 每个用户都可以创建并聊天在自己的线程中
2. 用户不能看到彼此的线程
3. 列出线程时只能看到自己的

## 添加范围授权处理程序 {#scoped-authorization}

广泛的`@auth.on`处理程序匹配所有[授权事件](../../concepts/auth.md#authorization-events)。这很简洁，但意味着`value`字典的内容没有很好地限定范围，并且我们对每个资源应用相同的用户级访问控制。如果我们想要更细粒度，我们还可以控制对资源的特定操作。

更新`src/security/auth.py`以添加特定资源类型的处理程序：

```python
# 保留我们之前的处理程序...

from langgraph_sdk import Auth

@auth.on.threads.create
async def on_thread_create(
    ctx: Auth.types.AuthContext,
    value: Auth.types.on.threads.create.value,
):
    """在创建线程时添加所有者。
    
    这个处理程序在创建新线程时运行，并做两件事：
    1. 在正在创建的线程上设置元数据以跟踪所有权
    2. 返回一个过滤器，确保只有创建者可以访问它
    """
    # 示例值：
    #  {'thread_id': UUID('99b045bc-b90b-41a8-b882-dabc541cf740'), 'metadata': {}, 'if_exists': 'raise'}

    # 在正在创建的线程上添加所有者元数据
    # 这个元数据与线程一起存储并持久化
    metadata = value.setdefault("metadata", {})
    metadata["owner"] = ctx.user.identity
    
    
    # 返回过滤器以限制访问仅限创建者
    return {"owner": ctx.user.identity}

@auth.on.threads.read
async def on_thread_read(
    ctx: Auth.types.AuthContext,
    value: Auth.types.on.threads.read.value,
):
    """只让用户读取自己的线程。
    
    这个处理程序在读取操作时运行。我们不需要设置
    元数据，因为线程已经存在 - 我们只需要
    返回一个过滤器，确保用户只能看到自己的线程。
    """
    return {"owner": ctx.user.identity}

@auth.on.assistants
async def on_assistants(
    ctx: Auth.types.AuthContext,
    value: Auth.types.on.assistants.value,
):
    # 为了说明目的，我们将拒绝所有涉及助手资源的请求
    # 示例值：
    # {
    #     'assistant_id': UUID('63ba56c3-b074-4212-96e2-cc333bbc4eb4'),
    #     'graph_id': 'agent',
    #     'config': {},
    #     'metadata': {},
    #     'name': 'Untitled'
    # }
    raise Auth.exceptions.HTTPException(
        status_code=403,
        detail="用户缺乏所需的权限。",
    )

# 假设你像这样在存储中组织信息（user_id, resource_type, resource_id）
@auth.on.store()
async def authorize_store(ctx: Auth.types.AuthContext, value: dict):
    # 每个存储项的“namespace”字段是一个元组，你可以将其视为项目的目录。
    namespace: tuple = value["namespace"]
    assert namespace[0] == ctx.user.identity, "未授权"
```

请注意，现在我们不再使用一个全局处理程序，而是有特定的处理程序用于：

1. 创建线程
2. 读取线程
3. 访问助手

前三个处理程序匹配每个资源的特定**操作**（参见[资源操作](../../concepts/auth.md#resource-actions)），而最后一个处理程序（`@auth.on.assistants`）匹配`assistants`资源的_任何_操作。对于每个请求，LangGraph将运行与正在访问的资源和操作最匹配的处理程序。这意味着上述四个处理程序将运行，而不是广泛范围的“`@auth.on`”处理程序。

尝试将以下测试代码添加到你的测试文件中：

```python
# ... 和之前一样
# 尝试创建一个助手。这应该失败
try:
    await alice.assistants.create("agent")
    print("❌ Alice不应该能够创建助手！")
except Exception as e:
    print("✅ Alice正确拒绝了访问:", e)

# 尝试搜索助手。这也应该失败
try:
    await alice.assistants.search()
    print("❌ Alice不应该能够搜索助手！")
except Exception as e:
    print("✅ Alice正确拒绝了搜索助手的访问:", e)

# Alice仍然可以创建线程
alice_thread = await alice.threads.create()
print(f"✅ Alice创建了线程: {alice_thread['thread_id']}")
```

然后再次运行测试代码：

```bash
✅ Alice创建了线程: dcea5cd8-eb70-4a01-a4b6-643b14e8f754
✅ Bob正确拒绝了访问: Client error '404 Not Found' for url 'http://localhost:2024/threads/dcea5cd8-eb70-4a01-a4b6-643b14e8f754'
For more information check: https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/404
✅ Bob创建了自己的线程: 400f8d41-e946-429f-8f93-4fe395bc3eed
✅ Alice看到1个线程
✅ Bob看到1个线程
✅ Alice正确拒绝了访问:
For more information check: https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/500
✅ Alice正确拒绝了搜索助手的访问:
```

恭喜！你已经构建了一个每个用户都有自己私密对话的聊天机器人。虽然这个系统使用简单的基于令牌的认证，但我们学到的授权模式将适用于实现任何真实的认证系统。在下一个教程中，我们将使用OAuth2替换我们的测试用户为真实用户账户。

## 下一步是什么？

现在你可以控制对资源的访问，你可能想要：

1. 继续学习[生产环境认证](add_auth_server.md)以添加真实用户账户
2. 阅读更多关于[授权模式](../../concepts/auth.md#authorization)的内容
3. 查看[API参考](../../cloud/reference/sdk/python_sdk_ref.md#langgraph_sdk.auth.Auth)以获取本教程中使用的接口和方法的详细信息