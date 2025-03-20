# 如何部署到LangGraph云

LangGraph云可在<a href="https://www.langchain.com/langsmith" target="_blank">LangSmith</a>中使用。要部署LangGraph云API，请导航到<a href="https://smith.langchain.com/" target="_blank">LangSmith UI</a>。

## 先决条件

1. LangGraph云应用程序从GitHub仓库部署。配置并上传LangGraph云应用程序到GitHub仓库以便部署到LangGraph云。
1. [验证LangGraph API在本地运行](test_locally.md)。如果API无法成功运行（即`langgraph dev`），部署到LangGraph云也会失败。

## 创建新部署

从<a href="https://smith.langchain.com/" target="_blank">LangSmith UI</a>开始...

1. 在左侧导航面板中，选择`LangGraph Platform`。`LangGraph Platform`视图包含现有LangGraph云部署的列表。
1. 在右上角，选择`+ New Deployment`以创建新部署。
1. 在`Create New Deployment`面板中，填写必填字段。
    1. `Deployment details`
        1. 选择`Import from GitHub`并按照GitHub OAuth工作流程安装并授权LangChain的`hosted-langserve` GitHub应用程序以访问选定的仓库。安装完成后，返回`Create New Deployment`面板并从下拉菜单中选择要部署的GitHub仓库。**注意**：安装LangChain的`hosted-langserve` GitHub应用程序的GitHub用户必须是组织或账户的[所有者](https://docs.github.com/en/organizations/managing-peoples-access-to-your-organization-with-roles/roles-in-an-organization#organization-owners)。
        1. 指定部署的名称。
        1. 指定所需的`Git Branch`。部署与分支相关联。当创建新修订时，将部署链接分支的代码。分支可以在[部署设置](#deployment-settings)中稍后更新。
        1. 指定[LangGraph API配置文件](../reference/cli.md#configuration-file)的完整路径，包括文件名。例如，如果文件`langgraph.json`位于仓库的根目录中，只需指定`langgraph.json`。
        1. 勾选/取消勾选复选框以`Automatically update deployment on push to branch`。如果勾选，当推送到指定的`Git Branch`时，部署将自动更新。此设置可以在[部署设置](#deployment-settings)中稍后启用/禁用。
    1. 选择所需的`Deployment Type`。
        1. `Development`部署适用于非生产用例，并配备最少的资源。
        1. `Production`部署可以处理多达500个请求/秒，并配备高可用存储和自动备份。
    1. 确定部署是否应`Shareable through LangGraph Studio`。
        1. 如果未勾选，部署将仅对具有有效LangSmith API密钥的工作区可访问。
        1. 如果勾选，部署将通过LangGraph Studio对任何LangSmith用户可访问。将提供直接URL到LangGraph Studio以与其他LangSmith用户共享。
    1. 指定`Environment Variables`和秘密。请参阅[环境变量参考](../reference/env_var.md)以配置部署的其他变量。
        1. 敏感值（如API密钥，例如`OPENAI_API_KEY`）应指定为秘密。
        1. 也可以指定其他非秘密环境变量。
    1. 自动创建一个新的LangSmith `Tracing Project`，其名称与部署相同。
1. 在右上角，选择`Submit`。几秒钟后，`Deployment`视图将出现，新部署将排队等待配置。

## 创建新修订

当[创建新部署](#create-new-deployment)时，默认会创建新修订。可以创建后续修订以部署新的代码更改。

从<a href="https://smith.langchain.com/" target="_blank">LangSmith UI</a>开始...

1. 在左侧导航面板中，选择`LangGraph Platform`。`LangGraph Platform`视图包含现有LangGraph云部署的列表。
1. 选择一个现有部署以创建新修订。
1. 在`Deployment`视图中，在右上角，选择`+ New Revision`。
1. 在`New Revision`模态框中，填写必填字段。
    1. 指定[LangGraph API配置文件](../reference/cli.md#configuration-file)的完整路径，包括文件名。例如，如果文件`langgraph.json`位于仓库的根目录中，只需指定`langgraph.json`。
    1. 确定部署是否应`Shareable through LangGraph Studio`。
        1. 如果未勾选，部署将仅对具有有效LangSmith API密钥的工作区可访问。
        1. 如果勾选，部署将通过LangGraph Studio对任何LangSmith用户可访问。将提供直接URL到LangGraph Studio以与其他LangSmith用户共享。
    1. 指定`Environment Variables`和秘密。现有秘密和环境变量已预先填充。请参阅[环境变量参考](../reference/env_var.md)以配置修订的其他变量。
        1. 添加新秘密或环境变量。
        1. 删除现有秘密或环境变量。
        1. 更新现有秘密或环境变量的值。
1. 选择`Submit`。几秒钟后，`New Revision`模态框将关闭，新修订将排队等待部署。

## 查看构建和服务器日志

每个修订的构建和服务器日志都可用。

从`LangGraph Platform`视图开始...

1. 从`Revisions`表中选择所需的修订。右侧将滑出一个面板，默认选择`Build`选项卡，显示修订的构建日志。
1. 在面板中，选择`Server`选项卡以查看修订的服务器日志。服务器日志仅在修订部署后可用。
1. 在`Server`选项卡中，根据需要调整日期/时间范围选择器。默认情况下，日期/时间范围选择器设置为`Last 7 days`。

## 中断修订

中断修订将停止修订的部署。

!!! warning "未定义行为"
    中断的修订具有未定义的行为。这仅在您需要部署新修订并且已经有一个修订“卡住”时有用。将来，此功能可能会被移除。

从`LangGraph Platform`视图开始...

1. 从`Revisions`表中选择所需修订行右侧的菜单图标（三个点）。
1. 从菜单中选择`Interrupt`。
1. 将出现一个模态框。查看确认消息。选择`Interrupt revision`。

## 删除部署

从<a href="https://smith.langchain.com/" target="_blank">LangSmith UI</a>开始...

1. 在左侧导航面板中，选择`LangGraph Platform`。`LangGraph Platform`视图包含现有LangGraph云部署的列表。
1. 选择所需部署行右侧的菜单图标（三个点）并选择`Delete`。
1. 将出现一个`Confirmation`模态框。选择`Delete`。

## 部署设置

从`LangGraph Platform`视图开始...

1. 在右上角，选择齿轮图标（`Deployment Settings`）。
1. 更新`Git Branch`到所需分支。
1. 勾选/取消勾选复选框以`Automatically update deployment on push to branch`。
    1. 分支创建/删除和标签创建/删除事件不会触发更新。只有推送到现有分支才会触发更新。
    1. 快速连续推送到分支不会触发后续更新。将来，此功能可能会更改/改进。

## 添加或删除GitHub仓库

安装并授权LangChain的`hosted-langserve` GitHub应用程序后，可以修改应用程序的仓库访问权限以添加新仓库或删除现有仓库。如果创建了新仓库，可能需要显式添加。

1. 从GitHub个人资料中，导航到`Settings` > `Applications` > `hosted-langserve` > 点击`Configure`。
1. 在`Repository access`下，选择`All repositories`或`Only select repositories`。如果选择`Only select repositories`，则必须显式添加新仓库。
1. 点击`Save`。
1. 创建新部署时，下拉菜单中的GitHub仓库列表将更新以反映仓库访问权限的更改。

## 白名单IP地址

所有来自2025年1月6日之后创建的`LangGraph Platform`部署的流量将通过NAT网关。
此NAT网关将根据您部署的区域有几个静态IP地址。请参考下表以获取要列入白名单的IP地址列表：

| 美国             | 欧洲             |
|----------------|----------------|
| 35.197.29.146  | 34.13.192.67   |
| 34.145.102.123 | 34.147.105.64  |
| 34.169.45.153  | 34.90.22.166   |
| 34.82.222.17   | 34.147.36.213  |
| 35.227.171.135 | 34.32.137.113  | 
| 34.169.88.30   | 34.91.238.184  |
| 34.19.93.202   | 35.204.101.241 |
| 34.19.34.50    | 35.204.48.32   |