# 使用Server API实现人机交互

要在智能体或工作流中审查、编辑和批准工具调用，请使用LangGraph的[人机交互](../../concepts/human_in_the_loop.md)功能。

## LangGraph API调用与恢复

=== "Python"

    ```python
    from langgraph_sdk import get_client
    # highlight-next-line
    from langgraph_sdk.schema import Command
    client = get_client(url=<DEPLOYMENT_URL>)

    # 使用名为"agent"的部署图
    assistant_id = "agent"

    # 创建线程
    thread = await client.threads.create()
    thread_id = thread["thread_id"]

    # 运行图直到遇到中断
    result = await client.runs.wait(
        thread_id,
        assistant_id,
        input={"some_text": "original text"}   # (1)!
    )

    print(result['__interrupt__']) # (2)!
    # > [
    # >     {
    # >         'value': {'text_to_revise': 'original text'},
    # >         'resumable': True,
    # >         'ns': ['human_node:fc722478-2f21-0578-c572-d9fc4dd07c3b'],
    # >         'when': 'during'
    # >     }
    # > ]


    # 恢复图执行
    print(await client.runs.wait(
        thread_id,
        assistant_id,
        # highlight-next-line
        command=Command(resume="Edited text")   # (3)!
    ))
    # > {'some_text': 'Edited text'}
    ```

    1. 图以初始状态启动
    2. 当图遇到中断时，返回包含有效载荷和元数据的中断对象
    3. 使用`Command(resume=...)`恢复图执行，注入人工输入并继续执行

