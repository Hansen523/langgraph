---
search:
  boost: 2
tags:
  - agent
hide:
  - tags
---

# 评估

要评估您的智能体性能，您可以使用 `LangSmith` 的[评估功能](https://docs.smith.langchain.com/evaluation)。首先需要定义一个评估函数来评判智能体的结果，比如最终输出或轨迹。根据评估方法的不同，可能涉及参考输出，也可能不涉及：

```python
def evaluator(*, outputs: dict, reference_outputs: dict):
    # 将智能体输出与参考输出进行比较
    output_messages = outputs["messages"]
    reference_messages = reference["messages"]
    score = compare_messages(output_messages, reference_messages)
    return {"key": "evaluator_score", "score": score}
```

为了快速开始，您可以使用 `AgentEvals` 包中预构建的评估器：

```bash
pip install -U agentevals
```

## 创建评估器

评估智能体性能的常见方法之一是将它的轨迹（调用工具的顺序）与参考轨迹进行比较：

```python
import json
# highlight-next-line
from agentevals.trajectory.match import create_trajectory_match_evaluator

outputs = [
    {
        "role": "assistant",
        "tool_calls": [
            {
                "function": {
                    "name": "get_weather",
                    "arguments": json.dumps({"city": "san francisco"}),
                }
            },
            {
                "function": {
                    "name": "get_directions",
                    "arguments": json.dumps({"destination": "presidio"}),
                }
            }
        ],
    }
]
reference_outputs = [
    {
        "role": "assistant",
        "tool_calls": [
            {
                "function": {
                    "name": "get_weather",
                    "arguments": json.dumps({"city": "san francisco"}),
                }
            },
        ],
    }
]

# 创建评估器
evaluator = create_trajectory_match_evaluator(
    # highlight-next-line
    trajectory_match_mode="superset",  # (1)!
)

# 运行评估器
result = evaluator(
    outputs=outputs, reference_outputs=reference_outputs
)
```

1. 指定如何比较轨迹。`superset` 会将输出轨迹视为有效，如果它是参考轨迹的超集。其他选项包括：[严格匹配](https://github.com/langchain-ai/agentevals?tab=readme-ov-file#strict-match)、[无序匹配](https://github.com/langchain-ai/agentevals?tab=readme-ov-file#unordered-match)和[子集匹配](https://github.com/langchain-ai/agentevals?tab=readme-ov-file#subset-and-superset-match)。

下一步，了解更多关于如何[自定义轨迹匹配评估器](https://github.com/langchain-ai/agentevals?tab=readme-ov-file#agent-trajectory-match)。

### 使用LLM作为评判者

您可以使用LLM作为评判者的评估器，它会利用LLM将轨迹与参考输出进行比较并输出评分：

```python
import json
from agentevals.trajectory.llm import (
    # highlight-next-line
    create_trajectory_llm_as_judge,
    TRAJECTORY_ACCURACY_PROMPT_WITH_REFERENCE
)

evaluator = create_trajectory_llm_as_judge(
    prompt=TRAJECTORY_ACCURACY_PROMPT_WITH_REFERENCE,
    model="openai:o3-mini"
)
```

## 运行评估器

要运行评估器，首先需要创建一个[LangSmith数据集](https://docs.smith.langchain.com/evaluation/concepts#datasets)。使用预构建的AgentEvals评估器时，数据集需要遵循以下架构：

- **输入**: `{"messages": [...]}` 调用智能体的输入消息。
- **输出**: `{"messages": [...]}` 智能体输出的预期消息历史。对于轨迹评估，可以仅保留助手的消息。

```python
from langsmith import Client
from langgraph.prebuilt import create_react_agent
from agentevals.trajectory.match import create_trajectory_match_evaluator

client = Client()
agent = create_react_agent(...)
evaluator = create_trajectory_match_evaluator(...)

experiment_results = client.evaluate(
    lambda inputs: agent.invoke(inputs),
    # 替换为您的数据集名称
    data="<您的数据集名称>",
    evaluators=[evaluator]
)
```