# 中断

本指南假设您已经了解什么是双文本发送，您可以在[双文本发送概念指南](../../concepts/double_texting.md)中学习相关知识。

本指南将介绍双文本发送的`interrupt`选项，该选项会中断图的前一次运行，并使用双文本启动一个新的运行。此选项不会删除第一次运行，而是将其保留在数据库中，并将其状态设置为`interrupted`。以下是使用`interrupt`选项的快速示例。

## 设置

首先，我们将定义一个快速辅助函数，用于打印出JS和CURL模型的输出（如果使用Python，可以跳过此步骤）：

=== "Javascript"

    ```js
    function prettyPrint(m) {
      const padded = " " + m['type'] + " ";
      const sepLen = Math.floor((80 - padded.length) / 2);
      const sep = "=".repeat(sepLen);
      const secondSep = sep + (padded.length % 2 ? "=" : "");
      
      console.log(`${sep}${padded}${secondSep}`);
      console.log("\n\n");
      console.log(m.content);
    }
    ```

=== "CURL"

    ```bash
    # 将此内容放在名为pretty_print.sh的文件中
    pretty_print() {
      local type="$1"
      local content="$2"
      local padded=" $type "
      local total_width=80
      local sep_len=$(( (total_width - ${#padded}) / 2 ))
      local sep=$(printf '=%.0s' $(eval "echo {1.."${sep_len}"}"))
      local second_sep=$sep
      if (( (total_width - ${#padded}) % 2 )); then
        second_sep="${second_sep}="
      fi

      echo "${sep}${padded}${second_sep}"
      echo
      echo "$content"
    }
    ```

现在，让我们导入所需的包并实例化我们的客户端、助手和线程。

=== "Python"

    ```python
    import asyncio

    from langchain_core.messages import convert_to_messages
    from langgraph_sdk import get_client

    client = get_client(url=<DEPLOYMENT_URL>)
    # 使用名为"agent"的图
    assistant_id = "agent"
    thread = await client.threads.create()
    ```

