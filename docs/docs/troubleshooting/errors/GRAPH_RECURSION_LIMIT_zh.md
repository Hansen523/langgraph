# GRAPH_RECURSION_LIMIT

您的LangGraph [`StateGraph`](https://langchain-ai.github.io/langgraph/reference/graphs/#langgraph.graph.state.StateGraph)在达到停止条件前已触达最大步数限制。
这通常由类似下例的代码引起的无限循环导致：

```python
class State(TypedDict):
    some_key: str

builder = StateGraph(State)
builder.add_node("a", ...)
builder.add_node("b", ...)
builder.add_edge("a", "b")
builder.add_edge("b", "a")
...

graph = builder.compile()
```

但复杂图结构也可能自然触发默认限制。

## 故障排查

- 若您的图预期不应经历多次迭代，很可能存在循环结构。请检查逻辑中是否有无限循环。
- 若您构建的是复杂图结构，可在调用图时通过`config`对象传入更高的`recursion_limit`值：

```python
graph.invoke({...}, {"recursion_limit": 100})
```