=== "JavaScript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";
    const client = new Client({ apiUrl: <DEPLOYMENT_URL> });

    // 使用名为"agent"的部署图
    const assistantID = "agent";

    // 创建线程
    const thread = await client.threads.create();
    const threadID = thread["thread_id"];

    // 运行图直到遇到中断
    const result = await client.runs.wait(
      threadID,
      assistantID,
      { input: { "some_text": "original text" } }   // (1)!
    );

    console.log(result['__interrupt__']); // (2)!
    // > [
    // >     {
    // >         'value': {'text_to_revise': 'original text'},
    // >         'resumable': True,
    // >         'ns': ['human_node:fc722478-2f21-0578-c572-d9fc4dd07c3b'],
    // >         'when': 'during'
    // >     }
    // > ]

    // 恢复图执行
    console.log(await client.runs.wait(
        threadID,
        assistantID,
        // highlight-next-line
        { command: { resume: "Edited text" }}   // (3)!
    ));
    // > {'some_text': 'Edited text'}
    ```

    1. 图以初始状态启动
    2. 当图遇到中断时，返回包含有效载荷和元数据的中断对象
    3. 使用`{ resume: ... }`命令对象恢复图执行，注入人工输入并继续执行

=== "cURL"

    创建线程:

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads \
    --header 'Content-Type: application/json' \
    --data '{}'
    ```

    运行图直到遇到中断:

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/wait \
    --header 'Content-Type: application/json' \
    --data "{
      \"assistant_id\": \"agent\",
      \"input\": {\"some_text\": \"original text\"}
    }"
    ```

    恢复图执行:

    ```bash
    curl --request POST \
     --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/wait \
     --header 'Content-Type: application/json' \
     --data "{
       \"assistant_id\": \"agent\",
       \"command\": {
         \"resume\": \"Edited text\"
       }
     }"
    ```

??? example "扩展示例：使用`interrupt`"

    这是一个可在LangGraph API服务器上运行的图示例。
    更多详情请参见[LangGraph平台快速入门](../quick_start.md)。

    ```python
    from typing import TypedDict
    import uuid

    from langgraph.checkpoint.memory import InMemorySaver
    from langgraph.constants import START
    from langgraph.graph import StateGraph
    # highlight-next-line
    from langgraph.types import interrupt, Command

    class State(TypedDict):
        some_text: str

    def human_node(state: State):
        # highlight-next-line
        value = interrupt( # (1)!
            {
                "text_to_revise": state["some_text"] # (2)!
            }
        )
        return {
            "some_text": value # (3)!
        }


    # 构建图
    graph_builder = StateGraph(State)
    graph_builder.add_node("human_node", human_node)
    graph_builder.add_edge(START, "human_node")

    graph = graph_builder.compile()
    ```

    1. `interrupt(...)`在`human_node`处暂停执行，将给定有效载荷呈现给人工
    2. 任何JSON可序列化值都可传递给`interrupt`函数。这里是一个包含待修订文本的字典
    3. 恢复后，`interrupt(...)`的返回值是人工提供的输入，用于更新状态

    运行LangGraph API服务器后，您可以使用
    [LangGraph SDK](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/python_sdk_ref/)与之交互

    === "Python"

        ```python
        from langgraph_sdk import get_client
        # highlight-next-line
        from langgraph_sdk.schema import Command
        client = get_client(url=<DEPLOYMENT_URL>)

        # 使用名为"agent"的部署图
        assistant_id = "agent"

        # 创建线程
        thread = await client.threads.create()
        thread_id = thread["thread_id"]

        # 运行图直到遇到中断
        result = await client.runs.wait(
            thread_id,
            assistant_id,
            input={"some_text": "original text"}   # (1)!
        )

        print(result['__interrupt__']) # (2)!
        # > [
        # >     {
        # >         'value': {'text_to_revise': 'original text'},
        # >         'resumable': True,
        # >         'ns': ['human_node:fc722478-2f21-0578-c572-d9fc4dd07c3b'],
        # >         'when': 'during'
        # >     }
        # > ]


        # 恢复图执行
        print(await client.runs.wait(
            thread_id,
            assistant_id,
            # highlight-next-line
            command=Command(resume="Edited text")   # (3)!
        ))
        # > {'some_text': 'Edited text'}
        ```

        1. 图以初始状态启动
        2. 当图遇到中断时，返回包含有效载荷和元数据的中断对象
        3. 使用`Command(resume=...)`恢复图执行，注入人工输入并继续执行

    === "JavaScript"

        ```js
        import { Client } from "@langchain/langgraph-sdk";
        const client = new Client({ apiUrl: <DEPLOYMENT_URL> });

        // 使用名为"agent"的部署图
        const assistantID = "agent";

        // 创建线程
        const thread = await client.threads.create();
        const threadID = thread["thread_id"];

        // 运行图直到遇到中断
        const result = await client.runs.wait(
          threadID,
          assistantID,
          { input: { "some_text": "original text" } }   // (1)!
        );

        console.log(result['__interrupt__']); // (2)!
        // > [
        // >     {
        // >         'value': {'text_to_revise': 'original text'},
        // >         'resumable': True,
        // >         'ns': ['human_node:fc722478-2f21-0578-c572-d9fc4dd07c3b'],
        // >         'when': 'during'
        // >     }
        // > ]

        // 恢复图执行
        console.log(await client.runs.wait(
            threadID,
            assistantID,
            // highlight-next-line
            { command: { resume: "Edited text" }}   // (3)!
        ));
        // > {'some_text': 'Edited text'}
        ```

        1. 图以初始状态启动
        2. 当图遇到中断时，返回包含有效载荷和元数据的中断对象
        3. 使用`{ resume: ... }`命令对象恢复图执行，注入人工输入并继续执行

    === "cURL"

        创建线程:

        ```bash
        curl --request POST \
        --url <DEPLOYMENT_URL>/threads \
        --header 'Content-Type: application/json' \
        --data '{}'
        ```

        运行图直到遇到中断:

        ```bash
        curl --request POST \
        --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/wait \
        --header 'Content-Type: application/json' \
        --data "{
          \"assistant_id\": \"agent\",
          \"input\": {\\"some_text\\": \\"original text\\"}
        }"
        ```

        恢复图执行:

        ```bash
        curl --request POST \
        --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/wait \
        --header 'Content-Type: application/json' \
        --data "{
          \"assistant_id\": \"agent\",
          \"command\": {
            \"resume\": \"Edited text\"
          }
        }"
        ```

## 了解更多

- [人机交互概念指南](../../concepts/human_in_the_loop.md): 了解更多关于LangGraph人机交互功能
- [常见模式](../../how-tos/human_in_the_loop/add-human-in-the-loop.md#common-patterns): 学习如何实现批准/拒绝操作、请求用户输入、工具调用审查和验证人工输入等模式