# Agent Chat UI

LangChain 提供了一个强大的预构建用户界面，可与使用 [`create_agent`](/oss/python/langchain/agents) 创建的代理无缝协作。无论您是在本地运行还是在已部署的环境中（如 [LangSmith](/langsmith/)）运行，此界面都旨在为您的代理提供丰富、交互式的体验，且只需最少的设置。

## Agent Chat UI

[Agent Chat UI](https://github.com/langchain-ai/agent-chat-ui) 是一个 Next.js 应用程序，为与任何 LangChain 代理交互提供会话界面。它支持实时聊天、工具可视化以及时间旅行调试和状态分叉等高级功能。

Agent Chat UI 是开源的，可以根据您的应用程序需求进行定制。

<Frame>
  <iframe className="w-full aspect-video rounded-xl" src="https://www.youtube.com/embed/lInrwVnZ83o?si=Uw66mPtCERJm0EjU" title="Agent Chat UI" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowFullScreen />
</Frame>

### 功能特性

<Accordion title="工具可视化">
  Studio 会以直观的界面自动呈现工具调用和结果。

  <Frame>
        <img src="https://mintcdn.com/langchain-5e9cc07a/zA84oCipUuW8ow2z/oss/images/studio_tools.gif?s=64e762e917f092960472b61a862a81cb" alt="Studio 中的工具可视化" data-og-width="1280" width="1280" data-og-height="833" height="833" data-path="oss/images/studio_tools.gif" data-optimize="true" data-opv="3" />
  </Frame>
</Accordion>

<Accordion title="时间旅行调试">
  浏览对话历史并从任意点进行分叉

  <Frame>
        <img src="https://mintcdn.com/langchain-5e9cc07a/TCDks4pdsHdxWmuJ/oss/images/studio_fork.gif?s=0bb5a397d4b2ed3ff8ec62b9d0f92e3e" alt="Studio 中的时间旅行调试" data-og-width="1280" width="1280" data-og-height="833" height="833" data-path="oss/images/studio_fork.gif" data-optimize="true" data-opv="3" />
  </Frame>
</Accordion>

<Accordion title="状态检查">
  在执行过程中的任何点查看和修改代理状态

  <Frame>
        <img src="https://mintcdn.com/langchain-5e9cc07a/zA84oCipUuW8ow2z/oss/images/studio_state.gif?s=908d69765b0655cb532620c6e0fa96c8" alt="Studio 中的状态检查" data-og-width="1280" width="1280" data-og-height="833" height="833" data-path="oss/images/studio_state.gif" data-optimize="true" data-opv="3" />
  </Frame>
</Accordion>

<Accordion title="人在环路中">
  内置支持审查和响应代理请求的功能

  <Frame>
        <img src="https://mintcdn.com/langchain-5e9cc07a/TCDks4pdsHdxWmuJ/oss/images/studio_hitl.gif?s=ce7ce6378caf4db29ea6062b9aff0220" alt="Studio 中的人环路" data-og-width="1280" width="1280" data-og-height="833" height="833" data-path="oss/images/studio_hitl.gif" data-optimize="true" data-opv="3" />
  </Frame>
</Accordion>

<Tip>
  您可以在 Agent Chat UI 中使用生成式 UI。有关更多信息，请参阅 [使用 LangGraph 实现生成式用户界面](/langsmith/generative-ui-react)。
</Tip>

### 快速开始

最快的方式是使用托管版本：

1. **访问 [Agent Chat UI](https://agentchat.vercel.app)**
2. **连接您的代理**，输入您的部署 URL 或本地服务器地址
3. **开始聊天** - UI 将自动检测并呈现工具调用和中断

### 本地开发

如需定制或本地开发，您可以在本地运行 Agent Chat UI：

<CodeGroup>
  ```bash 使用 npx theme={null}
  # 创建新的 Agent Chat UI 项目
  npx create-agent-chat-app --project-name my-chat-ui
  cd my-chat-ui

  # 安装依赖并启动
  pnpm install
  pnpm dev
  ```

  ```bash 克隆仓库 theme={null}
  # 克隆仓库
  git clone https://github.com/langchain-ai/agent-chat-ui.git
  cd agent-chat-ui

  # 安装依赖并启动
  pnpm install
  pnpm dev
  ```
</CodeGroup>

### 连接到您的代理

Agent Chat UI 可以连接到 [本地](/oss/python/langchain/studio#setup-local-langgraph-server) 和 [已部署的代理](/oss/python/langchain/deploy)。

启动 Agent Chat UI 后，您需要配置它以连接到您的代理：

1. **图 ID**：输入您的图名称（在您的 `langgraph.json` 文件中的 `graphs` 下查找）
2. **部署 URL**：您的 LangGraph 服务器端点（例如，本地开发为 `http://localhost:2024`，或您已部署的代理的 URL）
3. **LangSmith API 密钥（可选）**：添加您的 LangSmith API 密钥（如果您使用本地 LangGraph 服务器，则不需要）

配置完成后，Agent Chat UI 将自动获取并显示代理的任何中断线程。

<Tip>
  Agent Chat UI 提供开箱即用的工具调用和工具结果消息渲染功能。如需自定义显示哪些消息，请参阅 [聊天中隐藏消息](https://github.com/langchain-ai/agent-chat-ui?tab=readme-ov-file#hiding-messages-in-the-chat)。
</Tip>

***

<Callout icon="pen-to-square" iconType="regular">
  [在 GitHub 上编辑此页面的源代码。](https://github.com/langchain-ai/docs/edit/main/src/oss/langchain/ui.mdx)
</Callout>

<Tip icon="terminal" iconType="regular">
  [通过 MCP 以编程方式连接这些文档](/use-these-docs) 到 Claude、VSCode 等，以获取实时答案。
</Tip>