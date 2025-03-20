---
title: 教程
---

# 教程

刚接触LangGraph或LLM应用开发？阅读这些材料，快速上手构建你的第一个应用。

## 快速开始 🚀 {#quick-start}

- [LangGraph快速入门](introduction.ipynb)：构建一个可以使用工具并跟踪对话历史的聊天机器人。添加人机交互功能，探索时间旅行的工作原理。
- [常见工作流](workflows/index.md)：使用LangGraph实现的最常见LLM工作流概述。
- [LangGraph服务器快速入门](langgraph-platform/local-server.md)：在本地启动LangGraph服务器，并使用REST API和LangGraph Studio Web UI与其交互。
- [LangGraph模板快速入门](../concepts/template_applications.md)：使用模板应用开始构建LangGraph平台。
- [使用LangGraph云快速部署](../cloud/quick_start.md)：使用LangGraph云部署LangGraph应用。

## 使用案例 🛠️ {#use-cases}

探索针对特定场景的实用实现：

### 聊天机器人

- [客户支持](customer-support/customer-support.ipynb)：构建一个多功能的航班、酒店和租车支持机器人。
- [从用户需求生成提示](chatbots/information-gather-prompting.ipynb)：构建一个信息收集聊天机器人。
- [代码助手](code_assistant/langgraph_code_assistant.ipynb)：构建一个代码分析和生成助手。

### RAG

- [代理式RAG](rag/langgraph_agentic_rag.ipynb)：使用代理在回答用户问题之前确定如何检索最相关的信息。
- [自适应RAG](rag/langgraph_adaptive_rag.ipynb)：自适应RAG是一种将（1）查询分析与（2）主动/自我纠正RAG结合的策略。实现自：https://arxiv.org/abs/2403.14403
    - 使用本地LLM的版本：[使用本地LLM的自适应RAG](rag/langgraph_adaptive_rag_local.ipynb)
- [纠正式RAG](rag/langgraph_crag.ipynb)：使用LLM对从给定来源检索到的信息进行质量评估，如果质量较低，则尝试从另一个来源检索信息。实现自：https://arxiv.org/pdf/2401.15884.pdf 
    - 使用本地LLM的版本：[使用本地LLM的纠正式RAG](rag/langgraph_crag_local.ipynb)
- [自我RAG](rag/langgraph_self_rag.ipynb)：自我RAG是一种将自我反思/自我评估纳入检索文档和生成的RAG策略。实现自https://arxiv.org/abs/2310.11511。
    - 使用本地LLM的版本：[使用本地LLM的自我RAG](rag/langgraph_self_rag_local.ipynb) 
- [SQL代理](sql-agent.ipynb)：构建一个可以回答SQL数据库问题的SQL代理。

### 代理架构

#### 多代理系统

- [网络](multi_agent/multi-agent-collaboration.ipynb)：使两个或多个代理协作完成任务
- [监督者](multi_agent/agent_supervisor.ipynb)：使用LLM来协调和委派给各个代理
- [分层团队](multi_agent/hierarchical_agent_teams.ipynb)：协调嵌套的代理团队以解决问题
 
#### 规划代理

- [计划与执行](plan-and-execute/plan-and-execute.ipynb)：实现一个基本的规划与执行代理
- [无观察推理](rewoo/rewoo.ipynb)：通过将观察保存为变量来减少重新规划
- [LLM编译器](llm-compiler/LLMCompiler.ipynb)：从规划器中流式传输并急切执行任务的DAG

#### 反思与批评

- [基本反思](reflection/reflection.ipynb)：提示代理反思并修订其输出
- [Reflexion](reflexion/reflexion.ipynb)：批评缺失和多余的细节以指导下一步
- [思维树](tot/tot.ipynb)：使用评分树搜索问题的候选解决方案
- [语言代理树搜索](lats/lats.ipynb)：使用反思和奖励驱动代理的蒙特卡洛树搜索
- [自我发现代理](self-discover/self-discover.ipynb)：分析一个了解自身能力的代理

### 评估

- [基于代理](chatbot-simulation-evaluation/agent-simulation-evaluation.ipynb)：通过模拟用户交互评估聊天机器人
- [在LangSmith中](chatbot-simulation-evaluation/langsmith-agent-simulation-evaluation.ipynb)：在LangSmith中基于对话数据集评估聊天机器人

### 实验性

- [网络研究（STORM）](storm/storm.ipynb)：通过研究和多视角QA生成类似维基百科的文章
- [TNT-LLM](tnt-llm/tnt-llm.ipynb)：构建用户意图的丰富且可解释的分类系统，使用微软为其Bing Copilot应用开发的分类系统。
- [网络导航](web-navigation/web_voyager.ipynb)：构建一个可以导航和与网站交互的代理
- [竞赛编程](usaco/usaco.ipynb)：构建一个具有少量“情景记忆”和人机交互协作的代理，以解决美国计算机奥林匹克竞赛的问题；改编自Shi、Tang、Narasimhan和Yao的论文["Can Language Models Solve Olympiad Programming?"](https://arxiv.org/abs/2404.10952v1)。
- [复杂数据提取](extraction/retries.ipynb)：构建一个可以使用函数调用来完成复杂提取任务的代理

## LangGraph平台 🧱 {#platform}

### 认证与访问控制

在以下三部分指南中，为现有的LangGraph平台部署添加自定义认证和授权：

1. [设置自定义认证](auth/getting_started.md)：实现OAuth2认证以授权用户访问你的部署
2. [资源授权](auth/resource_auth.md)：让用户拥有私人对话
3. [连接认证提供者](auth/add_auth_server.md)：添加真实用户账户并使用OAuth2进行验证