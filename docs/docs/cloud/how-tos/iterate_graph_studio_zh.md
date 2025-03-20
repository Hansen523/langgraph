# LangGraph Studio中的提示工程

在LangGraph Studio中，您可以通过使用LangSmith Playground来迭代图中使用的提示。具体步骤如下：

1. 打开一个现有线程或创建一个新线程。
2. 在线程日志中，任何进行了LLM调用的节点都会有一个“查看LLM运行”按钮。点击此按钮将打开一个弹出窗口，显示该节点的LLM运行情况。
3. 选择您想要编辑的LLM运行。这将打开LangSmith Playground，并加载所选的LLM运行。

![Studio中的Playground](../img/studio_playground.png){width=1200}

在此处，您可以编辑提示，测试不同的模型配置，并仅重新运行此LLM调用，而无需重新运行整个图。当您对更改满意时，可以将更新后的提示复制回您的图中。

有关如何使用LangSmith Playground的更多信息，请参阅[LangSmith Playground文档](https://docs.smith.langchain.com/prompt_engineering/how_to_guides#playground)。