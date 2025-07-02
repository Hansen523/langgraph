# 如何添加自定义路由

当将智能体部署到LangGraph平台时，您的服务器会自动暴露以下路由：创建运行和线程、与长期记忆存储交互、管理可配置助手以及其他核心功能（[查看所有默认API端点](../../cloud/reference/api/api_ref.md)）。

您可以通过提供自己的[`Starlette`](https://www.starlette.io/applications/)应用（包括[`FastAPI`](https://fastapi.tiangolo.com/)、[`FastHTML`](https://fastht.ml/)等兼容应用）来添加自定义路由。通过在`langgraph.json`配置文件中指定应用的路径，让LangGraph平台知晓这个自定义应用。

定义自定义应用对象允许您添加任何想要的路由，从添加`/login`端点到编写完整的全栈Web应用都可以实现，所有功能都部署在同一个LangGraph服务器中。

以下是使用FastAPI的示例。

## 创建应用

从**现有的**LangGraph平台应用开始，将以下自定义路由代码添加到您的`webapp.py`文件中。如果您是从零开始，可以使用CLI从模板创建一个新应用。

```bash
langgraph new --template=new-langgraph-project-python my_new_project
```

有了LangGraph项目后，添加以下应用代码：

```python
# ./src/agent/webapp.py
from fastapi import FastAPI

# highlight-next-line
app = FastAPI()


@app.get("/hello")
def read_root():
    return {"Hello": "World"}

```

## 配置`langgraph.json`

在`langgraph.json`配置文件中添加以下内容。确保路径指向您在`webapp.py`文件中创建的FastAPI应用实例`app`。

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
  // 其他配置选项如auth、store等
}
```

## 启动服务器

在本地测试服务器：

```bash
langgraph dev --no-browser
```

如果在浏览器中访问`localhost:2024/hello`（`2024`是默认开发端口），您应该看到`/hello`端点返回`{"Hello": "World"}`。


!!! note "覆盖默认端点"

    您在应用中创建的路由优先于系统默认路由，这意味着您可以覆盖并重新定义任何默认端点的行为。

## 部署

您可以按原样将此应用部署到LangGraph平台或自托管平台。

## 下一步

现在您已向部署中添加了自定义路由，可以使用相同的技术进一步自定义服务器的行为，例如定义[自定义中间件](./custom_middleware.md)和[自定义生命周期事件](./custom_lifespan.md)。