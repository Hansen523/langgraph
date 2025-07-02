# llms.txt  

以下是采用[`llms.txt`](https://llmstxt.org/)格式的文档文件列表，具体包含`llms.txt`和`llms-full.txt`。这些文件让大语言模型（LLM）和智能体能够访问编程文档与API接口，尤其适用于集成开发环境（IDE）场景。

| Language Version | llms.txt                                                                                                   | llms-full.txt                                                                                                        |
|------------------|------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------|
| LangGraph Python | [https://langchain-ai.github.io/langgraph/llms.txt](https://langchain-ai.github.io/langgraph/llms.txt)     | [https://langchain-ai.github.io/langgraph/llms-full.txt](https://langchain-ai.github.io/langgraph/llms-full.txt)     |
| LangGraph JS     | [https://langchain-ai.github.io/langgraphjs/llms.txt](https://langchain-ai.github.io/langgraphjs/llms.txt) | [https://langchain-ai.github.io/langgraphjs/llms-full.txt](https://langchain-ai.github.io/langgraphjs/llms-full.txt) |
| LangChain Python | [https://python.langchain.com/llms.txt](https://python.langchain.com/llms.txt)                             | N/A                                                                                                                  |
| LangChain JS     | [https://js.langchain.com/llms.txt](https://js.langchain.com/llms.txt)                                     | N/A                                                                                                                  |

!!! info "输出内容校验提示"  
    即使获取了最新文档，当前最先进的模型仍可能生成错误代码。请将生成内容视为初稿，在上线前务必人工校验。

## `llms.txt`与`llms-full.txt`差异说明

- **`llms.txt`** 为索引文件，包含带简短描述的内容链接。LLM或智能体需访问这些链接获取详细信息。

- **`llms-full.txt`** 将所有详细内容直接整合在单一文件中，无需额外跳转。

使用`llms-full.txt`时需特别注意文件体积。对于大型文档，该文件可能超出LLM上下文窗口的承载上限。

## 通过MCP服务器使用`llms.txt`

截至2025年3月9日，IDE[尚未原生支持`llms.txt`](https://x.com/jeremyphoward/status/1902109312216129905?t=1eHFv2vdNdAckajnug0_Vw&s=19)。但您仍可通过MCP服务器有效使用：

### 🚀 使用`mcpdoc`服务器  
我们提供专为LLM和IDE设计的**MCP文档服务器**：  
👉 **[langchain-ai/mcpdoc GitHub仓库](https://github.com/langchain-ai/mcpdoc)**  

该服务器支持将`llms.txt`集成到**Cursor**、**Windsurf**、**Claude**及**Claude Code**等工具中。  

📘 仓库中提供详细的**配置指南和使用示例**。

## `llms-full.txt`使用方案

LangGraph的`llms-full.txt`通常包含数十万token，超过多数LLM的上下文限制。推荐使用方式：

1. **IDE环境（如Cursor/Windsurf）**：  
   - 添加为自定义文档，IDE将自动分块索引内容，实现检索增强生成（RAG）。

2. **非IDE环境**：  
   - 选用大上下文窗口的聊天模型  
   - 实施RAG策略高效管理文档查询