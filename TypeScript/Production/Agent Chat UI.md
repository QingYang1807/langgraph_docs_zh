以下是这篇 **Agent Chat UI** 文档的中文流畅翻译版👇

---

# Agent Chat UI（智能体聊天界面）

**LangChain** 提供了一个功能强大的预构建用户界面，可与通过 [`create_agent()`](/oss/javascript/langchain/agents) 创建的智能体无缝配合使用。
无论你是在本地运行，还是部署在云端（例如 [LangSmith](/langsmith/)），这个界面都能以极少的配置，为智能体提供丰富的交互体验。

---

## Agent Chat UI 简介

[Agent Chat UI](https://github.com/langchain-ai/agent-chat-ui) 是一个基于 **Next.js** 的应用，提供与任意 LangChain 智能体交互的对话界面。
它支持实时聊天、工具调用可视化，并具备高级功能，如时间回溯调试（Time Travel Debugging）与状态分叉（State Forking）。

该项目是开源的，你可以根据自身应用场景进行定制和扩展。

<Frame>
  <iframe className="w-full aspect-video rounded-xl" src="https://www.youtube.com/embed/lInrwVnZ83o?si=Uw66mPtCERJm0EjU" title="Agent Chat UI" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowFullScreen />
</Frame>

---

### 功能特性

#### 🧩 工具可视化（Tool Visualization）

Studio 会自动渲染工具调用及其结果，以直观的方式展示执行流程。

<Frame>
  <img src="https://mintcdn.com/langchain-5e9cc07a/zA84oCipUuW8ow2z/oss/images/studio_tools.gif" alt="Studio 中的工具可视化" width="1280" height="833" />
</Frame>

---

#### 🕒 时间回溯调试（Time-Travel Debugging）

可以浏览对话历史，并从任意时间点分叉继续执行。

<Frame>
  <img src="https://mintcdn.com/langchain-5e9cc07a/TCDks4pdsHdxWmuJ/oss/images/studio_fork.gif" alt="Studio 中的时间回溯调试" width="1280" height="833" />
</Frame>

---

#### 🧠 状态检查（State Inspection）

在任意执行阶段查看或修改智能体的内部状态。

<Frame>
  <img src="https://mintcdn.com/langchain-5e9cc07a/zA84oCipUuW8ow2z/oss/images/studio_state.gif" alt="Studio 中的状态检查" width="1280" height="833" />
</Frame>

---

#### 👥 人机协同（Human-in-the-Loop）

内置对“人工审阅与响应”流程的支持，可在运行时由人类接管或提供输入。

<Frame>
  <img src="https://mintcdn.com/langchain-5e9cc07a/TCDks4pdsHdxWmuJ/oss/images/studio_hitl.gif" alt="Studio 中的人机协同功能" width="1280" height="833" />
</Frame>

---

💡 **提示：**
你可以在 Agent Chat UI 中使用 **生成式 UI（Generative UI）**。
详情请参阅 [使用 LangGraph 实现生成式界面](/langsmith/generative-ui-react)。

---

## 快速开始（Quick Start）

最简单的启动方式是使用在线托管版本：

1. **访问 [Agent Chat UI](https://agentchat.vercel.app)**
2. **连接你的智能体** — 输入你的部署地址或本地服务器地址
3. **开始聊天！** UI 会自动检测并渲染工具调用和中断点

---

## 本地开发（Local Development）

如果你需要定制功能或在本地调试，可自行运行 Agent Chat UI：

<CodeGroup>

```bash
# 方式一：使用 npx 创建新项目
npx create-agent-chat-app --project-name my-chat-ui
cd my-chat-ui

# 安装依赖并启动
pnpm install
pnpm dev
```

```bash
# 方式二：直接克隆官方仓库
git clone https://github.com/langchain-ai/agent-chat-ui.git
cd agent-chat-ui

pnpm install
pnpm dev
```

</CodeGroup>

---

## 连接你的智能体

Agent Chat UI 可连接到 **本地智能体**（例如通过 [LangGraph 本地服务器](/oss/javascript/langgraph/studio#setup-local-langgraph-server)）
或 **已部署的智能体**（参考 [部署指南](/oss/javascript/langgraph/deploy)）。

启动界面后，需要填写以下信息：

1. **Graph ID（图标识）**：即你的智能体图名称，可在 `langgraph.json` 文件的 `graphs` 中找到。
2. **Deployment URL（部署地址）**：你的 LangGraph 服务端地址。

   * 本地开发：`http://localhost:2024`
   * 线上部署：填写实际访问 URL
3. **LangSmith API Key（可选）**：若连接 LangSmith 环境，可添加此密钥；本地模式则无需。

配置完成后，Agent Chat UI 将自动加载并展示智能体的所有中断线程（interrupted threads）。

---

💬 **提示：**
Agent Chat UI 内置对工具调用和结果消息的渲染支持。
若想自定义显示哪些消息，请参阅
👉 [在聊天中隐藏消息 (Hiding Messages in the Chat)](https://github.com/langchain-ai/agent-chat-ui?tab=readme-ov-file#hiding-messages-in-the-chat)

---

✏️ **[在 GitHub 上编辑此页面](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/ui.mdx)**

💻 **[将这些文档与 Claude、VSCode 等工具集成](/use-these-docs)**
通过 **MCP（Model Context Protocol）** 获取实时解答。

---
