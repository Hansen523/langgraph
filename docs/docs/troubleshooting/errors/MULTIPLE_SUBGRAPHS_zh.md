# 多重子图调用错误

您在当前LangGraph节点中启用了多次子图调用，且每个子图都开启了检查点功能。

由于子图检查点命名空间的内部限制，目前不支持这种操作方式。

## 故障排除

以下建议可能帮助解决该问题：

- 如果不需要中断/从子图恢复运行，请在编译时关闭检查点功能：`.compile(checkpointer=False)`
- 避免在同一节点中多次调用子图，改用[`Send`](https://langchain-ai.github.io/langgraph/concepts/low_level/#send) API实现

（注：保持原文中的代码格式和内链标记不变，仅翻译说明性文字内容）