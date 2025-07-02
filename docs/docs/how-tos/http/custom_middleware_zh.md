# 如何添加自定义中间件

在将代理部署到LangGraph平台时，您可以在不修改核心服务器逻辑的情况下，通过添加自定义中间件来处理日志记录请求指标、注入或检查标头以及强制执行安全策略等问题。这与[添加自定义路由](./custom_routes.md)的方式相同。您只需要提供自己的[`Starlette`](https://www.starlette.io/applications/)应用（包括[`FastAPI`](https://fastapi.tiangolo.com/)、[`FastHTML`](https://fastht.ml/)和其他兼容应用）。

添加中间件可以让您在整个部署过程中全局拦截和修改请求和响应，无论它们是访问您的自定义端点还是内置的LangGraph平台API。

以下是一个使用FastAPI的示例。

???+ note "仅限Python"

    目前我们仅支持在Python部署中使用`langgraph-api>=0.0.26`添加自定义中间件。

## 创建应用

从一个**现有的**LangGraph平台应用开始，将以下中间件代码添加到您的`webapp.py`文件中。如果您是从零开始，可以使用CLI从模板创建一个新应用。

```bash
langgraph new --template=new-langgraph-project-python my_new_project
```

一旦您有了一个LangGraph项目，添加以下应用代码：

```python
# ./src/agent/webapp.py
from fastapi import FastAPI, Request
from starlette.middleware.base import BaseHTTPMiddleware

# highlight-next-line
app = FastAPI()

class CustomHeaderMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        response = await call_next(request)
        response.headers['X-Custom-Header'] = 'Hello from middleware!'
        return response

# Add the middleware to the app
app.add_middleware(CustomHeaderMiddleware)
```

## 配置`langgraph.json`

将以下内容添加到您的`langgraph.json`配置文件中。确保路径指向您上面创建的`webapp.py`文件。

```json
{
  "dependencies": ["."],
  "graphs": {
    "agent": "./src/agent/graph.py:graph"
  },
  "env": ".env",
  "http": {
    "app": "./src/agent/webapp.py:app"
  }
  // Other configuration options like auth, store, etc.
}
```

## 启动服务器

在本地测试服务器：

```bash
langgraph dev --no-browser
```

现在，任何对您服务器的请求都会在其响应中包含自定义标头 `X-Custom-Header`。

## 部署

您可以按原样将此应用部署到LangGraph平台或您的自托管平台。

## 下一步

现在您已经为您的部署添加了自定义中间件，您可以使用类似的技术来添加[自定义路由](./custom_routes.md)或定义[自定义生命周期事件](./custom_lifespan.md)，以进一步定制服务器的行为。