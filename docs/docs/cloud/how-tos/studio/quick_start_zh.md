请将以下内容翻译成中文，一定要保持与原来内容格式一致，只翻译文本内容,不要翻译cell里面的内容。

!!! info "前提条件"

    - [LangGraph Studio 概述](../../../concepts/langgraph_stio.md)

LangGraph Studio 支持连接两种类型的图：

- 部署在 [LangGraph 平台](../../../cloud/quick_start.md) 上的图
- 通过 [LangGraph 服务器](../../../tutorials/langgraph-platform/local-server.md) 本地运行的图。

LangGraph Studio 可从 LangSmith UI 中访问，位于 LangGraph 平台部署选项卡内。

## 部署应用

对于已在 LangGraph 平台 [部署](../../quick_start.md) 的应用，您可以将 Studio 作为该部署的一部分进行访问。为此，请在 LangSmith UI 中导航至 LangGraph 平台内的部署，并点击“LangGraph Studio”按钮。

这将加载连接到您实时部署的 Studio UI，允许您在该部署中创建、读取和更新 [线程](../../../concepts/persistence.md#threads)、[助手](../../../concepts/assistants.md) 和 [记忆](../../../concepts//memory.md)。

## 本地开发服务器

要使用 LangGraph Studio 测试本地运行的应用，请确保您的应用按照 [此指南](https://langchain-ai.github.io/langgraph/cloud/deployment/setup/) 进行设置。

!!! info "LangSmith 追踪"
    对于本地开发，如果您不希望数据被追踪到 LangSmith，请在应用的 `.env` 文件中设置 `LANGSMITH_TRACING=false`。禁用追踪后，数据不会离开您的本地服务器。

接下来，安装 [LangGraph CLI](../../../concepts/langgraph_cli.md)：

```
pip install -U "langgraph-cli[inmem]"
```

并运行：

```
langgraph dev
```

!!! warning "浏览器兼容性"
    Safari 会阻止 `localhost` 连接至 Studio。要解决此问题，请使用 `--tunnel` 运行上述命令，通过安全隧道访问 Studio。

这将启动本地 LangGraph 服务器，以内存模式运行。服务器将以监视模式运行，监听代码变更并自动重启。阅读此 [参考](https://langchain-ai.github.io/langgraph/cloud/reference/cli/#dev) 了解启动 API 服务器的所有选项。

如果成功，您将看到以下日志：

> 准备就绪！
>
> - API: [http://localhost:2024](http://localhost:2024/)
>
> - 文档: http://localhost:2024/docs
>
> - LangGraph Studio Web UI: https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:2024

运行后，您将被自动引导至 LangGraph Studio。

对于已运行的服务器，可通过以下方式访问 Studio：

1. 直接导航至以下 URL：`https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:2024`。
2. 在 LangSmith 中，导航至 LangGraph 平台部署选项卡，点击“LangGraph Studio”按钮，输入 `http://127.0.0.1:2024` 并点击“连接”。

如果服务器运行在不同的主机或端口上，只需更新 `baseUrl` 以匹配。

### （可选）附加调试器

如需逐步调试，包括断点和变量检查：

```bash
# 安装 debugpy 包
pip install debugpy

# 启用调试启动服务器
langgraph dev --debug-port 5678
```

然后附加您偏好的调试器：

=== "VS Code"

    将此配置添加到 `launch.json`：

    ```json
    {
        "name": "附加到 LangGraph",
        "type": "debugpy",
        "request": "attach",
        "connect": {
          "host": "0.0.0.0",
          "port": 5678
        }
    }
    ```

=== "PyCharm" 

    1. 转到运行 → 编辑配置 
    2. 点击 + 并选择“Python 调试服务器” 
    3. 设置 IDE 主机名：`localhost` 
    4. 设置端口：`5678`（或您在上一步中选择的端口号） 
    5. 点击“确定”并开始调试

## 故障排除

如遇启动问题，请参阅此 [故障排除指南](../../../troubleshooting/studio.md)。

## 后续步骤

查看以下指南以获取更多关于如何使用 Studio 的信息：

- [运行应用](../invoke_studio.md)
- [管理助手](./manage_assistants.md)
- [管理线程](../threads_studio.md)
- [迭代提示](../iterate_graph_studio.md)
- [调试 LangSmith 追踪](../clone_traces_studio.md)
- [添加节点至数据集](../datasets_studio.md)