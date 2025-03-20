# 如何设置 LangGraph.js 应用程序以进行部署

要将 [LangGraph.js](https://langchain-ai.github.io/langgraphjs/) 应用程序部署到 LangGraph Cloud（或自托管），必须使用 [LangGraph API 配置文件](../reference/cli.md#configuration-file) 进行配置。本指南讨论了使用 `package.json` 指定项目依赖项来设置 LangGraph.js 应用程序以进行部署的基本步骤。

本指南基于 [此仓库](https://github.com/langchain-ai/langgraphjs-studio-starter)，你可以通过它了解更多关于如何设置 LangGraph 应用程序以进行部署的信息。

最终的仓库结构将如下所示：

```bash
my-app/
├── src # 所有项目代码都在这里
│   ├── utils # 可选的图工具
│   │   ├── tools.ts # 图的工具
│   │   ├── nodes.ts # 图的节点函数
│   │   └── state.ts # 图的状态定义
│   └── agent.ts # 构建图的代码
├── package.json # 包依赖项
├── .env # 环境变量
└── langgraph.json # LangGraph 的配置文件
```

在每一步之后，都会提供一个示例文件目录，以演示如何组织代码。

## 指定依赖项

依赖项可以在 `package.json` 中指定。如果未创建这些文件，则可以在 [LangGraph API 配置文件](#create-langgraph-api-config) 中稍后指定依赖项。

示例 `package.json` 文件：

```json
{
  "name": "langgraphjs-studio-starter",
  "packageManager": "yarn@1.22.22",
  "dependencies": {
    "@langchain/community": "^0.2.31",
    "@langchain/core": "^0.2.31",
    "@langchain/langgraph": "^0.2.0",
    "@langchain/openai": "^0.2.8"
  }
}
```

示例文件目录：

```bash
my-app/
└── package.json # 包依赖项
```

## 指定环境变量

环境变量可以选择性地在文件中指定（例如 `.env`）。请参阅 [环境变量参考](../reference/env_var.md) 以配置部署的其他变量。

示例 `.env` 文件：

```
MY_ENV_VAR_1=foo
MY_ENV_VAR_2=bar
OPENAI_API_KEY=key
TAVILY_API_KEY=key_2
```

示例文件目录：

```bash
my-app/
├── package.json
└── .env # 环境变量
```

## 定义图

实现你的图！图可以在单个文件或多个文件中定义。注意每个编译图的变量名称，这些变量名称将包含在 LangGraph 应用程序中。变量名称将在创建 [LangGraph API 配置文件](../reference/cli.md#configuration-file) 时使用。

以下是 `agent.ts` 的示例：

```ts
import type { AIMessage } from "@langchain/core/messages";
import { TavilySearchResults } from "@langchain/community/tools/tavily_search";
import { ChatOpenAI } from "@langchain/openai";

import { MessagesAnnotation, StateGraph } from "@langchain/langgraph";
import { ToolNode } from "@langchain/langgraph/prebuilt";

const tools = [
  new TavilySearchResults({ maxResults: 3, }),
];

// 定义调用模型的函数
async function callModel(
  state: typeof MessagesAnnotation.State,
) {
  /**
   * 调用驱动我们代理的 LLM。
   * 可以自定义提示、模型和其他逻辑！
   */
  const model = new ChatOpenAI({
    model: "gpt-4o",
  }).bindTools(tools);

  const response = await model.invoke([
    {
      role: "system",
      content: `You are a helpful assistant. The current date is ${new Date().getTime()}.`
    },
    ...state.messages
  ]);

  // MessagesAnnotation 支持返回单个消息或消息数组
  return { messages: response };
}

// 定义决定是否继续的函数
function routeModelOutput(state: typeof MessagesAnnotation.State) {
  const messages = state.messages;
  const lastMessage: AIMessage = messages[messages.length - 1];
  // 如果 LLM 正在调用工具，则路由到那里。
  if ((lastMessage?.tool_calls?.length ?? 0) > 0) {
    return "tools";
  }
  // 否则结束图。
  return "__end__";
}

// 定义一个新图。
// 有关定义自定义图状态的更多信息，请参阅 https://langchain-ai.github.io/langgraphjs/how-tos/define-state/#getting-started
const workflow = new StateGraph(MessagesAnnotation)
  // 定义我们将在其间循环的两个节点
  .addNode("callModel", callModel)
  .addNode("tools", new ToolNode(tools))
  // 将入口点设置为 `callModel`
  // 这意味着此节点是第一个被调用的节点
  .addEdge("__start__", "callModel")
  .addConditionalEdges(
    // 首先，我们定义边的源节点。我们使用 `callModel`。
    // 这意味着这些边是在 `callModel` 节点被调用后采取的。
    "callModel",
    // 接下来，我们传入确定目标节点的函数，
    // 该函数将在源节点被调用后调用。
    routeModelOutput,
    // 条件边可以路由到的可能目的地列表。
    // 这是条件边在 Studio 中正确渲染图所必需的
    [
      "tools",
      "__end__"
    ],
  )
  // 这意味着在 `tools` 被调用后，`callModel` 节点将被调用。
  .addEdge("tools", "callModel");

// 最后，我们编译它！
// 这将编译成一个可以调用和部署的图。
export const graph = workflow.compile();
```

!!! info "将 `CompiledGraph` 分配给变量"
    LangGraph Cloud 的构建过程要求将 `CompiledGraph` 对象分配给 JavaScript 模块的顶级变量（或者，你可以提供 [一个创建图的函数](./graph_rebuild.md)）。

示例文件目录：

```bash
my-app/
├── src # 所有项目代码都在这里
│   ├── utils # 可选的图工具
│   │   ├── tools.ts # 图的工具
│   │   ├── nodes.ts # 图的节点函数
│   │   └── state.ts # 图的状态定义
│   └── agent.ts # 构建图的代码
├── package.json # 包依赖项
├── .env # 环境变量
└── langgraph.json # LangGraph 的配置文件
```

## 创建 LangGraph API 配置

创建一个名为 `langgraph.json` 的 [LangGraph API 配置文件](../reference/cli.md#configuration-file)。有关配置文件中 JSON 对象每个键的详细说明，请参阅 [LangGraph CLI 参考](../reference/cli.md#configuration-file)。

示例 `langgraph.json` 文件：

```json
{
  "node_version": "20",
  "dockerfile_lines": [],
  "dependencies": ["."],
  "graphs": {
    "agent": "./src/agent.ts:graph"
  },
  "env": ".env"
}
```

请注意，`CompiledGraph` 的变量名称出现在顶级 `graphs` 键的每个子键值的末尾（即 `:<variable_name>`）。

!!! info "配置位置"
    LangGraph API 配置文件必须放置在与包含编译图和相关依赖项的 TypeScript 文件相同或更高层级的目录中。

## 下一步

在设置项目并将其放入 GitHub 仓库后，就可以 [部署你的应用程序](./cloud.md) 了。