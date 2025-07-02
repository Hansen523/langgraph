# 如何部署自托管控制平面

在部署之前，请仔细阅读[自托管控制平面概念指南](../../concepts/langgraph_self_hosted_control_plane.md)部署选项。

!!! info "重要提示"
    自托管控制平面部署选项需要[企业版](../../concepts/plans.md)计划。

## 先决条件

1. 您正在使用Kubernetes。
1. 您已部署自托管LangSmith。
1. 使用[LangGraph CLI](../../concepts/langgraph_cli.md)[在本地测试您的应用程序](../../tutorials/langgraph-platform/local-server.md)。
1. 使用[LangGraph CLI](../../concepts/langgraph_cli.md)构建Docker镜像（即`langgraph build`）并将其推送到您的Kubernetes集群可以访问的注册表。
1. 集群上已安装`KEDA`。

         helm repo add kedacore https://kedacore.github.io/charts 
         helm install keda kedacore/keda --namespace keda --create-namespace
1. 入口配置
    1. 您必须为LangSmith实例设置入口。所有代理都将作为Kubernetes服务部署在此入口后面。
    1. 您可以使用此指南为您的实例[设置入口](https://docs.smith.langchain.com/self_hosting/configuration/ingress)。
1. 您的集群中有足够的空间进行多次部署。推荐使用`Cluster-Autoscaler`来自动配置新节点。
1. 有效的Dynamic PV provisioner或集群上有可用的PV。您可以通过运行以下命令验证：

        kubectl get storageclass

## 设置

1. 在配置自托管LangSmith实例时，启用`langgraphPlatform`选项。这将配置一些关键资源。
    1. `listener`：这是一个监听[控制平面](../../concepts/langgraph_control_plane.md)变化的服务，用于创建/更新下游CRD。
    1. `LangGraphPlatform CRD`：用于LangGraph平台部署的CRD。这包含管理LangGraph平台部署实例的规格。
    1. `operator`：此操作员处理LangGraph Platform CRD的变化。
    1. `host-backend`：这是[控制平面](../../concepts/langgraph_control_plane.md)。
1. 图表将使用两个额外的镜像。使用最新版本中指定的镜像。

        hostBackendImage:
          repository: "docker.io/langchain/hosted-langserve-backend"
          pullPolicy: IfNotPresent
        operatorImage:
          repository: "docker.io/langchain/langgraph-operator"
          pullPolicy: IfNotPresent

1. 在您的LangSmith配置文件中（通常为`langsmith_config.yaml`），启用`langgraphPlatform`选项。请注意，您还必须有一个有效的入口设置：

        config:
          langgraphPlatform:
            enabled: true
            langgraphPlatformLicenseKey: "YOUR_LANGGRAPH_PLATFORM_LICENSE_KEY"
1. 在您的`values.yaml`文件中，配置`hostBackendImage`和`operatorImage`选项（如果需要镜像镜像）

1. 您也可以通过覆盖[此处](https://github.com/langchain-ai/helm/blob/main/charts/langsmith/values.yaml#L898)的基本模板来配置代理的基本模板。
1. 您可以从[控制平面UI](../../concepts/langgraph_control_plane.md#control-plane-ui)创建部署。