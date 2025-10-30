# v1 版本的新特性

**LangGraph v1 是专注于稳定性的代理运行时版本。** 它保持核心图 API 和执行模型不变，同时优化了类型安全、文档和开发者体验。

它设计为与 [LangChain v1](/oss/python/releases/langchain-v1)（其 `create_agent` 构建在 LangGraph 之上）紧密协作，因此您可以从高层次开始，并在需要时降级到细粒度控制。

<CardGroup cols={1}>
  <Card title="稳定的核心 API" icon="diagram-project">
    图原语（状态、节点、边）和执行/运行时模型保持不变，使升级变得简单直接。
  </Card>

  <Card title="默认的可靠性" icon="database">
    通过检查点、持久化、流式处理和人机交互实现的持久执行仍然是首要功能。
  </Card>

  <Card title="与 LangChain v1 无缝集成" icon="link">
    LangChain 的 `create_agent` 在 LangGraph 上运行。使用 LangChain 快速启动；使用 LangGraph 进行自定义编排。
  </Card>
</CardGroup>

要升级，

<CodeGroup>
  ```bash pip theme={null}
  pip install -U langgraph
  ```

  ```bash uv theme={null}
  uv add langgraph
  ```
</CodeGroup>

## `create_react_agent` 的弃用

LangGraph 的预构建 [`create_react_agent`](https://reference.langchain.com/python/langgraph/prebuilt/#langgraph.prebuilt.chat_agent_executor.create_react_agent) 已被弃用，取而代之的是 LangChain 的 [`create_agent`](https://reference.langchain.com/python/langchain/agents/#langchain.agents.create_agent)。它提供了更简单的接口，并通过引入中间件提供更大的定制潜力。

* 有关新的 [`create_agent`](https://reference.langchain.com/python/langchain/agents/#langchain.agents.create_agent) API 的信息，请参阅 [LangChain v1 发布说明](/oss/python/releases/langchain-v1#create-agent)。
* 有关从 [`create_react_agent`](https://reference.langchain.com/python/langgraph/prebuilt/#langgraph.prebuilt.chat_agent_executor.create_react_agent) 迁移到 [`create_agent`](https://reference.langchain.com/python/langchain/agents/#langchain.agents.create_agent) 的信息，请参阅 [LangChain v1 迁移指南](/oss/python/migrate/langchain-v1#create-agent)。

## 报告问题

请使用 [`'v1'` 标签](https://github.com/langchain-ai/langgraph/issues?q=state%3Aopen%20label%3Av1)在 [GitHub](https://github.com/langchain-ai/langgraph/issues) 上报告发现的任何 1.0 版本问题。

## 其他资源

<CardGroup cols={3}>
  <Card title="LangGraph 1.0" icon="rocket" href="https://blog.langchain.com/langchain-langchain-1-0-alpha-releases/">
    阅读发布公告
  </Card>

  <Card title="概述" icon="book" href="/oss/python/langgraph/overview" arrow>
    什么是 LangGraph 以及何时使用它
  </Card>

  <Card title="图 API" icon="diagram-project" href="/oss/python/langgraph/graph-api" arrow>
    使用状态、节点和边构建图
  </Card>

  <Card title="LangChain 代理" icon="robot" href="/oss/python/langchain/agents" arrow>
    构建在 LangGraph 之上的高级代理
  </Card>

  <Card title="迁移指南" icon="arrow-right-arrow-left" href="/oss/python/migrate/langgraph-v1" arrow>
    如何迁移到 LangGraph v1
  </Card>

  <Card title="GitHub" icon="github" href="https://github.com/langchain-ai/langgraph">
    报告问题或做出贡献
  </Card>
</CardGroup>

## 另请参阅

* [版本控制](/oss/python/versioning) - 理解版本号
* [发布策略](/oss/python/release-policy) - 详细的发布策略

***

<Callout icon="pen-to-square" iconType="regular">
  [在 GitHub 上编辑此页面的源代码。](https://github.com/langchain-ai/docs/edit/main/src/oss/python/releases/langgraph-v1.mdx)
</Callout>

<Tip icon="terminal" iconType="regular">
  [以编程方式连接这些文档](/use-these-docs) 到 Claude、VSCode 等工具，通过 MCP 获取实时答案。
</Tip>