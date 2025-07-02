# 在数据集上运行实验

LangGraph Studio支持通过让您在预定义的LangSmith数据集上运行助手来进行评估。这使您能够了解应用程序在各种输入上的表现，将结果与参考输出进行比较，并使用[评估器](../../../agents/evals.md)对结果进行评分。

本指南将向您展示如何在Studio中端到端运行实验。

---

## 先决条件

在运行实验之前，请确保具备以下条件：

1.  **LangSmith数据集**：您的数据集应包含要测试的输入，并可选择包含用于比较的参考输出。

    - 输入的Schema必须与助手所需的输入Schema匹配。有关Schema的更多信息，请参见[此处](../../../concepts/low_level.md#schema)。
    - 有关创建数据集的更多信息，请参见[如何管理数据集](https://docs.smith.langchain.com/evaluation/how_to_guides/manage_datasets_in_application#set-up-your-dataset)。

2.  **（可选）评估器**：您可以在LangSmith中为数据集附加评估器（例如LLM-as-a-Judge、启发式方法或自定义函数）。这些评估器将在图处理完所有输入后自动运行。

    - 了解更多信息，请阅读[评估概念](https://docs.smith.langchain.com/evaluation/concepts#evaluators)。

3.  **正在运行的应用程序**：实验可以针对以下对象运行：
    - 部署在[LangGraph平台](../../quick_start.md)上的应用程序。
    - 通过[langgraph-cli](../../../tutorials/langgraph-platform/local-server.md)启动的本地运行的应用程序。

---

## 分步指南

### 1. 启动实验

点击Studio页面右上角的**Run experiment**按钮。

### 2. 选择数据集

在弹出的模态框中，选择用于实验的数据集（或特定的数据集分割）并点击**Start**。

### 3. 监控进度

数据集中的所有输入现在都将针对活动的助手运行。通过右上角的徽章监控实验的进度。

您可以在实验在后台运行时继续在Studio中工作。随时点击箭头图标按钮导航到LangSmith并查看详细的实验结果。

---

## 故障排除

### "Run experiment"按钮不可用

如果"Run experiment"按钮不可用，请检查以下内容：

- **已部署的应用程序**：如果您的应用程序部署在LangGraph平台上，您可能需要创建一个新修订版以启用此功能。
- **本地开发服务器**：如果您在本地运行应用程序，请确保已升级到最新版本的`langgraph-cli`（`pip install -U langgraph-cli`）。此外，确保通过在项目的`.env`文件中设置`LANGSMITH_API_KEY`来启用跟踪。

### 评估器结果缺失

当您运行实验时，任何附加的评估器都会被安排在队列中执行。如果您没有立即看到结果，很可能意味着它们仍在等待处理。