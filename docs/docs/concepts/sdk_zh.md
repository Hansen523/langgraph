# LangGraph SDK

!!! info "前提条件"
    - [LangGraph 平台](./langgraph_platform.md)
    - [LangGraph 服务器](./langgraph_server.md)

LangGraph 平台提供了 Python 和 JS 的 SDK，用于与 [LangGraph 服务器 API](./langgraph_server.md) 进行交互。

## 安装

您可以使用适合您语言的包管理器来安装这些包。

=== "Python"
    ```bash
    pip install langgraph-sdk
    ```

=== "JS"
    ```bash
    yarn add @langchain/langgraph-sdk
    ```


## API 参考

您可以在此处找到 SDK 的 API 参考：

- [Python SDK 参考](../cloud/reference/sdk/python_sdk_ref.md)
- [JS/TS SDK 参考](../cloud/reference/sdk/js_ts_sdk_ref.md)

## Python 同步 vs. 异步

Python SDK 提供了同步 (`get_sync_client`) 和异步 (`get_client`) 客户端来与 LangGraph 服务器 API 进行交互。

=== "异步"
    ```python
    from langgraph_sdk import get_client

    client = get_client(url=..., api_key=...)
    await client.assistants.search()
    ```

=== "同步"

    ```python
    from langgraph_sdk import get_sync_client

    client = get_sync_client(url=..., api_key=...)
    client.assistants.search()
    ```

## 相关

- [LangGraph CLI API 参考](../cloud/reference/cli.md)
- [Python SDK 参考](../cloud/reference/sdk/python_sdk_ref.md)
- [JS/TS SDK 参考](../cloud/reference/sdk/js_ts_sdk_ref.md)