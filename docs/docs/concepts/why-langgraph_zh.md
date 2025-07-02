# 概述

LangGraph专为希望构建强大、适应性强的AI智能体的开发者而设计。开发者选择LangGraph的原因是：

- **可靠性与可控性**。通过审核检查和人机协同批准机制来引导智能体行为。LangGraph为长时间运行的工作流保存上下文，确保您的智能体始终不偏离轨道。
- **底层可扩展架构**。使用完全描述性的底层原语构建自定义智能体，不受限制定制化的僵化抽象束缚。设计可扩展的多智能体系统，每个智能体都能根据您的用例需求扮演特定角色。
- **一流的流式支持**。通过逐令牌流式传输和中间步骤流式输出，LangGraph让用户可以实时清晰地观察智能体的推理过程和行为轨迹。

## 学习LangGraph基础

要掌握LangGraph的核心概念和功能，请完成以下基础教程系列：

1. [构建基础聊天机器人](../tutorials/get-started/1-build-basic-chatbot.md)
2. [添加工具](../tutorials/get-started/2-add-tools.md)
3. [添加记忆功能](../tutorials/get-started/3-add-memory.md)
4. [添加人机协同控制](../tutorials/get-started/4-human-in-the-loop.md)
5. [自定义状态](../tutorials/get-started/5-customize-state.md)
6. [时间回溯](../tutorials/get-started/6-time-travel.md)

完成本教程系列后，您将用LangGraph构建出一个具备以下能力的客服聊天机器人：

* ✅ **回答常见问题**（通过联网搜索）
* ✅ **保持跨对话的状态记忆**
* ✅ **将复杂查询路由**至人工审核
* ✅ **使用自定义状态**控制行为
* ✅ **回溯探索**不同对话路径