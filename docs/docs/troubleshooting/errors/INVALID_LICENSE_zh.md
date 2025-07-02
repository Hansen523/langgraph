# 无效许可证

当尝试启动自托管LangGraph平台服务器时，若许可证验证失败则会引发此错误。该错误专属于LangGraph平台，与开源库无关。

## 触发场景

在运行自托管LangGraph平台部署时，若未持有有效的企业许可证或API密钥，将出现此错误。

## 排查指南

### 确认部署类型

首先需明确目标部署模式。

#### 本地开发环境

若仅需本地开发，可运行`langgraph dev`启用轻量级内存服务器。详见[本地服务器](../../tutorials/langgraph-platform/local-server.md)文档。

#### 托管型LangGraph平台

如需快速获得托管环境，建议采用[云SaaS](../../concepts/langgraph_cloud.md)部署方案。该选项无需额外许可证密钥。

#### 独立容器版（轻量级）

若年节点执行量预计不超过100万次且无需企业级功能（如定时任务等），建议选择[独立容器](../../concepts/deployment_options.md)部署方案。

通过设置有效的`LANGSMITH_API_KEY`环境变量（如`langgraph.json`引用的`.env`文件中）并构建Docker镜像即可部署。该API密钥必须关联**Plus**及以上等级账户。

#### 独立容器版（企业级）

完整自托管需设置`LANGGRAPH_CLOUD_LICENSE_KEY`环境变量。如需获取企业许可证密钥，请联系LangChain技术支持团队。

更多部署选项及功能对比，请参阅[部署选项](../../concepts/deployment_options.md)文档。

### 验证凭证

若已确认需自托管LangGraph平台，请校验凭证有效性。

#### 独立容器版（轻量级）

1. 确保在部署环境或`.env`文件中配置了有效的`LANGSMITH_API_KEY`
2. 确认提供的API密钥关联**Plus**或**Enterprise**等级（或等同）账户

#### 独立容器版（企业级）

1. 确保在部署环境或`.env`文件中配置了有效的`LANGGRAPH_CLOUD_LICENSE_KEY`
2. 验证密钥仍处有效期内且未过期