# 如何部署自托管数据平面

在部署前，请先阅读[自托管数据平面概念指南](../../concepts/langgraph_self_hosted_data_plane.md)了解此部署方案。

!!! info "重要提示"
    自托管数据平面部署方案需要[企业版](../../concepts/plans.md)计划。

## 前置条件

1. 使用[LangGraph CLI](../../concepts/langgraph_cli.md)[在本地测试应用](../../tutorials/langgraph-platform/local-server.md)
1. 使用[LangGraph CLI](../../concepts/langgraph_cli.md)构建Docker镜像(即`langgraph build`)并推送至您的Kubernetes集群或Amazon ECS集群可访问的镜像仓库

## Kubernetes部署方案

### 前置条件
1. 集群已安装`KEDA`

        helm repo add kedacore https://kedacore.github.io/charts
        helm install keda kedacore/keda --namespace keda --create-namespace

1. 集群已安装有效的`Ingress`控制器
1. 集群有足够资源运行多个部署，建议启用`Cluster-Autoscaler`自动扩容节点
1. 需开放以下两个控制平面URL的出站连接，监听器会轮询这些端点获取部署信息：

        https://api.host.langchain.com
        https://api.smith.langchain.com

### 设置步骤

1. 提供您的LangSmith组织ID，我们将为您的组织启用自托管数据平面
1. 我们会提供[Helm chart](https://github.com/langchain-ai/helm/tree/main/charts/langgraph-dataplane)，您可通过它配置Kubernetes集群，该chart包含以下核心组件：
    1. `langgraph-listener`：监听LangChain[控制平面](../../concepts/langgraph_control_plane.md)的服务，当您的部署发生变化时会创建/更新下游CRD
    1. `LangGraphPlatform CRD`：用于管理LangGraph Platform部署的定制资源定义
    1. `langgraph-platform-operator`：处理LangGraph Platform CRD变更的操作器
1. 配置`langgraph-dataplane-values.yaml`文件

        config:
          langgraphPlatformLicenseKey: "" # 您的LangGraph Platform许可证密钥
          langsmithApiKey: "" # 工作空间API密钥  
          langsmithWorkspaceId: "" # 工作空间ID
          hostBackendUrl: "https://api.host.langchain.com" # 仅欧盟地区需覆盖此值
          smithBackendUrl: "https://api.smith.langchain.com" # 仅欧盟地区需覆盖此值

1. 部署`langgraph-dataplane` Helm chart

        helm repo add langchain https://langchain-ai.github.io/helm/
        helm repo update
        helm upgrade -i langgraph-dataplane langchain/langgraph-dataplane --values langgraph-dataplane-values.yaml

1. 成功部署后，您将在命名空间中看到两个服务

        NAME                                          READY   STATUS              RESTARTS   AGE
        langgraph-dataplane-listener-7fccd788-wn2dx   0/1     Running             0          9s
        langgraph-dataplane-redis-0                   0/1     ContainerCreating   0          9s

1. 通过[控制平面UI](../../concepts/langgraph_control_plane.md#control-plane-ui)创建部署

## Amazon ECS部署方案

即将推出！