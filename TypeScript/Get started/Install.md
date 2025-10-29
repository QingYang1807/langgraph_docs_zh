# 安装 LangGraph

要安装基础的 LangGraph 包：

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

使用 LangGraph 时，您通常需要访问大语言模型（LLMs）并定义工具。
您可以按照自己认为合适的方式来实现这一点。

一种实现方式（我们在文档中将会使用）是使用 [LangChain](/oss/javascript/langchain/overview)。

安装 LangChain：

<CodeGroup>
  ```bash npm theme={null}
  npm install langchain
  ```

  ```bash pnpm theme={null}
  pnpm add langchain
  ```

  ```bash yarn theme={null}
  yarn add langchain
  ```

  ```bash bun theme={null}
  bun add langchain
  ```
</CodeGroup>

要使用特定的 LLM 提供商包，您需要单独安装它们。

请参阅 [集成](/oss/javascript/integrations/providers/overview) 页面获取特定提供商的安装说明。

***

<Callout icon="pen-to-square" iconType="regular">
  [在 GitHub 上编辑此页面的源代码。](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/install.mdx)
</Callout>

<Tip icon="terminal" iconType="regular">
  [以编程方式连接这些文档](/use-these-docs)，通过 MCP 连接到 Claude、VSCode 等，获取实时答案。
</Tip>