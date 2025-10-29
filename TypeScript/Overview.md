# LangGraph 概览

<Callout icon="bullhorn" color="#DFC5FE" iconType="regular">
  **LangGraph v1.0 现已发布！**

  如需查看完整的更改列表和代码升级说明，请参阅[发布说明](/oss/javascript/releases/langgraph-v1)和[迁移指南](/oss/javascript/migrate/langgraph-v1)。

  如果您遇到任何问题或有反馈，请[提出 issue](https://github.com/langchain-ai/docs/issues/new?template=02-langgraph.yml\&labels=langgraph,js/ts)，以便我们改进。要查看 v0.x 文档，请[前往归档内容](https://github.com/langchain-ai/langgraphjs/tree/main/docs/docs)。
</Callout>

LangGraph 是一个低级编排框架和运行时，用于构建、管理和部署长时间运行、有状态的应用程序，受到了包括 Klarna、Replit、Elastic 等众多塑造智能体未来的公司的信赖。

LangGraph 是一个非常低级的框架，完全专注于智能体的**编排**。在使用 LangGraph 之前，我们建议您先熟悉用于构建智能体的一些组件，从[模型](/oss/javascript/langchain/models)和[工具](/oss/javascript/langchain/tools)开始。

在文档中，我们会经常使用 [LangChain](/oss/javascript/langchain/overview) 组件来集成模型和工具，但您不需要使用 LangChain 也能使用 LangGraph。如果您是智能体开发的新手，或者想要一个更高级别的抽象，我们建议您使用 LangChain 的[智能体](/oss/javascript/langchain/agents)，它们为常见的 LLM 和工具调用循环提供了预构建的架构。

LangGraph 专注于对智能体编排至关重要的底层能力：持久化执行、流式处理、人机协作等。

## <Icon icon="download" size={20} /> 安装

<CodeGroup>
  ```bash npm theme={null}
  npm install @langchain/langgraph @langchain/core
  ```

  ```bash pnpm theme={null}
  pnpm add @langchain/langgraph @langchain/core
  ```

  ```bash yarn theme={null}
  yarn add @langchain/langgraph @langchain/core
  ```

  ```bash bun theme={null}
  bun add @langchain/langgraph @langchain/core
  ```
</CodeGroup>

然后，创建一个简单的 hello world 示例：

```typescript  theme={null}
import { MessagesAnnotation, StateGraph, START, END } from "@langchain/langgraph";

const mockLlm = (state: typeof MessagesAnnotation.State) => {
  return { messages: [{ role: "ai", content: "hello world" }] };
};

const graph = new StateGraph(MessagesAnnotation)
  .addNode("mock_llm", mockLlm)
  .addEdge(START, "mock_llm")
  .addEdge("mock_llm", END)
  .compile();

await graph.invoke({ messages: [{ role: "user", content: "hi!" }] });
```

## 核心优势

LangGraph 为*任何*长时间运行、有状态的工作流或智能体提供低级支持基础设施。LangGraph 不抽象提示词或架构，并提供以下核心优势：

* [持久化执行](/oss/javascript/langgraph/durable-execution)：构建能够在故障中持久保存并可以长时间运行的智能体，从中断处恢复执行。
* [人机协作](/oss/javascript/langgraph/interrupts)：通过在任何点检查和修改智能体状态来融入人工监督。
* [全面的记忆](/oss/javascript/concepts/memory)：创建具有短期工作记忆（用于持续推理）和跨会话长期记忆的有状态智能体。
* [使用 LangSmith 调试](/langsmith/home)：通过可视化工具深入了解复杂的智能体行为，这些工具可以跟踪执行路径、捕获状态转换并提供详细的运行时指标。
* [生产就绪的部署](/langsmith/deployments)：凭借专为处理有状态、长时间运行工作流的独特挑战而设计的可扩展基础设施，自信地部署复杂的智能体系统。

## LangGraph 生态系统

虽然 LangGraph 可以单独使用，但它也能与任何 LangChain 产品无缝集成，为开发者提供构建智能体的完整工具套件。为了改进您的 LLM 应用程序开发，可以将 LangGraph 与以下产品搭配使用：

* [LangSmith](http://www.langchain.com/langsmith) — 有助于智能体评估和可观测性。调试性能不佳的 LLM 应用运行，评估智能体轨迹，在生产环境中获得可见性，并随时间推移提高性能。
* [LangSmith](/langsmith/home) — 使用专为长时间运行、有状态工作流构建的部署平台，轻松部署和扩展智能体。在团队之间发现、重用、配置和共享智能体 — 并通过 [Studio](/langsmith/studio) 中的可视化原型快速迭代。
* [LangChain](/oss/javascript/langchain/overview) - 提供集成和可组合组件以简化 LLM 应用程序开发。包含构建在 LangGraph 之上的智能体抽象。

## 致谢

LangGraph 的灵感来源于 [Pregel](https://research.google/pubs/pub37252/) 和 [Apache Beam](https://beam.apache.org/)。其公共接口借鉴了 [NetworkX](https://networkx.org/documentation/latest/) 的设计。LangGraph 由 LangChain 的创建者 LangChain Inc. 构建，但可以独立于 LangChain 使用。

***

<Callout icon="pen-to-square" iconType="regular">
  [在 GitHub 上编辑此页面的源代码。](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/overview.mdx)
</Callout>

<Tip icon="terminal" iconType="regular">
  [以编程方式连接这些文档](/use-these-docs)，通过 MCP 连接到 Claude、VSCode 等，以获得实时答案。
</Tip>