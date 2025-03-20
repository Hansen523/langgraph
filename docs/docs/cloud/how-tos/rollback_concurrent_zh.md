# 回滚

本指南假设您已经了解什么是双重发送，您可以在[双重发送概念指南](../../concepts/double_texting.md)中了解更多。

本指南涵盖了双重发送的`rollback`选项，该选项会中断之前的运行图，并使用双重发送启动一个新的运行图。此选项与`interrupt`选项非常相似，但在这种情况下，第一次运行会从数据库中完全删除，无法重新启动。以下是使用`rollback`选项的快速示例。

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
    # 将此内容放入名为pretty_print.sh的文件中
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

    import httpx
    from langchain_core.messages import convert_to_messages
    from langgraph_sdk import get_client

    client = get_client(url=<DEPLOYMENT_URL>)
    # 使用部署名称为“agent”的图
    assistant_id = "agent"
    thread = await client.threads.create()
    ```

=== "Javascript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";

    const client = new Client({ apiUrl: <DEPLOYMENT_URL> });
    // 使用部署名称为“agent”的图
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

现在让我们运行一个线程，并将多任务参数设置为“rollback”：

=== "Python"

    ```python
    # 第一次运行将被回滚
    rolled_back_run = await client.runs.create(
        thread["thread_id"],
        assistant_id,
        input={"messages": [{"role": "user", "content": "what's the weather in sf?"}]},
    )
    run = await client.runs.create(
        thread["thread_id"],
        assistant_id,
        input={"messages": [{"role": "user", "content": "what's the weather in nyc?"}]},
        multitask_strategy="rollback",
    )
    # 等待第二次运行完成
    await client.runs.join(thread["thread_id"], run["run_id"])
    ```

=== "Javascript"

    ```js
    // 第一次运行将被中断
    let rolledBackRun = await client.runs.create(
      thread["thread_id"],
      assistantId,
      { input: { messages: [{ role: "human", content: "what's the weather in sf?" }] } }
    );

    let run = await client.runs.create(
      thread["thread_id"],
      assistant_id,
      { 
        input: { messages: [{ role: "human", content: "what's the weather in nyc?" }] },
        multitaskStrategy: "rollback" 
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
    }" && curl --request POST \
    --url <DEPLOY<ENT_URL>>/threads/<THREAD_ID>/runs \
    --header 'Content-Type: application/json' \
    --data "{
      \"assistant_id\": \"agent\",
      \"input\": {\"messages\": [{\\"role\\": \\"human\\", \\"content\\": \\"what\'s the weather in nyc?\\"}]},
      \"multitask_strategy\": \"rollback\"
    }" && curl --request GET \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/<RUN_ID>/join
    ```

## 查看运行结果

我们可以看到线程中只有第二次运行的数据

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
    
    what's the weather in nyc?
    ================================== AI消息 ==================================
    
    [{'id': 'toolu_01JzPqefao1gxwajHQ3Yh3JD', 'input': {'query': 'weather in nyc'}, 'name': 'tavily_search_results_json', 'type': 'tool_use'}]
    工具调用：
      tavily_search_results_json (toolu_01JzPqefao1gxwajHQ3Yh3JD)
     调用ID：toolu_01JzPqefao1gxwajHQ3Yh3JD
      参数：
        query: weather in nyc
    ================================= 工具消息 =================================
    名称：tavily_search_results_json
    
    [{"url": "https://www.weatherapi.com/", "content": "{'location': {'name': 'New York', 'region': 'New York', 'country': 'United States of America', 'lat': 40.71, 'lon': -74.01, 'tz_id': 'America/New_York', 'localtime_epoch': 1718734479, 'localtime': '2024-06-18 14:14'}, 'current': {'last_updated_epoch': 1718733600, 'last_updated': '2024-06-18 14:00', 'temp_c': 29.4, 'temp_f': 84.9, 'is_day': 1, 'condition': {'text': 'Sunny', 'icon': '//cdn.weatherapi.com/weather/64x64/day/113.png', 'code': 1000}, 'wind_mph': 2.2, 'wind_kph': 3.6, 'wind_degree': 158, 'wind_dir': 'SSE', 'pressure_mb': 1025.0, 'pressure_in': 30.26, 'precip_mm': 0.0, 'precip_in': 0.0, 'humidity': 63, 'cloud': 0, 'feelslike_c': 31.3, 'feelslike_f': 88.3, 'windchill_c': 28.3, 'windchill_f': 82.9, 'heatindex_c': 29.6, 'heatindex_f': 85.3, 'dewpoint_c': 18.4, 'dewpoint_f': 65.2, 'vis_km': 16.0, 'vis_miles': 9.0, 'uv': 7.0, 'gust_mph': 16.5, 'gust_kph': 26.5}}"}]
    ================================== AI消息 ==================================
    
    天气API结果显示，纽约市当前的天气是晴朗的，温度约为85°F（29°C）。风速约为2-3 mph，来自东南偏南方向。总的来说，纽约市今天是一个阳光明媚的夏日。

验证原始的回滚运行已被删除

=== "Python"

    ```python
    try:
        await client.runs.get(thread["thread_id"], rolled_back_run["run_id"])
    except httpx.HTTPStatusError as _:
        print("原始运行已正确删除")
    ```

=== "Javascript"

    ```js
    try {
      await client.runs.get(thread["thread_id"], rolledBackRun["run_id"]);
    } catch (e) {
      console.log("原始运行已正确删除");
    }
    ```

输出：

    原始运行已正确删除