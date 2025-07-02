# 如何添加自定义生命周期事件

将代理部署到LangGraph平台时，通常需要在服务器启动时初始化资源（如数据库连接），并确保在关闭时正确释放这些资源。生命周期事件允许您接入服务器的启动和关闭流程，处理这些关键性的初始化和清理任务。

这与[添加自定义路由](./custom_routes.md)的方式相同。您只需提供自己的[`Starlette`](https://www.starlette.io/applications/)应用（包括[`FastAPI`](https://fastapi.tiangolo.com/)、[`FastHTML`](https://fastht.ml/)和其他兼容应用）。

以下是使用FastAPI的示例。

???+ note "仅限Python"

    目前我们仅支持在Python部署中使用`langgraph-api>=0.0.26`来实现自定义生命周期事件。

## 创建应用

从**现有**的LangGraph平台应用开始，将以下生命周期代码添加到您的`webapp.py`文件中。如果是从零开始，可以使用CLI从模板创建新应用。

```bash
langgraph new --template=new-langgraph-project-python my_new_project
```

拥有LangGraph项目后，添加以下应用代码：

```python
# ./src/agent/webapp.py
from contextlib import asynccontextmanager
from fastapi import FastAPI
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker

@asynccontextmanager
async def lifespan(app: FastAPI):
    # 示例代码...
    engine = create_async_engine("postgresql+asyncpg://user:pass@localhost/db")
    # 创建可重用的会话工厂
    async_session = sessionmaker(engine, class_=AsyncSession)
    # 存储在应用状态中
    app.state.db_session = async_session
    yield
    # 清理连接
    await engine.dispose()

# highlight-next-line
app = FastAPI(lifespan=lifespan)

# ... 可以添加自定义路由（如果需要）
```

## 配置`langgraph.json`

在`langgraph.json`配置文件中添加以下内容。确保路径指向您上面创建的`webapp.py`文件。

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

您应该会在服务器启动时看到初始化消息，在按下`Ctrl+C`停止时看到清理消息。

## 部署

您可以按现状将应用部署到LangGraph平台或自托管平台。

## 后续步骤

现在您已为部署添加了生命周期事件，可以使用类似技术添加[自定义路由](./custom_routes.md)或[自定义中间件](./custom_middleware.md)来进一步定制服务器行为。