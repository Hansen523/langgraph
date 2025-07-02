# 无效的图节点返回值错误

一个 LangGraph 的 [`StateGraph`](https://langchain-ai.github.io/langgraph/reference/graphs/#langgraph.graph.state.StateGraph) 从节点收到了非字典类型的返回值。以下是示例:

```python
class State(TypedDict):
    some_key: str

def bad_node(state: State):
    # 应该返回一个包含"some_key"值的字典，而不是列表
    return ["whoops"]

builder = StateGraph(State)
builder.add_node(bad_node)
...

graph = builder.compile()
```

调用上述图将产生如下错误:

```python
graph.invoke({ "some_key": "someval" });
```

```
InvalidUpdateError: 期望字典类型，得到 ['whoops']
如需故障排除，请访问: https://python.langchain.com/docs/troubleshooting/errors/INVALID_GRAPH_NODE_RETURN_VALUE
```

您的图节点必须返回一个包含您定义的状态中一个或多个键的字典。

## 故障排除

以下方法可能有助于解决此错误:

- 如果您的节点中有复杂逻辑，请确保所有代码路径都返回适合您定义状态的字典。