=== "Javascript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";

    const client = new Client({ apiUrl: <DEPLOYMENT_URL> });
    // 使用名为"agent"的图
    const assistantId = "agent";
    const thread = await client.threads.create();
    ```

=== "CURL"

    ```bash
    curl --request POST \
      --url <DEPLOYMENT_URL>/threads \
      --header 'Content-Type: application/json' \
      --data '{}'
    ```

## 创建运行

现在我们可以启动两次运行，并等待第二次运行完成：

=== "Python"

    ```python
    # 第一次运行将被中断
    interrupted_run = await client.runs.create(
        thread["thread_id"],
        assistant_id,
        input={"messages": [{"role": "user", "content": "what's the weather in sf?"}]},
    )
    # 稍等片刻以获取第一次运行的部分输出
    await asyncio.sleep(2)
    run = await client.runs.create(
        thread["thread_id"],
        assistant_id,
        input={"messages": [{"role": "user", "content": "what's the weather in nyc?"}]},
        multitask_strategy="interrupt",
    )
    # 等待第二次运行完成
    await client.runs.join(thread["thread_id"], run["run_id"])
    ```

=== "Javascript"

    ```js
    // 第一次运行将被中断
    let interruptedRun = await client.runs.create(
      thread["thread_id"],
      assistantId,
      { input: { messages: [{ role: "human", content: "what's the weather in sf?" }] } }
    );
    // 稍等片刻以获取第一次运行的部分输出
    await new Promise(resolve => setTimeout(resolve, 2000)); 

    let run = await client.runs.create(
      thread["thread_id"],
      assistantId,
      { 
        input: { messages: [{ role: "human", content: "what's the weather in nyc?" }] },
        multitaskStrategy: "interrupt" 
      }
    );

    // 等待第二次运行完成
    await client.runs.join(thread["thread_id"], run["run_id"]);
    ```

=== "CURL"

    ```bash
    curl --request POST \
    --url <DEPLOY<ENT_URL>>/threads/<THREAD_ID>/runs \
    --header 'Content-Type: application/json' \
    --data "{
      \"assistant_id\": \"agent\",
      \"input\": {\"messages\": [{\"role\": \"human\", \"content\": \"what\'s the weather in sf?\"}]},
    }" && sleep 2 && curl --request POST \
    --url <DEPLOY<ENT_URL>>/threads/<THREAD_ID>/runs \
    --header 'Content-Type: application/json' \
    --data "{
      \"assistant_id\": \"agent\",
      \"input\": {\"messages\": [{\\"role\\": \\"human\\", \\"content\\": \\"what\'s the weather in nyc?\\"}]},
      \"multitask_strategy\": \"interrupt\"
    }" && curl --request GET \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/<RUN_ID>/join
    ```

## 查看运行结果

我们可以看到线程中包含了第一次运行的部分数据和第二次运行的完整数据

=== "Python"

    ```python
    state = await client.threads.get_state(thread["thread_id"])

    for m in convert_to_messages(state["values"]["messages"]):
        m.pretty_print()
    ```

=== "Javascript"

    ```js
    const state = await client.threads.getState(thread["thread_id"]);

    for (const m of state['values']['messages']) {
      prettyPrint(m);
    }
    ```

=== "CURL"

    ```bash
    source pretty_print.sh && curl --request GET \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/state | \
    jq -c '.values.messages[]' | while read -r element; do
        type=$(echo "$element" | jq -r '.type')
        content=$(echo "$element" | jq -r '.content | if type == "array" then tostring else . end')
        pretty_print "$type" "$content"
    done
    ```

输出：

    ================================ 人类消息 =================================
    
    what's the weather in sf?
    ================================== AI消息 ==================================
    
    [{'id': 'toolu_01MjNtVJwEcpujRGrf3x6Pih', 'input': {'query': 'weather in san francisco'}, 'name': 'tavily_search_results_json', 'type': 'tool_use'}]
    工具调用：
      tavily_search_results_json (toolu_01MjNtVJwEcpujRGrf3x6Pih)
     调用ID：toolu_01MjNtVJwEcpujRGrf3x6Pih
      参数：
        query: weather in san francisco
    ================================= 工具消息 =================================
    名称：tavily_search_results_json
    
    [{"url": "https://www.wunderground.com/hourly/us/ca/san-francisco/KCASANFR2002/date/2024-6-18", "content": "High 64F. Winds W at 10 to 20 mph. A few clouds from time to time. Low 49F. Winds W at 10 to 20 mph. Temp. San Francisco Weather Forecasts. Weather Underground provides local & long-range weather ..."}]
    ================================ 人类消息 =================================
    
    what's the weather in nyc?
    ================================== AI消息 ==================================
    
    [{'id': 'toolu_01KtE1m1ifPLQAx4fQLyZL9Q', 'input': {'query': 'weather in new york city'}, 'name': 'tavily_search_results_json', 'type': 'tool_use'}]
    工具调用：
      tavily_search_results_json (toolu_01KtE1m1ifPLQAx4fQLyZL9Q)
     调用ID：toolu_01KtE1m1ifPLQAx4fQLyZL9Q
      参数：
        query: weather in new york city
    ================================= 工具消息 =================================
    名称：tavily_search_results_json
    
    [{"url": "https://www.accuweather.com/en/us/new-york/10021/june-weather/349727", "content": "Get the monthly weather forecast for New York, NY, including daily high/low, historical averages, to help you plan ahead."}]
    ================================== AI消息 ==================================
    
    搜索结果提供了纽约市的天气预报和信息。根据AccuWeather的顶部结果，以下是关于纽约市天气的一些关键细节：
    
    - 这是纽约市六月份的月度天气预报。
    - 它包括每日最高和最低温度，以帮助您提前计划。
    - 还提供了纽约市六月的历史平均值作为参考点。
    - 更多详细的每日或每小时预报，包括降水概率、湿度、风速等，可以通过访问AccuWeather页面找到。
    
    总之，搜索提供了未来一个月纽约市预期天气条件的便捷概述，以便您在旅行或制定计划时有所准备。如果您需要任何其他详细信息，请告诉我！


验证原始的中断运行是否被中断

=== "Python"

    ```python
    print((await client.runs.get(thread["thread_id"], interrupted_run["run_id"]))["status"])
    ```

=== "Javascript"

    ```js
    console.log((await client.runs.get(thread['thread_id'], interruptedRun["run_id"]))["status"])
    ```

输出：

    'interrupted'