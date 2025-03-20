# 如何为您的LangGraph部署添加语义搜索

本指南解释了如何为您的LangGraph部署的跨线程[存储](../../concepts/persistence.md#memory-store)添加语义搜索，以便您的代理可以通过语义相似性搜索记忆和其他文档。

## 先决条件

- 一个LangGraph部署（参见[如何部署](setup_pyproject.md)）
- 您的嵌入提供商的API密钥（在本例中为OpenAI）
- `langchain >= 0.3.8`（如果您使用下面的字符串格式指定）

## 步骤

1. 更新您的`langgraph.json`配置文件以包含存储配置：

```json
{
    ...
    "store": {
        "index": {
            "embed": "openai:text-embedding-3-small",
            "dims": 1536,
            "fields": ["$"]
        }
    }
}
```

此配置：

- 使用OpenAI的text-embedding-3-small模型生成嵌入
- 将嵌入维度设置为1536（与模型的输出匹配）
- 索引存储数据中的所有字段（`["$"]`表示索引所有内容，或指定特定字段如`["text", "metadata.title"]`）

2. 要使用上述字符串嵌入格式，请确保您的依赖项包括`langchain >= 0.3.8`：

```toml
# 在pyproject.toml中
[project]
dependencies = [
    "langchain>=0.3.8"
]
```

或者如果使用requirements.txt：

```
langchain>=0.3.8
```

## 使用

配置完成后，您可以在LangGraph节点中使用语义搜索。存储需要一个命名空间元组来组织记忆：

```python
def search_memory(state: State, *, store: BaseStore):
    # 使用语义相似性搜索存储
    # 命名空间元组有助于组织不同类型的记忆
    # 例如，("user_facts", "preferences") 或 ("conversation", "summaries")
    results = store.search(
        namespace=("memory", "facts"),  # 按类型组织记忆
        query="您的搜索查询",
        limit=3  # 返回的结果数量
    )
    return results
```

## 自定义嵌入

如果您想使用自定义嵌入，可以传递自定义嵌入函数的路径：

```json
{
    ...
    "store": {
        "index": {
            "embed": "path/to/embedding_function.py:embed",
            "dims": 1536,
            "fields": ["$"]
        }
    }
}
```

部署将在指定路径中查找该函数。该函数必须是异步的并接受字符串列表：

```python
# path/to/embedding_function.py
from openai import AsyncOpenAI

client = AsyncOpenAI()

async def aembed_texts(texts: list[str]) -> list[list[float]]:
    """自定义嵌入函数必须：
    1. 是异步的
    2. 接受字符串列表
    3. 返回浮点数组列表（嵌入）
    """
    response = await client.embeddings.create(
        model="text-embedding-3-small",
        input=texts
    )
    return [e.embedding for e in response.data]
```

## 通过API查询

您还可以使用LangGraph SDK查询存储。由于SDK使用异步操作：

```python
from langgraph_sdk import get_client

async def search_store():
    client = get_client()
    results = await client.store.search_items(
        ("memory", "facts"),
        query="您的搜索查询",
        limit=3  # 返回的结果数量
    )
    return results

# 在异步上下文中使用
results = await search_store()
```