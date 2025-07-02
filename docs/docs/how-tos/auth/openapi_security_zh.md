# 在OpenAPI中记录API认证

本指南展示如何为LangGraph平台API文档定制OpenAPI安全模式。完善的认证文档能帮助API使用者理解如何认证，甚至支持自动客户端生成。更多关于LangGraph认证系统的细节，请参阅[认证与访问控制概念指南](../../concepts/auth.md)。

!!! note "实现与文档"
    本指南仅涵盖如何在OpenAPI中记录安全要求。要实现实际认证逻辑，请参阅[如何添加自定义认证](./custom_auth.md)。

本指南适用于所有LangGraph平台部署（云和自托管）。如果不使用LangGraph平台，仅使用LangGraph开源库则不适用。

## 默认模式

默认安全方案因部署类型而异：

=== "LangGraph平台"

默认情况下，LangGraph平台要求在`x-api-key`头中提供LangSmith API密钥：

```yaml
components:
  securitySchemes:
    apiKeyAuth:
      type: apiKey
      in: header
      name: x-api-key
security:
  - apiKeyAuth: []
```

使用LangGraph SDK时，可从环境变量中推断。

=== "自托管"

默认情况下，自托管部署没有安全方案。这意味着它们只能在安全网络中部署或需要认证。要添加自定义认证，请参阅[如何添加自定义认证](./custom_auth.md)。

## 自定义安全模式

要定制OpenAPI文档中的安全模式，在`langgraph.json`的`auth`配置中添加`openapi`字段。注意这仅更新API文档——还需实现相应认证逻辑，如[如何添加自定义认证](./custom_auth.md)所示。

注意LangGraph平台不提供认证端点——需在客户端应用中处理用户认证，并将凭证传递给LangGraph API。

=== "OAuth2 Bearer令牌"

    ```json
    {
      "auth": {
        "path": "./auth.py:my_auth",  // 在此实现认证逻辑
        "openapi": {
          "securitySchemes": {
            "OAuth2": {
              "type": "oauth2",
              "flows": {
                "implicit": {
                  "authorizationUrl": "https://your-auth-server.com/oauth/authorize",
                  "scopes": {
                    "me": "读取当前用户信息",
                    "threads": "访问创建和管理线程"
                  }
                }
              }
            }
          },
          "security": [
            {"OAuth2": ["me", "threads"]}
          ]
        }
      }
    }
    ```

=== "API密钥"

    ```json
    {
      "auth": {
        "path": "./auth.py:my_auth",  // 在此实现认证逻辑
        "openapi": {
          "securitySchemes": {
            "apiKeyAuth": {
              "type": "apiKey",
              "in": "header",
              "name": "X-API-Key"
            }
          },
          "security": [
            {"apiKeyAuth": []}
          ]
        }
      }
    }
    ```

## 测试

更新配置后：

1. 部署应用
2. 访问`/docs`查看更新的OpenAPI文档
3. 使用认证服务器的凭据测试端点（确保已先实现认证逻辑）