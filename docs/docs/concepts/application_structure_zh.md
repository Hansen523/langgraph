# 应用结构

!!! info "前提条件"

    - [LangGraph 服务器](./langgraph_server.md)
    - [LangGraph 术语表](./low_level.md)

## 概述

一个 LangGraph 应用由一个或多个图、一个 LangGraph API 配置文件（`langgraph.json`）、一个指定依赖项的文件以及一个可选的 .env 文件（用于指定环境变量）组成。

本指南展示了 LangGraph 应用的典型结构，并展示了如何使用 LangGraph 平台部署 LangGraph 应用所需的信息。

## 关键概念

要使用 LangGraph 平台进行部署，应提供以下信息：

1. 一个 [LangGraph API 配置文件](#configuration-file)（`langgraph.json`），用于指定应用的依赖项、图和环境变量。
2. 实现应用逻辑的 [图](#graphs)。
3. 一个指定运行应用所需的 [依赖项](#dependencies) 的文件。
4. 运行应用所需的 [环境变量](#environment-variables)。

## 文件结构

以下是 Python 和 JavaScript 应用的目录结构示例：

=== "Python (requirements.txt)"

    ```plaintext
    my-app/
    ├── my_agent # 所有项目代码都放在这里
    │   ├── utils # 图的工具
    │   │   ├── __init__.py
    │   │   ├── tools.py # 图的工具
    │   │   ├── nodes.py # 图的节点函数
    │   │   └── state.py # 图的状态定义
    │   ├── __init__.py
    │   └── agent.py # 构建图的代码
    ├── .env # 环境变量
    ├── requirements.txt # 包依赖项
    └── langgraph.json # LangGraph 的配置文件
    ```
=== "Python (pyproject.toml)"

    ```plaintext
    my-app/
    ├── my_agent # 所有项目代码都放在这里
    │   ├── utils # 图的工具
    │   │   ├── __init__.py
    │   │   ├── tools.py # 图的工具
    │   │   ├── nodes.py # 图的节点函数
    │   │   └── state.py # 图的状态定义
    │   ├── __init__.py
    │   └── agent.py # 构建图的代码
    ├── .env # 环境变量
    ├── langgraph.json  # LangGraph 的配置文件
    └── pyproject.toml # 项目的依赖项
    ```

=== "JS (package.json)"

    ```plaintext
    my-app/
    ├── src # 所有项目代码都放在这里
    │   ├── utils # 可选的图的工具
    │   │   ├── tools.ts # 图的工具
    │   │   ├── nodes.ts # 图的节点函数
    │   │   └── state.ts # 图的状态定义
    │   └── agent.ts # 构建图的代码
    ├── package.json # 包依赖项
    ├── .env # 环境变量
    └── langgraph.json # LangGraph 的配置文件
    ```

!!! note

    LangGraph 应用的目录结构可能会因编程语言和使用的包管理器而有所不同。


## 配置文件

`langgraph.json` 文件是一个 JSON 文件，用于指定部署 LangGraph 应用所需的依赖项、图、环境变量和其他设置。

该文件支持指定以下信息：


| 键                | 描述                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
|--------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `dependencies`     | **必需**。LangGraph API 服务器的依赖项数组。依赖项可以是以下之一：(1) `"."`，将查找本地 Python 包，(2) `pyproject.toml`，`setup.py` 或 `requirements.txt` 在应用目录 `"./local_package"` 中，或 (3) 包名称。                                                                                                                                                                                                                                                        |
| `graphs`           | **必需**。从图 ID 到定义编译图或生成图的函数的路径的映射。示例：<ul><li>`./your_package/your_file.py:variable`，其中 `variable` 是 `langgraph.graph.state.CompiledStateGraph` 的实例</li><li>`./your_package/your_file.py:make_graph`，其中 `make_graph` 是一个接收配置字典（`langchain_core.runnables.RunnableConfig`）并创建 `langgraph.graph.state.StateGraph` / `langgraph.graph.state.CompiledStateGraph` 实例的函数。</li></ul> |
| `env`              | `.env` 文件的路径或从环境变量到其值的映射。                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| `python_version`   | `3.11` 或 `3.12`。默认为 `3.11`。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| `pip_config_file`  | `pip` 配置文件的路径。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| `dockerfile_lines` | 在从父镜像导入后添加到 Dockerfile 的附加行数组。                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
!!! tip

    LangGraph CLI 默认使用当前目录中的配置文件 **langgraph.json**。


### 示例

=== "Python"

    * 依赖项涉及一个自定义本地包和 `langchain_openai` 包。
    * 将从文件 `./your_package/your_file.py` 中加载一个图，变量为 `variable`。
    * 环境变量从 `.env` 文件中加载。

    ```json
    {
        "dependencies": [
            "langchain_openai",
            "./your_package"
        ],
        "graphs": {
            "my_agent": "./your_package/your_file.py:agent"
        },
        "env": "./.env"
    }
    ```

=== "JavaScript"

    * 依赖项将从本地目录中的依赖项文件（例如 `package.json`）加载。
    * 将从文件 `./your_package/your_file.js` 中加载一个图，函数为 `agent`。
    * 环境变量 `OPENAI_API_KEY` 内联设置。

    ```json
    {
        "dependencies": [
            "."
        ],
        "graphs": {
            "my_agent": "./your_package/your_file.js:agent"
        },
        "env": {
            "OPENAI_API_KEY": "secret-key"
        }
    }
    ```

## 依赖项

LangGraph 应用可能依赖于其他 Python 包或 JavaScript 库（取决于应用编写的编程语言）。

通常需要指定以下信息以正确设置依赖项：

1. 目录中指定依赖项的文件（例如 `requirements.txt`，`pyproject.toml` 或 `package.json`）。
2. [LangGraph 配置文件](#configuration-file) 中的 `dependencies` 键，用于指定运行 LangGraph 应用所需的依赖项。
3. 可以使用 [LangGraph 配置文件](#configuration-file) 中的 `dockerfile_lines` 键指定任何附加的二进制文件或系统库。

## 图

使用 [LangGraph 配置文件](#configuration-file) 中的 `graphs` 键指定部署的 LangGraph 应用中可用的图。

您可以在配置文件中指定一个或多个图。每个图由一个名称（应唯一）和以下路径标识：(1) 编译图或 (2) 定义生成图的函数。

## 环境变量

如果您在本地使用部署的 LangGraph 应用，可以在 [LangGraph 配置文件](#configuration-file) 的 `env` 键中配置环境变量。

对于生产部署，通常需要在部署环境中配置环境变量。

## 相关

请参阅以下资源以获取更多信息：

- [应用结构](../how-tos/index.md#application-structure) 的指南。