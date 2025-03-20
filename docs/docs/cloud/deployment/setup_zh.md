# 如何设置 LangGraph 应用程序以进行部署

为了将 LangGraph 应用程序部署到 LangGraph Cloud（或自托管），必须使用 [LangGraph API 配置文件](../reference/cli.md#configuration-file) 进行配置。本指南将讨论使用 `requirements.txt` 指定项目依赖项以设置 LangGraph 应用程序进行部署的基本步骤。

本指南基于 [此仓库](https://github.com/langchain-ai/langgraph-example)，您可以通过它来了解更多关于如何设置 LangGraph 应用程序以进行部署的信息。

!!! tip "使用 pyproject.toml 进行设置"
    如果您更喜欢使用 poetry 进行依赖管理，请查看 [此指南](./setup_pyproject.md)，了解如何使用 `pyproject.toml` 进行 LangGraph Cloud 设置。

!!! tip "使用 Monorepo 进行设置"
    如果您有兴趣部署位于 monorepo 中的图，请查看 [此](https://github.com/langchain-ai/langgraph-example-monorepo) 仓库以获取示例。

最终的仓库结构将如下所示：

```bash
my-app/
├── my_agent # 所有项目代码都在这里
│   ├── utils # 图的实用工具
│   │   ├── __init__.py
│   │   ├── tools.py # 图的工具
│   │   ├── nodes.py # 图的节点函数
│   │   └── state.py # 图的状态定义
│   ├── requirements.txt # 包依赖项
│   ├── __init__.py
│   └── agent.py # 构建图的代码
├── .env # 环境变量
└── langgraph.json # LangGraph 的配置文件
```

每个步骤后，都会提供一个示例文件目录，以展示如何组织代码。

## 指定依赖项

依赖项可以选择在以下文件之一中指定：`pyproject.toml`、`setup.py` 或 `requirements.txt`。如果未创建这些文件，则可以在稍后的 [LangGraph API 配置文件](#create-langgraph-api-config) 中指定依赖项。

以下依赖项将包含在镜像中，您也可以在代码中使用它们，只要版本范围兼容：

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

示例 `requirements.txt` 文件：

```
langgraph
langchain_anthropic
tavily-python
langchain_community
langchain_openai

```

示例文件目录：

```bash
my-app/
├── my_agent # 所有项目代码都在这里
│   └── requirements.txt # 包依赖项
```

## 指定环境变量

环境变量可以选择在文件（例如 `.env`）中指定。请参阅 [环境变量参考](../reference/env_var.md) 以配置部署的其他变量。

示例 `.env` 文件：

```
MY_ENV_VAR_1=foo
MY_ENV_VAR_2=bar
OPENAI_API_KEY=key
```

示例文件目录：

```bash
my-app/
├── my_agent # 所有项目代码都在这里
│   └── requirements.txt # 包依赖项
└── .env # 环境变量
```

## 定义图

实现您的图！图可以在单个文件或多个文件中定义。请注意每个 [CompiledGraph][langgraph.graph.graph.CompiledGraph] 的变量名称，这些变量名称将包含在 LangGraph 应用程序中。这些变量名称将在稍后创建 [LangGraph API 配置文件](../reference/cli.md#configuration-file) 时使用。

示例 `agent.py` 文件，展示了如何从您定义的其他模块导入（此处未展示模块的代码，请参阅 [此仓库](https://github.com/langchain-ai/langgraph-example) 以查看其实现）：

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

!!! warning "将 `CompiledGraph` 分配给变量"
    LangGraph Cloud 的构建过程要求将 `CompiledGraph` 对象分配给 Python 模块的顶级变量（或者，您可以提供 [创建图的函数](./graph_rebuild.md)）。

示例文件目录：

```bash
my-app/
├── my_agent # 所有项目代码都在这里
│   ├── utils # 图的实用工具
│   │   ├── __init__.py
│   │   ├── tools.py # 图的工具
│   │   ├── nodes.py # 图的节点函数
│   │   └── state.py # 图的状态定义
│   ├── requirements.txt # 包依赖项
│   ├── __init__.py
│   └── agent.py # 构建图的代码
└── .env # 环境变量
```

## 创建 LangGraph API 配置

创建一个名为 `langgraph.json` 的 [LangGraph API 配置文件](../reference/cli.md#configuration-file)。请参阅 [LangGraph CLI 参考](../reference/cli.md#configuration-file) 以获取配置文件中 JSON 对象每个键的详细解释。

示例 `langgraph.json` 文件：

```json
{
  "dependencies": ["./my_agent"],
  "graphs": {
    "agent": "./my_agent/agent.py:graph"
  },
  "env": ".env"
}
```

请注意，`CompiledGraph` 的变量名称出现在顶级 `graphs` 键的每个子键值的末尾（即 `:<variable_name>`）。

!!! warning "配置位置"
    LangGraph API 配置文件必须放置在与包含编译图和相关依赖项的 Python 文件相同或更高级别的目录中。

示例文件目录：

```bash
my-app/
├── my_agent # 所有项目代码都在这里
│   ├── utils # 图的实用工具
│   │   ├── __init__.py
│   │   ├── tools.py # 图的工具
│   │   ├── nodes.py # 图的节点函数
│   │   └── state.py # 图的状态定义
│   ├── requirements.txt # 包依赖项
│   ├── __init__.py
│   └── agent.py # 构建图的代码
├── .env # 环境变量
└── langgraph.json # LangGraph 的配置文件
```

## 下一步

在设置好项目并将其放入 GitHub 仓库后，就可以 [部署您的应用程序](./cloud.md) 了。