# 调试LangSmith追踪记录

本指南说明如何在LangGraph Studio中打开LangSmith追踪记录进行交互式调查和调试。

## 打开已部署的执行线程

1. 打开LangSmith追踪记录，选择根运行项。
2. 点击"在Studio中运行"。

这将打开与关联LangGraph平台部署相连的LangGraph Studio，并自动选中该追踪记录的父线程。

## 使用远程追踪记录测试本地代理

本节介绍如何用LangSmith的远程追踪记录测试本地代理。这使您能将生产环境追踪记录作为本地测试的输入，方便在开发环境中调试和验证代理修改。

### 前提条件

- 一条LangSmith追踪的线程记录
- 本地运行的代理服务。设置说明参见[这里](../how-tos/studio/quick_start.md#local-development-server)

!!! info "本地代理要求"

    - langgraph>=0.3.18
    - langgraph-api>=0.0.32
    - 需包含远程追踪记录中存在的相同节点集

### 克隆执行线程

1. 打开LangSmith追踪记录，选择根运行项。
2. 点击"在Studio中运行"旁边的下拉菜单。
3. 输入您本地代理的URL地址。
4. 选择"本地克隆线程"。
5. 如果存在多个图，请选择目标图。

将在您的本地代理中创建一个新线程，其执行历史是从远程线程推断并复制而来，同时您将被导航到本地运行应用的LangGraph Studio界面。