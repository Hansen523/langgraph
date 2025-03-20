# 流式传输

为终端用户构建响应式应用？实时更新是保持用户参与度的关键，尤其是在应用运行过程中。

有三种主要类型的数据你可能希望进行流式传输：

1. 工作流进度（例如，在每个图节点执行后获取状态更新）。
2. 生成的大语言模型（LLM）令牌。
3. 自定义更新（例如，“已获取10/100条记录”）。

## 流式传输图输出（`.stream`和`.astream`）

`.stream`和`.astream`是用于从图运行中流式传输输出的同步和异步方法。在调用这些方法时，你可以指定几种不同的模式（例如`graph.stream(..., mode="...")`）：

- [`"values"`](../how-tos/streaming.ipynb#values)：在图的每一步之后流式传输状态的完整值。
- [`"updates"`](../how-tos/streaming.ipynb#updates)：在图的每一步之后流式传输状态的更新。如果在同一步骤中进行了多次更新（例如运行了多个节点），则这些更新将分别流式传输。
- [`"custom"`](../how-tos/streaming.ipynb#custom)：从图节点内部流式传输自定义数据。
- [`"messages"`](../how-tos/streaming-tokens.ipynb)：流式传输LLM令牌以及调用LLM的图节点的元数据。
- [`"debug"`](../how-tos/streaming.ipynb#debug)：在整个图执行过程中流式传输尽可能多的信息。

你还可以通过将它们作为列表传递来同时指定多个流式传输模式。当你这样做时，流式传输的输出将是元组`(stream_mode, data)`。例如：

```python
graph.stream(..., stream_mode=["updates", "messages"])
```

```
...
('messages', (AIMessageChunk(content='Hi'), {'langgraph_step': 3, 'langgraph_node': 'agent', ...}))
...
('updates', {'agent': {'messages': [AIMessage(content="Hi, how can I help you?")]}})
```

下面的可视化展示了`values`和`updates`模式之间的区别：

![values vs updates](../static/values_vs_updates.png)

## LangGraph平台

流式传输对于使LLM应用对终端用户具有响应性至关重要。在创建流式运行时，流式模式决定了哪些数据会被流式传输回API客户端。LangGraph平台支持五种流式模式：

- `values`：在每次[超步](https://langchain-ai.github.io/langgraph/concepts/low_level/#graphs)执行后流式传输图的完整状态。请参阅流式传输值的[操作指南](../cloud/how-tos/stream_values.md)。
- `messages-tuple`：流式传输节点内部生成的任何消息的LLM令牌。此模式主要用于支持聊天应用。请参阅流式传输消息的[操作指南](../cloud/how-tos/stream_messages.md)。
- `updates`：在每个节点执行后流式传输图状态的更新。请参阅流式传输更新的[操作指南](../cloud/how-tos/stream_updates.md)。
- `debug`：在整个图执行过程中流式传输调试事件。请参阅流式传输调试事件的[操作指南](../cloud/how-tos/stream_debug.md)。
- `events`：流式传输图执行过程中发生的所有事件（包括图的状态）。请参阅流式传输事件的[操作指南](../cloud/how-tos/stream_events.md)。此模式仅对将大型LCEL应用迁移到LangGraph的用户有用。通常，对于大多数应用来说，此模式并不必要。

你还可以同时指定多个流式模式。请参阅同时配置多个流式模式的[操作指南](../cloud/how-tos/stream_multiple.md)。

有关如何创建流式运行的详细信息，请参阅[API参考](../cloud/reference/api/api_ref.html#tag/threads-runs/POST/threads/{thread_id}/runs/stream)。

流式模式`values`、`updates`、`messages-tuple`和`debug`与LangGraph库中可用的模式非常相似——关于这些模式的更深入的概念解释，请参阅[上一节](#streaming-graph-outputs-stream-and-astream)。

流式模式`events`与在LangGraph库中使用`.astream_events`相同——关于此模式的更深入的概念解释，请参阅[上一节](#streaming-graph-outputs-stream-and-astream)。

所有发出的事件都有两个属性：

- `event`：事件的名称
- `data`：与事件相关的数据