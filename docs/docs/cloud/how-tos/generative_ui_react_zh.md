# 如何使用LangGraph实现生成式用户界面

!!! info "前提条件"

    - [LangGraph平台](../../concepts/langgraph_platform.md)
    - [LangGraph服务器](../../concepts/langgraph_server.md)
    - [`useStream()` React Hook](./use_stream_react.md)

生成式用户界面（Generative UI）允许代理超越文本，生成丰富的用户界面。这使得创建更具交互性和上下文感知的应用程序成为可能，其中用户界面会根据对话流程和AI响应进行适配。

![生成式用户界面示例](./img/generative_ui_sample.jpg)

LangGraph平台支持将React组件与图代码放在一起。这使您可以专注于为图构建特定的UI组件，同时轻松地插入到现有的聊天界面中，如[Agent Chat](https://agentchat.vercel.app)，并仅在需要时加载代码。

!!! warning "仅限LangGraph.js"

    目前只有LangGraph.js支持生成式用户界面。Python的支持即将推出。

## 教程

### 1. 定义和配置UI组件

首先，创建您的第一个UI组件。对于每个组件，您需要提供一个唯一的标识符，该标识符将用于在图代码中引用该组件。

```tsx title="src/agent/ui.tsx"
const WeatherComponent = (props: { city: string }) => {
  return <div>{props.city}的天气</div>;
};

export default {
  weather: WeatherComponent,
};
```

接下来，在`langgraph.json`配置中定义您的UI组件：

```json
{
  "node_version": "20",
  "graphs": {
    "agent": "./src/agent/index.ts:graph"
  },
  "ui": {
    "agent": "./src/agent/ui.tsx"
  }
}
```

`ui`部分指向图将使用的UI组件。默认情况下，我们建议使用与图名称相同的键，但您可以根据需要拆分组件，详情请参见[自定义UI组件的命名空间](#自定义UI组件的命名空间)。

LangGraph平台将自动打包您的UI组件代码和样式，并将其作为外部资源提供，以便`LoadExternalComponent`组件加载。一些依赖项如`react`和`react-dom`将自动从捆绑包中排除。

CSS和Tailwind 4.x也开箱即用，因此您可以在UI组件中自由使用Tailwind类以及`shadcn/ui`。

=== "`src/agent/ui.tsx`"

    ```tsx
    import "./styles.css";

    const WeatherComponent = (props: { city: string }) => {
      return <div className="bg-red-500">{props.city}的天气</div>;
    };

    export default {
      weather: WeatherComponent,
    };
    ```

=== "`src/agent/styles.css`"

    ```css
    @import "tailwindcss";
    ```

### 2. 在图中发送UI组件

使用`typedUi`实用程序从代理节点发出UI元素：

```typescript title="src/agent/index.ts"
import {
  typedUi,
  uiMessageReducer,
} from "@langchain/langgraph-sdk/react-ui/server";

import { ChatOpenAI } from "@langchain/openai";
import { v4 as uuidv4 } from "uuid";
import { z } from "zod";

import type ComponentMap from "./ui.js";

import {
  Annotation,
  MessagesAnnotation,
  StateGraph,
  type LangGraphRunnableConfig,
} from "@langchain/langgraph";

const AgentState = Annotation.Root({
  ...MessagesAnnotation.spec,
  ui: Annotation({ reducer: uiMessageReducer, default: () => [] }),
});

export const graph = new StateGraph(AgentState)
  .addNode("weather", async (state, config) => {
    // 提供组件映射的类型以确保`ui.push()`调用的类型安全，
    // 并将消息推送到`ui`并发送自定义事件。
    const ui = typedUi<typeof ComponentMap>(config);

    const weather = await new ChatOpenAI({ model: "gpt-4o-mini" })
      .withStructuredOutput(z.object({ city: z.string() }))
      .withConfig({ tags: ["langsmith:nostream"] })
      .invoke(state.messages);

    const response = {
      id: uuidv4(),
      type: "ai",
      content: `这是${weather.city}的天气`,
    };

    // 发出带有相关AI消息的UI元素
    ui.push({ name: "weather", props: weather }, { message: response });

    return { messages: [response] };
  })
  .addEdge("__start__", "weather")
  .compile();
```

### 3. 在React应用程序中处理UI元素

在客户端，您可以使用`useStream()`和`LoadExternalComponent`来显示UI元素。

```tsx title="src/app/page.tsx"
"use client";

import { useStream } from "@langchain/langgraph-sdk/react";
import { LoadExternalComponent } from "@langchain/langgraph-sdk/react-ui";

export default function Page() {
  const { thread, values } = useStream({
    apiUrl: "http://localhost:2024",
    assistantId: "agent",
  });

  return (
    <div>
      {thread.messages.map((message) => (
        <div key={message.id}>
          {message.content}
          {values.ui
            ?.filter((ui) => ui.metadata?.message_id === message.id)
            .map((ui) => (
              <LoadExternalComponent key={ui.id} stream={thread} message={ui} />
            ))}
        </div>
      ))}
    </div>
  );
}
```

在幕后，`LoadExternalComponent`将从LangGraph平台获取UI组件的JS和CSS，并在Shadow DOM中渲染它们，从而确保与应用程序其余部分的样式隔离。

## 操作指南

### 在组件加载时显示加载UI

您可以提供一个后备UI，在组件加载时渲染。

```tsx
<LoadExternalComponent
  stream={thread}
  message={ui}
  fallback={<div>加载中...</div>}
/>
```

### 在客户端提供自定义组件

如果您已经在客户端应用程序中加载了组件，您可以提供一个组件映射，直接渲染这些组件，而无需从LangGraph平台获取UI代码。

```tsx
const clientComponents = {
  weather: WeatherComponent,
};

<LoadExternalComponent
  stream={thread}
  message={ui}
  components={clientComponents}
/>;
```

### 自定义UI组件的命名空间

默认情况下，`LoadExternalComponent`将使用`useStream()`钩子中的`assistantId`来获取UI组件的代码。您可以通过向`LoadExternalComponent`组件提供`namespace`属性来自定义此行为。

=== "`src/app/page.tsx`"

    ```tsx
    <LoadExternalComponent
      stream={thread}
      message={ui}
      namespace="custom-namespace"
    />
    ```

=== "`langgraph.json`"

    ```json
    {
      "ui": {
        "custom-namespace": "./src/agent/ui.tsx"
      }
    }
    ```

### 从UI组件访问和与线程状态交互

您可以使用`useStreamContext`钩子在UI组件内部访问线程状态。

```tsx
import { useStreamContext } from "@langchain/langgraph-sdk/react-ui";

const WeatherComponent = (props: { city: string }) => {
  const { thread, submit } = useStreamContext();
  return (
    <>
      <div>{props.city}的天气</div>

      <button
        onClick={() => {
          const newMessage = {
            type: "human",
            content: `${props.city}的天气怎么样？`,
          };

          submit({ messages: [newMessage] });
        }}
      >
        重试
      </button>
    </>
  );
};
```

### 向客户端组件传递额外的上下文

您可以通过向`LoadExternalComponent`组件提供`meta`属性来向客户端组件传递额外的上下文。

```tsx
<LoadExternalComponent stream={thread} message={ui} meta={{ userId: "123" }} />
```

然后，您可以使用`useStreamContext`钩子在UI组件中访问`meta`属性。

```tsx
import { useStreamContext } from "@langchain/langgraph-sdk/react-ui";

const WeatherComponent = (props: { city: string }) => {
  const { meta } = useStreamContext<
    { city: string },
    { MetaType: { userId?: string } }
  >();

  return (
    <div>
      {props.city}的天气 (用户: {meta?.userId})
    </div>
  );
};
```

### 在节点执行完成之前流式更新UI

您可以使用`useStream()`钩子的`onCustomEvent`回调在节点执行完成之前流式更新UI。

```tsx
import { uiMessageReducer } from "@langchain/langgraph-sdk/react-ui";

const { thread, submit } = useStream({
  apiUrl: "http://localhost:2024",
  assistantId: "agent",
  onCustomEvent: (event, options) => {
    options.mutate((prev) => {
      const ui = uiMessageReducer(prev.ui ?? [], event);
      return { ...prev, ui };
    });
  },
});
```

### 从状态中移除UI消息

类似于通过附加RemoveMessage从状态中移除消息的方式，您可以通过调用`ui.delete`并传入UI消息的ID来从状态中移除UI消息。

```tsx
// 推送的消息
const message = ui.push({ name: "weather", props: { city: "London" } });

// 移除该消息
ui.delete(message.id);

// 返回新状态以持久化更改
return { ui: ui.items };
```

## 了解更多

- [JS/TS SDK参考](../reference/sdk/js_ts_sdk_ref.md)