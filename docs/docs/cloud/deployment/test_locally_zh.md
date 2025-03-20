# 如何在本地测试LangGraph应用

本指南假设您已正确设置了一个LangGraph应用，并拥有一个正确的配置文件和相应的编译图，同时您还拥有一个有效的LangChain API密钥。

本地测试确保没有与Python依赖项的冲突，并确认配置文件已正确指定。

## 设置

安装LangGraph CLI包：

```bash
pip install -U "langgraph-cli[inmem]"
```

确保您有一个API密钥，您可以从[LangSmith UI](https://smith.langchain.com)（设置 > API密钥）创建。这是验证您是否具有LangGraph Cloud访问权限所必需的。将密钥保存到安全位置后，将以下行添加到您的`.env`文件中：

```python
LANGSMITH_API_KEY = *********
```

## 启动API服务器

安装CLI后，您可以运行以下命令以启动用于本地测试的API服务器：

```shell
langgraph dev
```

这将启动本地的LangGraph API服务器。如果成功运行，您应该会看到类似以下内容：

>    准备就绪！
> 
>    - API: [http://localhost:2024](http://localhost:2024/)
>     
>    - 文档: http://localhost:2024/docs
>     
>    - LangGraph Studio Web UI: https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:2024

!!! 注意 "内存模式"

    `langgraph dev`命令以内存模式启动LangGraph服务器。此模式适用于开发和测试目的。对于生产使用，您应该部署具有持久存储后端访问权限的LangGraph服务器。

    如果您想使用持久存储后端测试您的应用程序，可以使用`langgraph up`命令代替`langgraph dev`。您需要
    在机器上安装`docker`才能使用此命令。


### 与服务器交互

我们现在可以使用LangGraph SDK与API服务器进行交互。首先，我们需要启动我们的客户端，选择我们的助手（在本例中是一个名为“agent”的图，确保选择您希望测试的正确助手）。

您可以通过传递身份验证或设置环境变量来初始化。

#### 使用身份验证初始化

=== "Python"

    ```python
    from langgraph_sdk import get_client

    # 只有在调用langgraph dev时更改了默认端口，才需要将url参数传递给get_client()
    client = get_client(url=<DEPLOYMENT_URL>,api_key=<LANGSMITH_API_KEY>)
    # 使用名为“agent”的图
    assistant_id = "agent"
    thread = await client.threads.create()
    ```

=== "Javascript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";

    // 只有在调用langgraph dev时更改了默认端口，才需要设置apiUrl
    const client = new Client({ apiUrl: <DEPLOYMENT_URL>, apiKey: <LANGSMITH_API_KEY> });
    // 使用名为“agent”的图
    const assistantId = "agent";
    const thread = await client.threads.create();
    ```

=== "CURL"

    ```bash
    curl --request POST \
      --url <DEPLOYMENT_URL>/threads \
      --header 'Content-Type: application/json'
      --header 'x-api-key: <LANGSMITH_API_KEY>'
    ```
  

#### 使用环境变量初始化

如果您的环境中设置了`LANGSMITH_API_KEY`，则无需显式传递身份验证给客户端

=== "Python"

    ```python
    from langgraph_sdk import get_client

    # 只有在调用langgraph dev时更改了默认端口，才需要将url参数传递给get_client()
    client = get_client()
    # 使用名为“agent”的图
    assistant_id = "agent"
    thread = await client.threads.create()
    ```

=== "Javascript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";

    // 只有在调用langgraph dev时更改了默认端口，才需要设置apiUrl
    const client = new Client();
    // 使用名为“agent”的图
    const assistantId = "agent";
    const thread = await client.threads.create();
    ```

=== "CURL"

    ```bash
    curl --request POST \
      --url <DEPLOYMENT_URL>/threads \
      --header 'Content-Type: application/json'
    ```

现在我们可以调用我们的图以确保其正常工作。确保更改输入以匹配您的图的正确架构。

=== "Python"

    ```python
    input = {"messages": [{"role": "user", "content": "what's the weather in sf"}]}
    async for chunk in client.runs.stream(
        thread["thread_id"],
        assistant_id,
        input=input,
        stream_mode="updates",
    ):
        print(f"Receiving new event of type: {chunk.event}...")
        print(chunk.data)
        print("\n\n")
    ```
=== "Javascript"

    ```js
    const input = { "messages": [{ "role": "user", "content": "what's the weather in sf"}] }

    const streamResponse = client.runs.stream(
      thread["thread_id"],
      assistantId,
      {
        input: input,
        streamMode: "updates",
      }
    );
    for await (const chunk of streamResponse) {
      console.log(`Receiving new event of type: ${chunk.event}...`);
      console.log(chunk.data);
      console.log("\n\n");
    }
    ```

=== "CURL"

    ```bash
    curl --request POST \
     --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/stream \
     --header 'Content-Type: application/json' \
     --data "{
       \"assistant_id\": \"agent\",
       \"input\": {\"messages\": [{\"role\": \"human\", \"content\": \"what's the weather in sf\"}]},
       \"stream_mode\": [
         \"events\"
       ]
     }" | \
     sed 's/\r$//' | \
     awk '
     /^event:/ {
         if (data_content != "") {
             print data_content "\n"
         }
         sub(/^event: /, "Receiving event of type: ", $0)
         printf "%s...\n", $0
         data_content = ""
     }
     /^data:/ {
         sub(/^data: /, "", $0)
         data_content = $0
     }
     END {
         if (data_content != "") {
             print data_content "\n"
         }
     }
     ' 
    ```

如果您的图正常工作，您应该会在控制台中看到您的图输出。当然，您可能需要以更多方式测试您的图，有关可以使用SDK发送的完整命令列表，请参阅[Python](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/python_sdk_ref/)和[JS/TS](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/js_ts_sdk_ref/)参考文档。