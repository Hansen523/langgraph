# 入队

本指南假设您已经了解什么是双文本处理，您可以在[双文本处理概念指南](../../concepts/double_texting.md)中了解更多。

本指南将介绍双文本处理的`enqueue`选项，该选项将中断添加到队列中，并按照客户端接收的顺序执行它们。以下是一个使用`enqueue`选项的快速示例。

## 设置

首先，我们将定义一个快速辅助函数，用于打印JS和CURL模型的输出（如果您使用Python，可以跳过此步骤）：

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

然后，让我们导入所需的包并实例化我们的客户端、助手和线程。

=== "Python"

    ```python
    import asyncio

    import httpx
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

现在让我们启动两个运行，第二个运行使用“enqueue”多任务策略中断第一个运行：

=== "Python"

    ```python
    first_run = await client.runs.create(
        thread["thread_id"],
        assistant_id,
        input={"messages": [{"role": "user", "content": "what's the weather in sf?"}]},
    )
    second_run = await client.runs.create(
        thread["thread_id"],
        assistant_id,
        input={"messages": [{"role": "user", "content": "what's the weather in nyc?"}]},
        multitask_strategy="enqueue",
    )
    ```

=== "Javascript"

    ```js
    const firstRun = await client.runs.create(
      thread["thread_id"],
      assistantId,
      input={"messages": [{"role": "user", "content": "what's the weather in sf?"}]},
    )

    const secondRun = await client.runs.create(
      thread["thread_id"],
      assistantId,
      input={"messages": [{"role": "user", "content": "what's the weather in nyc?"}]},
      multitask_strategy="enqueue",
    )
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
      \"input\": {\"messages\": [{\"role\": \"human\", \"content\": \"what\'s the weather in nyc?\"}]},
      \"multitask_strategy\": \"enqueue\"
    }"
    ```

## 查看运行结果

验证线程中是否包含两个运行的数据：

=== "Python"

    ```python
    # 等待第二个运行完成
    await client.runs.join(thread["thread_id"], second_run["run_id"])

    state = await client.threads.get_state(thread["thread_id"])

    for m in convert_to_messages(state["values"]["messages"]):
        m.pretty_print()
    ```

=== "Javascript"

    ```js
    await client.runs.join(thread["thread_id"], secondRun["run_id"]);

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
    
    旧金山的天气如何？
    ================================== AI消息 ==================================
    
    [{'id': 'toolu_01Dez1sJre4oA2Y7NsKJV6VT', 'input': {'query': 'weather in san francisco'}, 'name': 'tavily_search_results_json', 'type': 'tool_use'}]
    工具调用：
      tavily_search_results_json (toolu_01Dez1sJre4oA2Y7NsKJV6VT)
     调用ID：toolu_01Dez1sJre4oA2Y7NsKJV6VT
      参数：
        query: 旧金山的天气
    ================================= 工具消息 =================================
    名称：tavily_search_results_json
    
    [{"url": "https://www.accuweather.com/en/us/san-francisco/94103/weather-forecast/347629", "content": "获取旧金山当前的天气状况和未来预报，包括温度、降水、风速、空气质量等。查看每小时和10天的预报、雷达地图、警报和过敏信息。"}]
    ================================== AI消息 ==================================
    
    根据AccuWeather的数据，旧金山当前的天气状况如下：
    
    温度：57°F (14°C)
    天气状况：大部分晴朗
    风速：西南风10 mph
    湿度：72%
    
    未来几天的预报显示，天气以部分晴朗为主，白天最高气温在50多到60多华氏度（14-18°C）之间，夜间最低气温在40多到50多华氏度（9-11°C）之间。这是旧金山这个季节典型的温和干燥天气。
    
    AccuWeather预报中的一些关键细节：
    
    今天：大部分晴朗，最高气温62°F (17°C)
    今晚：部分多云，最低气温49°F (9°C) 
    明天：部分晴朗，最高气温59°F (15°C)
    周六：大部分晴朗，最高气温64°F (18°C)
    周日：部分晴朗，最高气温61°F (16°C)
    
    总结一下，未来几天旧金山的天气将保持春季的温和气候，白天有阳光和云层交替，夜间气温在40多到50多华氏度之间。典型的干燥天气，预报中没有降雨。
    ================================ 人类消息 =================================
    
    纽约的天气如何？
    ================================== AI消息 ==================================
    
    [{'text': '以下是纽约市当前的天气状况和预报：', 'type': 'text'}, {'id': 'toolu_01FFft5Sx9oS6AdVJuRWWcGp', 'input': {'query': 'weather in new york city'}, 'name': 'tavily_search_results_json', 'type': 'tool_use'}]
    工具调用：
      tavily_search_results_json (toolu_01FFft5Sx9oS6AdVJuRWWcGp)
     调用ID：toolu_01FFft5Sx9oS6AdVJuRWWcGp
      参数：
        query: 纽约市的天气
    ================================= 工具消息 =================================
    名称：tavily_search_results_json
    
    [{"url": "https://www.weatherapi.com/", "content": "{'location': {'name': 'New York', 'region': 'New York', 'country': 'United States of America', 'lat': 40.71, 'lon': -74.01, 'tz_id': 'America/New_York', 'localtime_epoch': 1718734479, 'localtime': '2024-06-18 14:14'}, 'current': {'last_updated_epoch': 1718733600, 'last_updated': '2024-06-18 14:00', 'temp_c': 29.4, 'temp_f': 84.9, 'is_day': 1, 'condition': {'text': 'Sunny', 'icon': '//cdn.weatherapi.com/weather/64x64/day/113.png', 'code': 1000}, 'wind_mph': 2.2, 'wind_kph': 3.6, 'wind_degree': 158, 'wind_dir': 'SSE', 'pressure_mb': 1025.0, 'pressure_in': 30.26, 'precip_mm': 0.0, 'precip_in': 0.0, 'humidity': 63, 'cloud': 0, 'feelslike_c': 31.3, 'feelslike_f': 88.3, 'windchill_c': 28.3, 'windchill_f': 82.9, 'heatindex_c': 29.6, 'heatindex_f': 85.3, 'dewpoint_c': 18.4, 'dewpoint_f': 65.2, 'vis_km': 16.0, 'vis_miles': 9.0, 'uv': 7.0, 'gust_mph': 16.5, 'gust_kph': 26.5}}"}]
    ================================== AI消息 ==================================
    
    根据WeatherAPI的天气数据：
    
    纽约市当前的天气状况（截至当地时间下午2:00）：
    - 温度：85°F (29°C)
    - 天气状况：晴朗
    - 风速：2 mph (4 km/h)，风向SSE
    - 湿度：63%
    - 体感温度：85°F (30°C)
    
    预报显示，未来几天天气将持续晴朗温暖：
    
    今天：晴朗，最高气温85°F (29°C)
    今晚：晴朗，最低气温68°F (20°C)
    明天：晴朗，最高气温88°F (31°C) 
    周四：大部分晴朗，最高气温90°F (32°C)
    周五：部分多云，最高气温87°F (31°C)
    
    因此，纽约市正经历美丽的晴朗天气，气温在80多华氏度（约30°C）之间，湿度在60%左右。总体而言，未来几天是户外活动的理想天气。