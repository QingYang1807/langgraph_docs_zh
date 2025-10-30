# LangGraph 概述


**LangGraph v1.0 现已发布！**

如需查看完整的变更列表和升级代码的说明，请参阅[发布说明](/oss/python/releases/langgraph-v1)和[迁移指南](/oss/python/migrate/langgraph-v1)。

如果您遇到任何问题或有反馈意见，请[创建问题](https://github.com/langchain-ai/docs/issues/new?template=02-langgraph.yml\&labels=langgraph,python)，以便我们改进。要查看 v0.x 文档，请[前往归档内容](https://github.com/langchain-ai/langgraph/tree/main/docs/docs)。

受到塑造代理未来的公司（包括 Klarna、Replit、Elastic 等）的信赖，LangGraph 是一个用于构建、管理和部署长时间运行、有状态代理的低级编排框架和运行时。

LangGraph 是非常低级的，完全专注于代理**编排**。在使用 LangGraph 之前，我们建议您熟悉一些用于构建代理的组件，从[模型](/oss/python/langchain/models)和[工具](/oss/python/langchain/tools)开始。

我们将在整个文档中经常使用 [LangChain](/oss/python/langchain/overview) 组件来集成模型和工具，但您不需要使用 LangChain 来使用 LangGraph。如果您刚刚开始使用代理或想要更高级别的抽象，我们建议您使用 LangChain 的[代理](/oss/python/langchain/agents)，它们为常见的 LLM 和工具调用循环提供预构建的架构。

LangGraph 专注于代理编排重要的底层能力：持久执行、流式传输、人工干预等。

## 安装


```bash
pip install -U langgraph
```

```bash
uv add langgraph
```

然后，创建一个简单的 hello world 示例：

```python
from langgraph.graph import StateGraph, MessagesState, START, END

def mock_llm(state: MessagesState):
    return {"messages": [{"role": "ai", "content": "hello world"}]}

graph = StateGraph(MessagesState)
graph.add_node(mock_llm)
graph.add_edge(START, "mock_llm")
graph.add_edge("mock_llm", END)
graph = graph.compile()

graph.invoke({"messages": [{"role": "user", "content": "hi!"}]})
```

## 核心优势

LangGraph 为*任何*长时间运行、有状态的工作流或代理提供低级支持基础设施。LangGraph 不抽象提示或架构，并提供以下核心优势：

* [持久执行](/oss/python/langgraph/durable-execution)：构建能够持久运行并在失败中恢复的代理，可以长时间运行，并从中断处恢复。
* [人工干预](/oss/python/langgraph/interrupts)：通过在任何点检查和修改代理状态来融入人工监督。
* [全面记忆](/oss/python/concepts/memory)：创建具有短期工作记忆（用于持续推理）和长期记忆（跨会话）的有状态代理。
* [使用 LangSmith 进行调试](/langsmith/home)：通过可视化工具深入了解复杂的代理行为，这些工具可以跟踪执行路径、捕获状态转换并提供详细的运行时指标。
* [生产就绪的部署](/langsmith/deployments)：自信地部署复杂的代理系统，使用专为处理有状态、长时间运行工作流的独特挑战而设计的可扩展基础设施。

## LangGraph 生态系统

虽然 LangGraph 可以单独使用，但它也与任何 LangChain 产品无缝集成，为开发者提供构建代理的完整工具套件。为了改进您的 LLM 应用程序开发，将 LangGraph 与以下产品配对使用：

* [LangSmith](http://www.langchain.com/langsmith) — 有助于代理评估和可观察性。调试表现不佳的 LLM 应用程序运行，评估代理轨迹，获得生产环境可见性，并随时间推移改进性能。
* [LangSmith](/langsmith/home) — 使用专为长时间运行、有状态工作流构建的部署平台轻松部署和扩展代理。在团队之间发现、重用、配置和共享代理 — 并通过 [Studio](/langsmith/studio) 中的可视化原型快速迭代。
* [LangChain](/oss/python/langchain/overview) - 提供集成和可组合组件来简化 LLM 应用程序开发。包含构建在 LangGraph 之上的代理抽象。

## 致谢

LangGraph 的灵感来自 [Pregel](https://research.google/pubs/pub37252/) 和 [Apache Beam](https://beam.apache.org/)。其公共接口借鉴了 [NetworkX](https://networkx.org/documentation/latest/) 的设计。LangGraph 由 LangChain Inc（LangChain 的创建者）构建，但可以在不使用 LangChain 的情况下使用。

***

[在 GitHub 上编辑此页面的源代码。](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/overview.mdx)

[以编程方式连接这些文档](/use-these-docs)到 Claude、VSCode 和更多工具，通过 MCP 获取实时答案。