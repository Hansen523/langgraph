# 拒绝

本指南假设您已经了解什么是双文本，您可以在[双文本概念指南](../../concepts/double_texting.md)中了解更多。

本指南涵盖了双文本的`reject`选项，该选项通过抛出错误来拒绝新运行的图，并继续执行原始运行直到完成。以下是使用`reject`选项的快速示例。

## 设置

首先，我们将定义一个快速辅助函数，用于打印出JS和CURL模型的输出（如果使用Python可以跳过此步骤）：

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
    import httpx
    from langchain_core.messages import convert_to_messages
    from langgraph_sdk import get_client

    client = get_client(url=<DEPLOYMENT_URL>)
    # 使用名为“agent”的图
    assistant_id = "agent"
    thread = await client.threads.create()
    ```

=== "Javascript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";

    const client = new Client({ apiUrl: <DEPLOYMENT_URL> });
    // 使用名为“agent”的图
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

现在我们可以运行一个线程，并尝试使用“reject”选项运行第二个线程，由于我们已经开始了一个运行，这应该会失败：

=== "Python"

    ```python
    run = await client.runs.create(
        thread["thread_id"],
        assistant_id,
        input={"messages": [{"role": "user", "content": "what's the weather in sf?"}]},
    )
    try:
        await client.runs.create(
            thread["thread_id"],
            assistant_id,
            input={
                "messages": [{"role": "user", "content": "what's the weather in nyc?"}]
            },
            multitask_strategy="reject",
        )
    except httpx.HTTPStatusError as e:
        print("Failed to start concurrent run", e)
    ```

=== "Javascript"

    ```js
    const run = await client.runs.create(
      thread["thread_id"],
      assistantId,
      input={"messages": [{"role": "user", "content": "what's the weather in sf?"}]},
    );
    
    try {
      await client.runs.create(
        thread["thread_id"],
        assistantId,
        { 
          input: {"messages": [{"role": "user", "content": "what's the weather in nyc?"}]},
          multitask_strategy:"reject"
        },
      );
    } catch (e) {
      console.error("Failed to start concurrent run", e);
    }
    ```

=== "CURL"

    ```bash
    curl --request POST \
    --url <DEPLOY<ENT_URL>>/threads/<THREAD_ID>/runs \
    --header 'Content-Type: application/json' \
    --data "{
      \"assistant_id\": \"agent\",
      \"input\": {\\"messages\\": [{\\"role\\": \\"human\\", \\"content\\": \\"what\'s the weather in sf?\\"}]},
    }" && curl --request POST \
    --url <DEPLOY<ENT_URL>>/threads/<THREAD_ID>/runs \
    --header 'Content-Type: application/json' \
    --data "{
      \"assistant_id\": \"agent\",
      \"input\": {\\"messages\\": [{\\"role\\": \\"human\\", \\"content\\": \\"what\'s the weather in nyc?\\"}]},
      \"multitask_strategy\": \"reject\"
    }" || { echo "Failed to start concurrent run"; echo "Error: $?" >&2; }
    ```

输出：

    Failed to start concurrent run Client error '409 Conflict' for url 'http://localhost:8123/threads/f9e7088b-8028-4e5c-88d2-9cc9a2870e50/runs'
    For more information check: https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/409

## 查看运行结果

我们可以验证原始线程是否已完成执行：

=== "Python"

    ```python
    # 等待原始运行完成
    await client.runs.join(thread["thread_id"], run["run_id"])

    state = await client.threads.get_state(thread["thread_id"])

    for m in convert_to_messages(state["values"]["messages"]):
        m.pretty_print()
    ```

=== "Javascript"

    ```js
    await client.runs.join(thread["thread_id"], run["run_id"]);

    const state = await client.threads.getState(thread["thread_id"]);

    for (const m of state["values"]["messages"]) {
      prettyPrint(m);
    }
    ```

=== "CURL"

    ```bash
    source pretty_print.sh && curl --request GET \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/<RUN_ID>/join && \
    curl --request GET --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/state | \
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
    
    [{'id': 'toolu_01CyewEifV2Kmi7EFKHbMDr1', 'input': {'query': 'weather in san francisco'}, 'name': 'tavily_search_results_json', 'type': 'tool_use'}]
    工具调用：
      tavily_search_results_json (toolu_01CyewEifV2Kmi7EFKHbMDr1)
     调用ID：toolu_01CyewEifV2Kmi7EFKHbMDr1
      参数：
        query: weather in san francisco
    ================================= 工具消息 =================================
    名称：tavily_search_results_json
    
    [{"url": "https://www.accuweather.com/en/us/san-francisco/94103/june-weather/347629", "content": "Get the monthly weather forecast for San Francisco, CA, including daily high/low, historical averages, to help you plan ahead."}]
    ================================== AI消息 ==================================
    
    根据Tavily的搜索结果，旧金山当前的天气情况如下：
    
    旧金山六月的平均高温约为65°F（18°C），平均低温约为54°F（12°C）。六月通常是旧金山较为凉爽和多雾的月份，因为夏季月份海洋层雾经常覆盖城市。
    
    关于旧金山六月典型天气的一些关键点：
    
    - 温和的温度，高温在60多华氏度，低温在50多华氏度
    - 多雾的早晨通常会转为晴朗的下午
    - 几乎没有降雨，因为六月属于旱季
    - 有风，来自太平洋的风
    - 建议穿多层衣物以应对变化的天气条件
    
    总之，您可以期待在旧金山这个时间的早晨温和多雾，下午晴朗但凉爽。海洋层使温度与加州其他地区相比保持适中。