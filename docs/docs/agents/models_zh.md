# 模型

LangGraph通过LangChain库提供了对[LLMs(语言模型)](https://python.langchain.com/docs/concepts/chat_models/)的内置支持。这使得将各种LLM集成到您的代理和工作流程中变得非常简单。

## 初始化模型

使用[`init_chat_model`](https://python.langchain.com/docs/how_to/chat_models_universal_init/)来初始化模型：

{!snippets/chat_model_tabs.md!}

### 直接实例化模型

如果模型提供商不支持`init_chat_model`，您可以直接实例化该提供商的模型类。该模型必须实现[BaseChatModel接口](https://python.langchain.com/api_reference/core/language_models/langchain_core.language_models.chat_models.BaseChatModel.html)并支持工具调用：

```python
# Anthropic已经通过`init_chat_model`支持，
# 但您也可以直接实例化它。
from langchain_anthropic import ChatAnthropic

model = ChatAnthropic(
  model="claude-3-7-sonnet-latest",
  temperature=0,
  max_tokens=2048
)
```

!!! important "工具调用支持"

    如果您正在构建需要模型调用外部工具的代理或工作流，请确保底层语言模型支持[工具调用](../concepts/tools.md)。兼容的模型可以在[LangChain集成目录](https://python.langchain.com/docs/integrations/chat/)中找到。

## 在代理中使用

当使用`create_react_agent`时，您可以通过其名称字符串指定模型，这是使用`init_chat_model`初始化模型的快捷方式。这允许您使用模型而无需直接导入或实例化它。

=== "模型名称"

      ```python
      from langgraph.prebuilt import create_react_agent

      create_react_agent(
         # highlight-next-line
         model="anthropic:claude-3-7-sonnet-latest",
         # 其他参数
      )
      ```

=== "模型实例"

      ```python
      from langchain_anthropic import ChatAnthropic
      from langgraph.prebuilt import create_react_agent

      model = ChatAnthropic(
          model="claude-3-7-sonnet-latest",
          temperature=0,
          max_tokens=2048
      )
      # 或者
      # model = init_chat_model("anthropic:claude-3-7-sonnet-latest")

      agent = create_react_agent(
        # highlight-next-line
        model=model,
        # 其他参数
      )
      ```

## 高级模型配置

### 禁用流式传输

要禁用单个LLM令牌的流式传输，在初始化模型时设置`disable_streaming=True`：

=== "`init_chat_model`"

    ```python
    from langchain.chat_models import init_chat_model

    model = init_chat_model(
        "anthropic:claude-3-7-sonnet-latest",
        # highlight-next-line
        disable_streaming=True
    )
    ```

=== "`ChatModel`"

    ```python
    from langchain_anthropic import ChatAnthropic

    model = ChatAnthropic(
        model="claude-3-7-sonnet-latest",
        # highlight-next-line
        disable_streaming=True
    )
    ```

参考[API参考](https://python.langchain.com/api_reference/core/language_models/langchain_core.language_models.chat_models.BaseChatModel.html#langchain_core.language_models.chat_models.BaseChatModel.disable_streaming)获取更多关于`disable_streaming`的信息

### 添加模型回退

您可以使用`model.with_fallbacks([...])`添加对不同模型或不同LLM提供商的回退：

=== "`init_chat_model`"

    ```python
    from langchain.chat_models import init_chat_model

    model_with_fallbacks = (
        init_chat_model("anthropic:claude-3-5-haiku-latest")
        # highlight-next-line
        .with_fallbacks([
            init_chat_model("openai:gpt-4.1-mini"),
        ])
    )
    ```

=== "`ChatModel`"

    ```python
    from langchain_anthropic import ChatAnthropic
    from langchain_openai import ChatOpenAI

    model_with_fallbacks = (
        ChatAnthropic(model="claude-3-5-haiku-latest")
        # highlight-next-line
        .with_fallbacks([
            ChatOpenAI(model="gpt-4.1-mini"),
        ])
    )
    ```

查看此[指南](https://python.langchain.com/docs/how_to/fallbacks/#fallback-to-better-model)获取更多关于模型回退的信息。

### 使用内置的速率限制器

Langchain包含一个内置的内存速率限制器。此速率限制器是线程安全的，可以在同一进程的多个线程中共享。

```python
from langchain_core.rate_limiters import InMemoryRateLimiter
from langchain_anthropic import ChatAnthropic

rate_limiter = InMemoryRateLimiter(
    requests_per_second=0.1,  # <-- 非常慢！我们每10秒只能发出一个请求！！
    check_every_n_seconds=0.1,  # 每100毫秒醒来检查是否允许发出请求，
    max_bucket_size=10,  # 控制最大突发大小。
)

model = ChatAnthropic(
   model_name="claude-3-opus-20240229", 
   rate_limiter=rate_limiter
)
```

查看LangChain文档获取更多关于如何[处理速率限制](https://python.langchain.com/docs/how_to/chat_model_rate_limiting/)的信息。

## 自带模型

如果您所需的LLM未被LangChain官方支持，请考虑以下选项：

1. **实现自定义LangChain聊天模型**：创建一个符合[LangChain聊天模型接口](https://python.langchain.com/docs/how_to/custom_chat_model/)的模型。这可以实现与LangGraph的代理和工作流的完全兼容，但需要理解LangChain框架。

2. **带有自定义流式传输的直接调用**：通过[添加自定义流式逻辑](../how-tos/streaming.md#use-with-any-llm)与`StreamWriter`直接使用您的模型。
   参考[自定义流式传输文档](../how-tos/streaming.md#use-with-any-llm)获取指导。这种方法适用于不需要预构建代理集成的自定义工作流。

## 其他资源

- [多模态输入](https://python.langchain.com/docs/how_to/multimodal_inputs/)
- [结构化输出](https://python.langchain.com/docs/how_to/structured_output/)
- [模型集成目录](https://python.langchain.com/docs/integrations/chat/)
- [强制模型调用特定工具](https://python.langchain.com/docs/how_to/tool_choice/)
- [所有聊天模型操作指南](https://python.langchain.com/docs/how_to/#chat-models)
- [聊天模型集成](https://python.langchain.com/docs/integrations/chat/)