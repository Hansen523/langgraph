# Webhooks

Webhooks（网络钩子）可实现从您的LangGraph平台应用程序到外部服务的事件驱动通信。例如，当对LangGraph平台的API调用完成运行时，您可能希望向其他服务发送更新通知。

许多LangGraph平台端点都接受`webhook`参数。如果该参数由能够接受POST请求的端点指定，LangGraph平台将在运行完成时发送请求。

更多详细信息，请参阅对应的[操作指南](../../cloud/how-tos/webhooks.md)。