# 为什么选择LangGraph？

## LLM应用

LLMs使得将智能嵌入到新类别的应用程序中成为可能。使用LLMs构建应用程序有许多模式。[工作流](https://www.anthropic.com/research/building-effective-agents)围绕着LLM调用有预定义的代码路径框架。LLMs可以通过这些预定义的代码路径引导控制流，有些人认为这是一个“[代理系统](https://www.anthropic.com/research/building-effective-agents)”。在其他情况下，可以移除这个框架，创建能够[计划](https://huyenchip.com/2025/01/07/agents.html)、通过[工具调用](https://python.langchain.com/docs/concepts/tool_calling/)采取行动，并直接[响应其自身行动的反馈](https://research.google/blog/react-synergizing-reasoning-and-acting-in-language-models/)的自主代理。

![代理工作流](img/agent_workflow.png)

## LangGraph提供的内容

LangGraph提供了位于*任何*工作流或代理之下的低级支持基础设施。它不抽象提示或架构，并提供三个核心优势：

### 持久性

LangGraph有一个[持久层](https://langchain-ai.github.io/langgraph/concepts/persistence/)，提供了许多优势：

- [内存](https://langchain-ai.github.io/langgraph/concepts/memory/)：LangGraph持久化应用程序状态的任意方面，支持在用户交互内和跨用户交互的对话和其他更新的内存；
- [人在回路](https://langchain-ai.github.io/langgraph/concepts/human_in_the_loop/)：由于状态被检查点，执行可以中断和恢复，允许通过人工输入进行决策、验证和修正。

### 流式传输

LangGraph还支持在执行过程中将工作流/代理状态流式传输给用户（或开发者）。LangGraph支持流式传输[事件](https://langchain-ai.github.io/langgraph/how-tos/streaming.ipynb#updates)（如工具调用的反馈）和嵌入应用程序中的[LLM调用的令牌](https://langchain-ai.github.io/langgraph/how-tos/streaming-tokens.ipynb)。

### 调试和部署

LangGraph通过[LangGraph平台](https://langchain-ai.github.io/langgraph/concepts/langgraph_platform/)为测试、调试和部署应用程序提供了便捷的入门。这包括[Studio](https://langchain-ai.github.io/langgraph/concepts/langgraph_studio/)，一个允许可视化、交互和调试工作流或代理的IDE。这还包括许多[部署选项](https://langchain-ai.github.io/langgraph/tutorials/deployment/)。