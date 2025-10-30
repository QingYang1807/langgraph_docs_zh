# LangChain 概览

> 💡 **LangChain v1.0 正式发布！**
> 查看完整更新内容与迁移指南，请访问：[版本发布说明](/oss/python/releases/langchain-v1) 与 [迁移指南](/oss/python/migrate/langchain-v1)。
> 若在升级中遇到问题或有反馈，请[提交 Issue](https://github.com/langchain-ai/docs/issues/new?template=01-langchain.yml)。
> 想查看旧版 v0.x 文档，可前往 [归档内容](https://github.com/langchain-ai/langchain/tree/v0.3/docs/docs)。

---

## 🧠 什么是 LangChain？

**LangChain 是构建基于大语言模型（LLM）的智能体（Agent）和应用的最简单方式。**

只需不到 10 行代码，你就能快速连接 OpenAI、Anthropic、Google 等多种模型提供商，甚至更多模型参见 [providers 概览](/oss/python/integrations/providers/overview)。
LangChain 提供**预构建的智能体架构**与**统一的模型集成接口**，让你轻松在应用中接入并调用 LLM。

### ✨ 何时使用 LangChain？

* 想要**快速构建智能体或自治应用** → 用 **LangChain**
* 需要**更高的自定义编排能力、确定性流程与智能体逻辑混合** → 用 **[LangGraph](/oss/python/langgraph/overview)**

LangChain 的智能体实际上是 **构建在 LangGraph 之上** 的，继承了其强大的运行时能力：

* 持久化（Persistence）
* 流式输出（Streaming）
* 人机协作（Human-in-the-loop）
* 可恢复执行（Durable Execution）

> ⚙️ 但如果你只是想使用 LangChain 的智能体功能，不必深入了解 LangGraph 即可上手。

---

## 📦 安装 LangChain

```bash
pip install -U langchain
```

或使用 `uv`：

```bash
uv add langchain
```

---

## 🚀 快速创建一个智能体

```python
# 若使用 Anthropic 模型，需先安装：
# pip install -qU "langchain[anthropic]"

from langchain.agents import create_agent

def get_weather(city: str) -> str:
    """获取指定城市的天气"""
    return f"{city} 的天气永远晴朗 ☀️"

agent = create_agent(
    model="anthropic:claude-sonnet-4-5",
    tools=[get_weather],
    system_prompt="You are a helpful assistant.",
)

# 运行智能体
agent.invoke(
    {"messages": [{"role": "user", "content": "what is the weather in sf"}]}
)
```

---

## 🌟 核心优势

| 模块                        | 描述                                                                                                    |
| ------------------------- | ----------------------------------------------------------------------------------------------------- |
| 🔁 **标准化模型接口**            | 各厂商的模型 API 差异较大（参数、响应格式等）。LangChain 提供统一接口，轻松切换模型、避免厂商锁定。<br/>👉 [了解更多](/oss/python/langchain/models) |
| 🪄 **简单易用的智能体框架**         | 在 10 行代码内构建一个可工作的 Agent。同时保留足够的灵活性以支持复杂的上下文工程。<br/>👉 [了解更多](/oss/python/langchain/agents)            |
| 🧩 **基于 LangGraph 构建**    | LangChain 的智能体架构运行在 LangGraph 之上，天然支持持久化、流式、人工干预等高级特性。<br/>👉 [了解更多](/oss/python/langgraph/overview)  |
| 👁️ **集成 LangSmith 调试工具** | 可视化追踪智能体执行路径、状态变化、运行时指标等，便于深入分析和调试。<br/>👉 [了解更多](/langsmith/home)                                    |

---

✏️ [在 GitHub 上编辑本页源码](https://github.com/langchain-ai/docs/edit/main/src/oss/langchain/overview.mdx)

💻 [通过 MCP 将本文档接入 Claude、VSCode 等工具，实现实时问答](/use-these-docs)
