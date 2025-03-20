# 模板应用

模板是开源参考应用，旨在帮助您在使用LangGraph进行构建时快速入门。它们提供了常见的代理工作流程的实用示例，可以根据您的需求进行定制。

您可以使用LangGraph CLI从模板创建应用程序。

!!! info "要求"

    - Python >= 3.11
    - [LangGraph CLI](https://langchain-ai.github.io/langgraph/cloud/reference/cli/): 需要 langchain-cli[inmem] >= 0.1.58

## 安装LangGraph CLI

```bash
pip install "langgraph-cli[inmem]" --upgrade
```

## 可用模板

| 模板                  | 描述                                                                              | Python                                                           | JS/TS                                                               |
|---------------------------|------------------------------------------------------------------------------------------|------------------------------------------------------------------|---------------------------------------------------------------------|
| **新LangGraph项目** | 一个简单、最小化的带有记忆功能的聊天机器人。                                                   | [Repo](https://github.com/langchain-ai/new-langgraph-project)    | [Repo](https://github.com/langchain-ai/new-langgraphjs-project)     |
| **ReAct 代理**           | 一个可以灵活扩展到许多工具的简单代理。                              | [Repo](https://github.com/langchain-ai/react-agent)              | [Repo](https://github.com/langchain-ai/react-agent-js)              |
| **记忆代理**          | 一个ReAct风格的代理，带有一个额外的工具来存储跨线程使用的记忆。    | [Repo](https://github.com/langchain-ai/memory-agent)             | [Repo](https://github.com/langchain-ai/memory-agent-js)             |
| **检索代理**       | 一个包含基于检索的问答系统的代理。                      | [Repo](https://github.com/langchain-ai/retrieval-agent-template) | [Repo](https://github.com/langchain-ai/retrieval-agent-template-js) |
| **数据丰富代理** | 一个执行网络搜索并将其发现组织成结构化格式的代理。 | [Repo](https://github.com/langchain-ai/data-enrichment)          | [Repo](https://github.com/langchain-ai/data-enrichment-js)          |


## 🌱 创建LangGraph应用

要从模板创建新应用，请使用 `langgraph new` 命令。

```bash
langgraph new
```

## 下一步

查看新LangGraph应用根目录中的 `README.md` 文件，以获取有关模板以及如何自定义的更多信息。

在正确配置应用并添加API密钥后，您可以使用LangGraph CLI启动应用：

```bash
langgraph dev 
```

请参阅以下指南，以获取有关如何部署应用的更多信息：

- **[启动本地LangGraph服务器](../tutorials/langgraph-platform/local-server.md)**: 本快速入门指南展示了如何为 **ReAct 代理** 模板在本地启动LangGraph服务器。其他模板的步骤类似。
- **[部署到LangGraph云](../cloud/quick_start.md)**: 使用LangGraph云部署您的LangGraph应用。
 
### LangGraph框架

- **[LangGraph概念](../concepts/index.md)**: 了解LangGraph的基础概念。
- **[LangGraph操作指南](../how-tos/index.md)**: 有关LangGraph常见任务的指南。

### 📚 了解更多关于LangGraph平台的信息

通过这些资源扩展您的知识：

- **[LangGraph平台概念](../concepts/index.md#langgraph-platform)**: 了解LangGraph平台的基础概念。
- **[LangGraph平台操作指南](../how-tos/index.md#langgraph-platform)**: 发现构建和部署应用的分步指南。