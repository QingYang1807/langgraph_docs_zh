# 学习

> 教程、概念指南和资源，帮助您入门。

在文档的**学习**部分，您将找到一系列教程、概念概述和额外资源，帮助您使用 LangChain 和 LangGraph 构建强大的应用程序。

## 用例

以下是按框架组织的常见用例教程。

### LangChain

[LangChain](/oss/python/langchain/overview) [代理](/oss/python/langchain/agents) 实现让大多数用例都能轻松上手。

<Card title="语义搜索" icon="magnifying-glass" href="/oss/python/langchain/knowledge-base" horizontal>
  使用 LangChain 组件构建一个基于 PDF 的语义搜索引擎。
</Card>

<Card title="RAG 代理" icon="user-magnifying-glass" href="/oss/python/langchain/rag" horizontal>
  创建一个检索增强生成(RAG)代理。
</Card>

<Card title="SQL 代理" icon="database" href="/oss/python/langchain/sql-agent" horizontal>
  构建一个具有人工循环审查功能的 SQL 代理来与数据库交互。
</Card>

<Card title="主管代理" icon="sitemap" href="/oss/python/langchain/supervisor" horizontal>
  构建一个委派给子代理的个人助理。
</Card>

### LangGraph

LangChain 的 [代理](/oss/python/langchain/agents) 实现使用 [LangGraph](/oss/python/langgraph/overview) 原语。
如果需要更深入的定制，可以直接在 LangGraph 中实现代理。

<Card title="自定义 RAG 代理" icon="user-magnifying-glass" href="/oss/python/langgraph/agentic-rag" horizontal>
  使用 LangGraph 原语构建 RAG 代理，实现细粒度控制。
</Card>

<Card title="自定义 SQL 代理" icon="database" href="/oss/python/langgraph/sql-agent" horizontal>
  直接在 LangGraph 中实现 SQL 代理，获得最大灵活性。
</Card>

## 概述

这些指南解释了 LangChain 和 LangGraph 的核心概念和基础 API。

<Card title="记忆" icon="brain" href="/oss/python/concepts/memory" horizontal>
  理解线程内和跨线程交互的持久性。
</Card>

<Card title="上下文工程" icon="book-open" href="/oss/python/concepts/context" horizontal>
  学习为 AI 应用程序提供正确信息和工具以完成任务的方法。
</Card>

<Card title="图 API" icon="chart-network" href="/oss/python/langgraph/graph-api" horizontal>
  探索 LangGraph 的声明式图构建 API。
</Card>

<Card title="函数式 API" icon="code" href="/oss/python/langgraph/functional-api" horizontal>
  将代理构建为单个函数。
</Card>

## 其他资源

<Card title="LangChain 学院" icon="graduation-cap" href="https://academy.langchain.com/" horizontal>
  课程和练习，提升您的 LangChain 技能。
</Card>

<Card title="案例研究" icon="screen-users" href="/oss/python/langgraph/case-studies" horizontal>
    了解团队如何在生产中使用 LangChain 和 LangGraph。
</Card>

***

<Callout icon="pen-to-square" iconType="regular">
  [在 GitHub 上编辑此页面的源代码。](https://github.com/langchain-ai/docs/edit/main/src/oss/learn.mdx)
</Callout>

<Tip icon="terminal" iconType="regular">
    通过 MCP 将这些文档以编程方式连接到 Claude、VSCode 等，获取实时答案。
</Tip>