# 可配置请求头

LangGraph允许通过运行时配置动态修改代理行为与权限。使用[LangGraph平台](../quick_start.md)时，您可以在请求体(`config`)或特定请求头中传递这些配置，从而实现基于用户身份或其他请求数据的调整。

为保护隐私，请通过`langgraph.json`文件中的`http.configurable_headers`部分控制哪些请求头会传递至运行时配置。

以下是如何定制包含与排除的请求头：

```json
{
  "http": {
    "configurable_headers": {
      "include": ["x-user-id", "x-organization-id", "my-prefix-*"],
      "exclude": ["authorization", "x-api-key"]
    }
  }
}
```

`include`和`exclude`列表支持精确匹配请求头名称，或使用`*`通配符匹配任意数量字符。出于安全考虑，不支持其他正则表达式模式。

## 在图中使用配置

您可以通过任何节点的`config`参数访问包含的请求头：

```python
def my_node(state, config):
  organization_id = config["configurable"].get("x-organization-id")
  ...
```

或通过上下文获取（适用于工具函数或其他嵌套函数）：

```python
from langgraph.config import get_config

def search_everything(query: str):
  organization_id = get_config()["configurable"].get("x-organization-id")
  ...
```

该机制甚至可用于动态编译图结构：

```python
# my_graph.py.
import contextlib

@contextlib.asynccontextmanager
async def generate_agent(config):
  organization_id = config["configurable"].get("x-organization-id")
  if organization_id == "org1":
    graph = ...
    yield graph
  else:
    graph = ...
    yield graph

```

```json
{
  "graphs": {"agent": "my_grph.py:generate_agent"}
}
```

### 禁用可配置请求头

如需完全禁用可配置请求头功能，只需在`exclude`列表设置通配符：

```json
{
  "http": {
    "configurable_headers": {
      "exclude": ["*"]
    }
  }
}
```

这将阻止所有请求头被添加到运行时配置中。

请注意排除规则优先级高于包含规则。