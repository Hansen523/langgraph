# 如何将LangGraph集成到你的React应用中

!!! info "先决条件"
    - [LangGraph平台](../../concepts/langgraph_platform.md)
    - [LangGraph服务器](../../concepts/langgraph_server.md)

`useStream()` React钩子提供了一种无缝的方式将LangGraph集成到你的React应用中。它处理了所有流式传输、状态管理和分支逻辑的复杂性，让你专注于构建出色的聊天体验。

主要功能：

- 消息流式传输：处理消息块流以形成完整的消息
- 自动状态管理，包括消息、中断、加载状态和错误
- 对话分支：从聊天历史中的任何点创建替代对话路径
- 与UI无关的设计：自带组件和样式

让我们探索如何在你的React应用中使用`useStream()`。

`useStream()`为创建定制聊天体验提供了坚实的基础。对于预构建的聊天组件和界面，我们还推荐查看[CopilotKit](https://docs.copilotkit.ai/coagents/quickstart/langgraph)和[assistant-ui](https://www.assistant-ui.com/docs/runtimes/langgraph)。

## 安装

```bash
npm install @langchain/langgraph-sdk @langchain/core
```

## 示例

```tsx
"use client";

import { useStream } from "@langchain/langgraph-sdk/react";
import type { Message } from "@langchain/langgraph-sdk";

export default function App() {
  const thread = useStream<{ messages: Message[] }>({
    apiUrl: "http://localhost:2024",
    assistantId: "agent",
    messagesKey: "messages",
  });

  return (
    <div>
      <div>
        {thread.messages.map((message) => (
          <div key={message.id}>{message.content as string}</div>
        ))}
      </div>

      <form
        onSubmit={(e) => {
          e.preventDefault();

          const form = e.target as HTMLFormElement;
          const message = new FormData(form).get("message") as string;

          form.reset();
          thread.submit({ messages: [{ type: "human", content: message }] });
        }}
      >
        <input type="text" name="message" />

        {thread.isLoading ? (
          <button key="stop" type="button" onClick={() => thread.stop()}>
            Stop
          </button>
        ) : (
          <button keytype="submit">Send</button>
        )}
      </form>
    </div>
  );
}
```

## 自定义你的UI

`useStream()`钩子处理了所有复杂的后台状态管理，为你提供了构建UI的简单接口。以下是开箱即用的功能：

- 线程状态管理
- 加载和错误状态
- 中断
- 消息处理和更新
- 分支支持

以下是一些如何有效使用这些功能的示例：

### 加载状态

`isLoading`属性告诉你流是否处于活动状态，使你能够：

- 显示加载指示器
- 在处理过程中禁用输入字段
- 显示取消按钮

```tsx
export default function App() {
  const { isLoading, stop } = useStream<{ messages: Message[] }>({
    apiUrl: "http://localhost:2024",
    assistantId: "agent",
    messagesKey: "messages",
  });

  return (
    <form>
      {isLoading && (
        <button key="stop" type="button" onClick={() => stop()}>
          Stop
        </button>
      )}
    </form>
  );
}
```

### 线程管理

使用内置的线程管理跟踪对话。你可以访问当前线程ID，并在创建新线程时收到通知：

```tsx
const [threadId, setThreadId] = useState<string | null>(null);

const thread = useStream<{ messages: Message[] }>({
  apiUrl: "http://localhost:2024",
  assistantId: "agent",

  threadId: threadId,
  onThreadId: setThreadId,
});
```

我们建议将`threadId`存储在URL的查询参数中，以便用户在页面刷新后可以恢复对话。

### 消息处理

`useStream()`钩子将跟踪从服务器接收到的消息块，并将它们连接在一起以形成完整的消息。可以通过`messages`属性检索完整的消息块。

默认情况下，`messagesKey`设置为`messages`，它将新的消息块附加到`values["messages"]`。如果你将消息存储在不同的键中，可以更改`messagesKey`的值。

```tsx
import type { Message } from "@langchain/langgraph-sdk";
import { useStream } from "@langchain/langgraph-sdk/react";

export default function HomePage() {
  const thread = useStream<{ messages: Message[] }>({
    apiUrl: "http://localhost:2024",
    assistantId: "agent",
    messagesKey: "messages",
  });

  return (
    <div>
      {thread.messages.map((message) => (
        <div key={message.id}>{message.content as string}</div>
      ))}
    </div>
  );
}
```

在后台，`useStream()`钩子将使用`streamMode: "messages-key"`从你的图节点中的任何LangChain聊天模型调用中接收消息流（即单个LLM令牌）。了解更多关于消息流式传输的信息，请参阅[如何从你的图中流式传输消息](./stream_messages.md)指南。

### 中断

`useStream()`钩子暴露了`interrupt`属性，它将填充线程中的最后一个中断。你可以使用中断来：

- 在执行节点之前渲染确认UI
- 等待人工输入，允许代理向用户提出澄清问题

了解更多关于中断的信息，请参阅[如何处理中断](../../how-tos/human_in_the_loop/wait-user-input.ipynb)指南。

```tsx
const thread = useStream<
  { messages: Message[] },
  { InterruptType: string }
>({
  apiUrl: "http://localhost:2024",
  assistantId: "agent",
  messagesKey: "messages",
});

if (thread.interrupt) {
  return (
    <div>
      Interrupted! {thread.interrupt.value}

      <button
        type="button"
        onClick={() => {
          // `resume`可以是代理接受的任何值
          thread.submit(undefined, { command: { resume: true } });
        }}
      >
        Resume
      </button>
    </div>
  );
}
```

### 分支

对于每条消息，你可以使用`getMessagesMetadata()`获取消息首次出现的第一个检查点。然后你可以从首次出现的检查点之前的检查点创建一个新的运行，以在线程中创建一个新分支。

可以通过以下方式创建分支：

1. 编辑之前的用户消息。
2. 请求重新生成之前的助手消息。

```tsx
"use client";

import type { Message } from "@langchain/langgraph-sdk";
import { useStream } from "@langchain/langgraph-sdk/react";
import { useState } from "react";

function BranchSwitcher({
  branch,
  branchOptions,
  onSelect,
}: {
  branch: string | undefined;
  branchOptions: string[] | undefined;
  onSelect: (branch: string) => void;
}) {
  if (!branchOptions || !branch) return null;
  const index = branchOptions.indexOf(branch);

  return (
    <div className="flex items-center gap-2">
      <button
        type="button"
        onClick={() => {
          const prevBranch = branchOptions[index - 1];
          if (!prevBranch) return;
          onSelect(prevBranch);
        }}
      >
        Prev
      </button>
      <span>
        {index + 1} / {branchOptions.length}
      </span>
      <button
        type="button"
        onClick={() => {
          const nextBranch = branchOptions[index + 1];
          if (!nextBranch) return;
          onSelect(nextBranch);
        }}
      >
        Next
      </button>
    </div>
  );
}

function EditMessage({
  message,
  onEdit,
}: {
  message: Message;
  onEdit: (message: Message) => void;
}) {
  const [editing, setEditing] = useState(false);

  if (!editing) {
    return (
      <button type="button" onClick={() => setEditing(true)}>
        Edit
      </button>
    );
  }

  return (
    <form
      onSubmit={(e) => {
        e.preventDefault();
        const form = e.target as HTMLFormElement;
        const content = new FormData(form).get("content") as string;

        form.reset();
        onEdit({ type: "human", content });
        setEditing(false);
      }}
    >
      <input name="content" defaultValue={message.content as string} />
      <button type="submit">Save</button>
    </form>
  );
}

export default function App() {
  const thread = useStream({
    apiUrl: "http://localhost:2024",
    assistantId: "agent",
    messagesKey: "messages",
  });

  return (
    <div>
      <div>
        {thread.messages.map((message) => {
          const meta = thread.getMessagesMetadata(message);
          const parentCheckpoint = meta?.firstSeenState?.parent_checkpoint;

          return (
            <div key={message.id}>
              <div>{message.content as string}</div>

              {message.type === "human" && (
                <EditMessage
                  message={message}
                  onEdit={(message) =>
                    thread.submit(
                      { messages: [message] },
                      { checkpoint: parentCheckpoint },
                    )
                  }
                />
              )}

              {message.type === "ai" && (
                <button
                  type="button"
                  onClick={() =>
                    thread.submit(undefined, { checkpoint: parentCheckpoint })
                  }
                >
                  <span>Regenerate</span>
                </button>
              )}

              <BranchSwitcher
                branch={meta?.branch}
                branchOptions={meta?.branchOptions}
                onSelect={(branch) => thread.setBranch(branch)}
              />
            </div>
          );
        })}
      </div>

      <form
        onSubmit={(e) => {
          e.preventDefault();

          const form = e.target as HTMLFormElement;
          const message = new FormData(form).get("message") as string;

          form.reset();
          thread.submit({ messages: [message] });
        }}
      >
        <input type="text" name="message" />

        {thread.isLoading ? (
          <button key="stop" type="button" onClick={() => thread.stop()}>
            Stop
          </button>
        ) : (
          <button key="submit" type="submit">
            Send
          </button>
        )}
      </form>
    </div>
  );
}
```

对于高级用例，你可以使用`experimental_branchTree`属性获取线程的树表示，它可以用于渲染基于非消息图的分支控件。

### TypeScript

`useStream()`钩子对使用TypeScript编写的应用非常友好，你可以为状态指定类型以获得更好的类型安全性和IDE支持。

```tsx
// 定义你的类型
type State = {
  messages: Message[];
  context?: Record<string, unknown>;
};

// 在钩子中使用它们
const thread = useStream<State>({
  apiUrl: "http://localhost:2024",
  assistantId: "agent",
  messagesKey: "messages",
});
```

你还可以为不同场景可选地指定类型，例如：

- `ConfigurableType`：`config.configurable`属性的类型（默认：`Record<string, unknown>`）
- `InterruptType`：中断值的类型 - 即`interrupt(...)`函数的内容（默认：`unknown`）
- `CustomEventType`：自定义事件的类型（默认：`unknown`）
- `UpdateType`：提交函数的类型（默认：`Partial<State>`）

```tsx

const thread = useStream<State, {
  UpdateType: {
    messages: Message[] | Message;
    context?: Record<string, unknown>;
  };
  InterruptType: string;
  CustomEventType: {
    type: "progress" | "debug";
    payload: unknown;
  };
  ConfigurableType: {
    model: string;
  };
}>({
  apiUrl: "http://localhost:2024",
  assistantId: "agent",
  messagesKey: "messages",
});
```

如果你使用LangGraph.js，你还可以重用你的图的注释类型。但是，请确保仅导入注释模式的类型，以避免导入整个LangGraph.js运行时（即通过`import type { ... }`指令）。

```tsx
import {
  Annotation,
  MessagesAnnotation,
  type StateType,
  type UpdateType,
} from "@langchain/langgraph/web";

const AgentState = Annotation.Root({
  ...MessagesAnnotation.spec,
  context: Annotation<string>(),
});

const thread = useStream<
  StateType<typeof AgentState.spec>,
  { UpdateType: UpdateType<typeof AgentState.spec> }
>({
  apiUrl: "http://localhost:2024",
  assistantId: "agent",
  messagesKey: "messages",
});
```

## 事件处理

`useStream()`钩子提供了几个回调选项，以帮助你响应不同的事件：

- `onError`：发生错误时调用。
- `onFinish`：流结束时调用。
- `onUpdateEvent`：接收到更新事件时调用。
- `onCustomEvent`：接收到自定义事件时调用。请参阅[自定义事件](../../concepts/streaming.md#custom)了解如何流式传输自定义事件。
- `onMetadataEvent`：接收到元数据事件时调用，其中包含运行ID和线程ID。

## 了解更多

- [JS/TS SDK参考](../reference/sdk/js_ts_sdk_ref.md)