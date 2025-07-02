# 添加自定义认证

!!! tip "前提条件"

    本指南假设您熟悉以下概念：

      * [**认证与访问控制**](../../concepts/auth.md)
      * [**LangGraph平台**](../../concepts/langgraph_platform.md)
    
    如需更详细的引导式教程，请参阅[**设置自定义认证**](../../tutorials/auth/getting_started.md)教程。

???+ note "部署类型支持"

    **托管版LangGraph平台**的所有部署都支持自定义认证，**企业版**自托管方案也支持。**精简版**自托管方案不支持。

本指南展示如何为LangGraph平台应用添加自定义认证。本指南适用于LangGraph平台和自托管部署，不适用于在自定义服务器中单独使用LangGraph开源库的场景。

## 1. 实现认证逻辑

```python
from langgraph_sdk import Auth

my_auth = Auth()

@my_auth.authenticate
async def authenticate(authorization: str) -> str:
    token = authorization.split(" ", 1)[-1] # "Bearer <token>"
    try:
        # 用您的认证提供商验证令牌
        user_id = await verify_token(token)
        return user_id
    except Exception:
        raise Auth.exceptions.HTTPException(
            status_code=401,
            detail="无效令牌"
        )

# 添加授权规则来控制资源访问权限
@my_auth.on
async def add_owner(
    ctx: Auth.types.AuthContext,
    value: dict,
):
    """在资源元数据中添加所有者并过滤"""
    filters = {"owner": ctx.user.identity}
    metadata = value.setdefault("metadata", {})
    metadata.update(filters)
    return filters

# 假设存储组织形式为(user_id, resource_type, resource_id)
@my_auth.on.store()
async def authorize_store(ctx: Auth.types.AuthContext, value: dict):
    namespace: tuple = value["namespace"]
    assert namespace[0] == ctx.user.identity, "未经授权"
```

## 2. 更新配置文件

在`langgraph.json`中添加认证文件路径：

```json hl_lines="7-9"
{
  "dependencies": ["."],
  "graphs": {
    "agent": "./agent.py:graph"
  },
  "env": ".env",
  "auth": {
    "path": "./auth.py:my_auth"
  }
}
```

## 3. 客户端连接配置

在服务器端设置认证后，客户端请求必须包含基于所选认证方案的授权信息。假设使用JWT令牌认证，可以通过以下方式访问部署：

=== "Python客户端"

    ```python
    from langgraph_sdk import get_client

    my_token = "your-token" # 实际使用时应从认证提供商获取签名令牌
    client = get_client(
        url="http://localhost:2024",
        headers={"Authorization": f"Bearer {my_token}"}
    )
    threads = await client.threads.search()
    ```

=== "Python远程图"

    ```python
    from langgraph.pregel.remote import RemoteGraph
    
    my_token = "your-token" # 实际使用时应从认证提供商获取签名令牌
    remote_graph = RemoteGraph(
        "agent",
        url="http://localhost:2024",
        headers={"Authorization": f"Bearer {my_token}"}
    )
    threads = await remote_graph.ainvoke(...)
    ```

=== "JavaScript客户端"

    ```javascript
    import { Client } from "@langchain/langgraph-sdk";

    const my_token = "your-token"; // 实际使用时应从认证提供商获取签名令牌
    const client = new Client({
      apiUrl: "http://localhost:2024",
      defaultHeaders: { Authorization: `Bearer ${my_token}` },
    });
    const threads = await client.threads.search();
    ```

=== "JavaScript远程图"

    ```javascript
    import { RemoteGraph } from "@langchain/langgraph/remote";

    const my_token = "your-token"; // 实际使用时应从认证提供商获取签名令牌
    const remoteGraph = new RemoteGraph({
      graphId: "agent",
      url: "http://localhost:2024",
      headers: { Authorization: `Bearer ${my_token}` },
    });
    const threads = await remoteGraph.invoke(...);
    ```

=== "CURL命令"

    ```bash
    curl -H "Authorization: Bearer ${your-token}" http://localhost:2024/threads
    ```