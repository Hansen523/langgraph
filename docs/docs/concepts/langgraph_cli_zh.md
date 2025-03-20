# LangGraph CLI

!!! info "前提条件"
    - [LangGraph 平台](./langgraph_platform.md)
    - [LangGraph 服务器](./langgraph_server.md)

LangGraph CLI 是一个跨平台的命令行工具，用于在本地构建和运行 [LangGraph API 服务器](./langgraph_server.md)。生成的服务器包括用于图运行、线程、助手等的所有 API 端点，以及运行代理所需的其他服务，包括用于检查点和存储的托管数据库。

## 安装

LangGraph CLI 可以通过 Homebrew（在 macOS 上）或 pip 安装：

=== "Homebrew"
    ```bash
    brew install langgraph-cli
    ```

=== "pip" 
    ```bash
    pip install langgraph-cli
    ```

## 命令

CLI 提供以下核心功能：

### `build`

`langgraph build` 命令为 [LangGraph API 服务器](./langgraph_server.md) 构建一个 Docker 镜像，可以直接部署。

### `dev`

!!! note "0.1.55 版本新增"
    `langgraph dev` 命令在 langgraph-cli 版本 0.1.55 中引入。

!!! note "仅支持 Python"

    目前，CLI 仅支持 Python >= 3.11。
    JS 支持即将推出。

`langgraph dev` 命令启动一个轻量级的开发服务器，无需安装 Docker。该服务器非常适合快速开发和测试，具有以下功能：

- 热重载：自动检测并重新加载代码更改
- 调试器支持：附加 IDE 的调试器进行逐行调试
- 具有本地持久性的内存状态：服务器状态存储在内存中以提高速度，但在重启之间保持本地持久性

要使用此命令，您需要安装带有 "inmem" 扩展的 CLI：

```bash
pip install -U "langgraph-cli[inmem]"
```

**注意**：此命令仅用于本地开发和测试。不推荐用于生产环境。由于它不使用 Docker，我们建议使用虚拟环境来管理项目的依赖项。

### `up`

`langgraph up` 命令在本地 Docker 容器中启动 [LangGraph API 服务器](./langgraph_server.md) 的一个实例。这需要本地运行 Docker 服务器。它还需要一个用于本地开发的 LangSmith API 密钥或用于生产使用的许可证密钥。

服务器包括用于图运行、线程、助手等的所有 API 端点，以及运行代理所需的其他服务，包括用于检查点和存储的托管数据库。

### `dockerfile`

`langgraph dockerfile` 命令生成一个 [Dockerfile](https://docs.docker.com/reference/dockerfile/)，可用于构建和部署 [LangGraph API 服务器](./langgraph_server.md) 的镜像。如果您希望进一步自定义 Dockerfile 或以更自定义的方式部署，这将非常有用。

??? note "更新您的 langgraph.json 文件"
    `langgraph dockerfile` 命令将 `langgraph.json` 文件中的所有配置转换为 Dockerfile 命令。使用此命令时，每当您更新 `langgraph.json` 文件时，您都必须重新运行它。否则，您的更改在构建或运行 Dockerfile 时将不会反映出来。

## 相关

- [LangGraph CLI API 参考](../cloud/reference/cli.md)