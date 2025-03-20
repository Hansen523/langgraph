# 如何设置LangGraph应用程序以进行部署

为了将LangGraph应用程序部署到LangGraph Cloud（或自托管），必须配置[LangGraph API配置文件](../reference/cli.md#configuration-file)。本指南将讨论使用`pyproject.toml`定义包依赖关系以设置LangGraph应用程序进行部署的基本步骤。

本指南基于[此仓库](https://github.com/langchain-ai/langgraph-example-pyproject)，您可以通过它了解更多关于如何设置LangGraph应用程序以进行部署的信息。

!!! tip "使用requirements.txt进行设置"
    如果您更喜欢使用`requirements.txt`进行依赖管理，请查看[此指南](./setup.md)。

!!! tip "使用Monorepo进行设置"
    如果您有兴趣部署位于monorepo中的图，请查看[此](https://github.com/langchain-ai/langgraph-example-monorepo)仓库以了解如何操作。

最终的仓库结构将如下所示：

```bash
my-app/
├── my_agent # 所有项目代码都在这里
│   ├── utils # 图的实用工具
│   │   ├── __init__.py
│   │   ├── tools.py # 图的工具
│   │   ├── nodes.py # 图的节点函数
│   │   └── state.py # 图的状态定义
│   ├── __init__.py
│   └── agent.py # 构建图的代码
├── .env # 环境变量
├── langgraph.json  # LangGraph的配置文件
└── pyproject.toml # 项目的依赖关系
```

在每一步之后，都会提供一个示例文件目录，以展示如何组织代码。

## 指定依赖关系

依赖关系可以选择在以下文件中指定：`pyproject.toml`、`setup.py`或`requirements.txt`。如果未创建这些文件，则可以在[LangGraph API配置文件](#create-langgraph-api-config)中稍后指定依赖关系。

以下依赖关系将包含在镜像中，您也可以在代码中使用它们，只要版本范围兼容：

```
langgraph>=0.2.56,<0.4.0
langgraph-sdk>=0.1.53
langgraph-checkpoint>=2.0.15,<3.0
langchain-core>=0.2.38,<0.4.0
langsmith>=0.1.63
orjson>=3.9.7
httpx>=0.25.0
tenacity>=8.0.0
uvicorn>=0.26.0
sse-starlette>=2.1.0,<2.2.0
uvloop>=0.18.0
httptools>=0.5.0
jsonschema-rs>=0.20.0
structlog>=23.1.0
```

示例`pyproject.toml`文件：

```toml
[tool.poetry]
name = "my-agent"
version = "0.0.1"
description = "为LangGraph云构建的优秀代理。"
authors = ["Polly the parrot <1223+polly@users.noreply.github.com>"]
license = "MIT"
readme = "README.md"

[tool.poetry.dependencies]
python = ">=3.9.0,<3.13"
langgraph = "^0.2.0"
langchain-fireworks = "^0.1.3"


[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"
```

示例文件目录：

```bash
my-app/
└── pyproject.toml   # 图所需的Python包
```

## 指定环境变量

环境变量可以选择在文件（如`.env`）中指定。请参阅[环境变量参考](../reference/env_var.md)以配置部署的其他变量。

示例`.env`文件：

```
MY_ENV_VAR_1=foo
MY_ENV_VAR_2=bar
FIREWORKS_API_KEY=key
```

示例文件目录：

```bash
my-app/
├── .env             # 包含环境变量的文件
└── pyproject.toml
```

## 定义图

实现您的图！图可以在单个文件或多个文件中定义。请注意每个[CompiledGraph][langgraph.graph.graph.CompiledGraph]的变量名称，这些变量将包含在LangGraph应用程序中。变量名称将在稍后创建[LangGraph API配置文件](../reference/cli.md#configuration-file)时使用。

示例`agent.py`文件，展示了如何从您定义的其他模块导入（此处未显示模块的代码，请参见[此仓库](https://github.com/langchain-ai/langgraph-example-pyproject)以查看其实现）：

```python
# my_agent/agent.py
from typing import Literal
from typing_extensions import TypedDict

from langgraph.graph import StateGraph, END, START
from my_agent.utils.nodes import call_model, should_continue, tool_node # 导入节点
from my_agent.utils.state import AgentState # 导入状态

# 定义配置
class GraphConfig(TypedDict):
    model_name: Literal["anthropic", "openai"]

workflow = StateGraph(AgentState, config_schema=GraphConfig)
workflow.add_node("agent", call_model)
workflow.add_node("action", tool_node)
workflow.add_edge(START, "agent")
workflow.add_conditional_edges(
    "agent",
    should_continue,
    {
        "continue": "action",
        "end": END,
    },
)
workflow.add_edge("action", "agent")

graph = workflow.compile()
```

!!! warning "将`CompiledGraph`分配给变量"
    LangGraph Cloud的构建过程要求将`CompiledGraph`对象分配给Python模块顶层的变量。

示例文件目录：

```bash
my-app/
├── my_agent # 所有项目代码都在这里
│   ├── utils # 图的实用工具
│   │   ├── __init__.py
│   │   ├── tools.py # 图的工具
│   │   ├── nodes.py # 图的节点函数
│   │   └── state.py # 图的状态定义
│   ├── __init__.py
│   └── agent.py # 构建图的代码
├── .env
└── pyproject.toml
```

## 创建LangGraph API配置

创建一个名为`langgraph.json`的[LangGraph API配置文件](../reference/cli.md#configuration-file)。请参阅[LangGraph CLI参考](../reference/cli.md#configuration-file)以了解配置文件中JSON对象每个键的详细说明。

示例`langgraph.json`文件：

```json
{
  "dependencies": ["."],
  "graphs": {
    "agent": "./my_agent/agent.py:graph"
  },
  "env": ".env"
}
```

请注意，`CompiledGraph`的变量名称出现在顶级`graphs`键的每个子键值的末尾（即`:<variable_name>`）。

!!! warning "配置位置"
    LangGraph API配置文件必须放置在与包含编译图和相关依赖项的Python文件相同或更高层级的目录中。

示例文件目录：

```bash
my-app/
├── my_agent # 所有项目代码都在这里
│   ├── utils # 图的实用工具
│   │   ├── __init__.py
│   │   ├── tools.py # 图的工具
│   │   ├── nodes.py # 图的节点函数
│   │   └── state.py # 图的状态定义
│   ├── __init__.py
│   └── agent.py # 构建图的代码
├── .env # 环境变量
├── langgraph.json  # LangGraph的配置文件
└── pyproject.toml # 项目的依赖关系
```

## 下一步

在设置项目并将其放入github仓库后，是时候[部署您的应用程序](./cloud.md)了。