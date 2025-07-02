# 无效的并发图状态更新（INVALID_CONCURRENT_GRAPH_UPDATE）

当LangGraph的[`StateGraph`](https://langchain-ai.github.io/langgraph/reference/graphs/#langgraph.graph.state.StateGraph)接收到来自多个节点对同一状态属性的并发更新时，而该属性不支持并发操作，就会发生此错误。

这种情况可能发生在以下场景：您在图中使用了[扇出结构](https://langchain-ai.github.io/langgraph/how-tos/map-reduce/)或其他并行执行逻辑，并且定义了如下所示的图：

```python hl_lines="2"
class State(TypedDict):
    some_key: str

def node(state: State):
    return {"some_key": "some_string_value"}

def other_node(state: State):
    return {"some_key": "some_string_value"}


builder = StateGraph(State)
builder.add_node(node)
builder.add_node(other_node)
builder.add_edge(START, "node")
builder.add_edge(START, "other_node")
graph = builder.compile()
```

如果上述图中的某个节点返回`{ "some_key": "some_string_value" }`，这将用`"some_string_value"`覆盖`"some_key"`的状态值。然而，如果多个节点（例如在单个步骤的扇出结构中）同时返回`"some_key"`的值，图将抛出此错误，因为不确定如何更新内部状态。

要解决这个问题，您可以定义一个组合多个值的归约器：

```python hl_lines="5-6"
import operator
from typing import Annotated

class State(TypedDict):
    # operator.add归约函数使该属性变为只追加模式
    some_key: Annotated[list, operator.add]
```

这将允许您定义处理并行执行的多个节点返回相同键值的逻辑。

## 故障排除

以下方法可能有助于解决此错误：

- 如果您的图需要并行执行节点，请确保为相关状态键定义了归约器