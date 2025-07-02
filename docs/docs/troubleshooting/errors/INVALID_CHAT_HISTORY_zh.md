# INVALID_CHAT_HISTORY

此错误发生在预构建的[create_react_agent][langgraph.prebuilt.chat_agent_executor.create_react_agent]中，当`call_model`图节点接收到格式不正确的消息列表时。具体来说，当存在带有`tool_calls`（LLM请求调用工具）的`AIMessages`却没有对应的`ToolMessage`（返回给LLM的工具调用结果）时，就会出现格式错误。

出现此错误可能有以下几个原因：

1. 在调用图时手动传递了格式错误的消息列表，例如`graph.invoke({'messages': [AIMessage(..., tool_calls=[...])]})`
2. 图在从`tools`节点（即一组ToolMessages）接收更新之前被中断，并且您用非None或非ToolMessage的输入调用了它，例如`graph.invoke({'messages': [HumanMessage(...)]}, config)`。
   这种中断可能是由以下方式之一触发的：
     - 您在`create_react_agent`中手动设置了`interrupt_before = ['tools']`
     - 某个工具抛出了未被[ToolNode][langgraph.prebuilt.tool_node.ToolNode]处理的错误（`"tools"`）

## 故障排除

要解决此问题，您可以采取以下措施之一：

1. 不要用格式错误的消息列表调用图
2. 在发生中断（手动或由于错误）时，您可以：

    - 提供与现有工具调用匹配的ToolMessages，并调用`graph.invoke({'messages': [ToolMessage(...)]})`。
    **注意**：这将把消息附加到历史记录中并从START节点开始运行图。
    - 手动更新状态并从中断处恢复图：

        1. 使用`graph.get_state(config)`从图状态获取最新的消息列表
        2. 修改消息列表，要么从AIMessages中移除未应答的工具调用，要么添加与未应答工具调用匹配的tool_call_ids的ToolMessages
        3. 调用`graph.update_state(config, {'messages': ...})`并传入修改后的消息列表
        4. 恢复图，例如调用`graph.invoke(None, config)`