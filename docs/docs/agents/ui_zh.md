---
search:
  boost: 2
tags:
  - agent
hide:
  - tags
---

# 用户界面

您可以使用预构建的聊天界面[Agent Chat UI](https://github.com/langchain-ai/agent-chat-ui)与任何LangGraph智能体进行交互。使用[已部署版本](https://agentchat.vercel.app)是最快捷的入门方式，并允许您同时与本地和云端部署的图结构交互。

## 在UI中运行智能体

首先，[本地](./deployment.md#launch-langgraph-server-locally)设置LangGraph API服务器，或在[LangGraph平台](https://langchain-ai.github.io/langgraph/cloud/quick_start/)上部署您的智能体。

然后访问[Agent Chat UI](https://agentchat.vercel.app)，或克隆仓库并[在本地运行开发服务器](https://github.com/langchain-ai/agent-chat-ui?tab=readme-ov-file#setup):

<video controls src="../assets/base-chat-ui.mp4" type="video/mp4"></video>

!!! 提示

    该UI开箱即用地支持渲染工具调用和工具结果消息。要自定义显示哪些消息，请参阅Agent Chat UI文档中的[隐藏聊天消息](https://github.com/langchain-ai/agent-chat-ui?tab=readme-ov-file#hiding-messages-in-the-chat)部分。

## 添加人工介入

Agent Chat UI全面支持[人工介入](../concepts/human_in_the_loop.md)工作流。要体验此功能，请将`src/agent/graph.py`（来自[部署](./deployment.md)指南）中的智能体代码替换为这个[智能体实现](../how-tos/human_in_the_loop/add-human-in-the-loop.md#add-interrupts-to-any-tool):

<video controls src="../assets/interrupt-chat-ui.mp4" type="video/mp4"></video>

!!! 重要

    若您的LangGraph智能体使用[`HumanInterrupt`模式][langgraph.prebuilt.interrupt.HumanInterrupt]进行中断，Agent Chat UI将能发挥最佳效果。若未使用该模式，UI虽能渲染传递给`interrupt`函数的输入，但对恢复图结构的支持将不完全。

## 生成式UI

您也可以在Agent Chat UI中使用生成式UI功能。

生成式UI允许您定义[React](https://react.dev/)组件，并从LangGraph服务器推送到UI界面。有关构建生成式UI LangGraph智能体的更多文档，请阅读[这些说明](https://langchain-ai.github.io/langgraph/cloud/how-tos/generative_ui_react/)。