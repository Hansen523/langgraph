## 组件模块

LangGraph平台由一系列协同工作的核心组件构成，这些组件共同支持LangGraph应用的开发、部署、调试与监控：

- [LangGraph服务器](./langgraph_server.md)：该服务器定义了融合智能体应用最佳实践的API架构，使开发者能专注于智能体逻辑构建而非底层服务搭建
- [LangGraph命令行工具](./langgraph_cli.md)：提供与本地LangGraph环境交互的命令行接口
- [LangGraph集成开发环境](./langgraph_studio.md)：专用IDE开发环境，可连接LangGraph服务器实现可视化应用交互与本地调试
- [Python/JS开发套件](./sdk.md)：提供与已部署LangGraph应用进行编程交互的SDK工具包
- [远程图执行](../how-tos/use-remote-graph.md)：RemoteGraph功能允许开发者像操作本地应用一样与任何已部署的LangGraph应用交互
- [LangGraph控制平面](./langgraph_control_plane.md)：包含创建/更新LangGraph服务器的控制台界面及支撑该界面的API体系
- [LangGraph数据平面](./langgraph_data_plane.md)：涵盖LangGraph服务器集群、对应基础设施及持续监听控制平面更新的"listener"应用

![LangGraph组件架构图](img/lg_platform.png)