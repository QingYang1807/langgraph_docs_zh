# LangChain 哲学（Philosophy 中文版）

> **LangChain 的使命：**
> 让开发者以最简单的方式构建 LLM 应用，并且具备足够的灵活性和生产级可靠性。

---

## 🌱 核心理念

LangChain 的设计基于以下信念：

1. **大语言模型（LLMs）是革命性技术。**
   它们带来了理解与生成自然语言的全新方式。

2. **LLMs 与外部数据结合后更强大。**
   将模型与数据库、API、知识库结合，可显著增强智能体能力。

3. **未来的应用将更“智能体化”（Agentic）。**
   应用不再只是被动响应，而是具备自主决策与任务规划的能力。

4. **我们仍处在早期阶段。**
   Agent 应用的研发与部署仍面临可靠性与可控性挑战。

5. **从原型到生产，是一条坎坷的路。**
   虽然快速搭建一个 Demo 很容易，但让智能体稳定可靠地运行仍非常困难。

---

## 🎯 双核心目标

<Steps>
  <Step title="1️⃣ 让开发者轻松使用最好的模型">
    各家模型厂商（如 OpenAI、Anthropic、Google）API 接口不一致，参数、消息结构都不同。
    LangChain 的目标是**标准化模型输入与输出接口**，让开发者可无痛切换最先进的模型，
    避免供应商锁定（vendor lock-in）。
  </Step>

  <Step title="2️⃣ 让模型不仅仅是“生成文本”">
    模型应该能**编排复杂的流程**，与数据和工具交互。
    LangChain 提供 [Tools 模块](/oss/python/langchain/tools)，
    让 LLM 动态调用外部函数、访问结构化或非结构化数据，真正成为**主动的智能体**。
  </Step>
</Steps>

---

## 🕰️ 历史演进

LangChain 随着 LLM 技术的迭代而不断进化。以下是其关键时间线：

---

### 🧩 **2022-10-24：v0.0.1**

LangChain 正式发布（早于 ChatGPT 一个月）。

首版包含两大组件：

* **LLM 抽象层**（统一不同模型的接口）
* **Chains（链式调用）**：为常见任务（如 RAG）提供固定的调用流程。

> “LangChain” 名字来自 *Language* + *Chain*。

---

### 🤖 **2022-12**

引入首批 **通用 Agent（智能体）**，基于 [ReAct 论文](https://arxiv.org/abs/2210.03629)。

* 模型生成 JSON 表示工具调用；
* 框架解析 JSON 决定执行的操作；
* 实现了“推理 + 行动”的闭环。

---

### 💬 **2023-01**

OpenAI 推出 **ChatCompletion API**。

* 输入结构从单字符串变为消息列表；
* LangChain 更新为“多轮对话”格式；
* 同期发布 **JavaScript 版 LangChain**，方便前端与应用开发者使用。

---

### 🧱 **2023-02**

**LangChain Inc. 公司成立**，目标是：

> “让智能体无处不在”。

LangChain 成为开源核心，同时启动商业化产品研发。

---

### 🔧 **2023-03**

OpenAI 推出 **Function Calling API**。

* 模型可以显式返回工具调用结构；
* LangChain 更新为优先使用 Function Calling 而非 JSON 解析。

---

### 🧠 **2023-06**

发布 **LangSmith（闭源）**：
提供智能体的可观测性与评估能力（Observability & Evals）。
LangChain 无缝集成 LangSmith，以帮助开发者构建更可靠的智能体。

---

### 🏗️ **2024-01 — v0.1.0**

LangChain 发布 **首个稳定版本 0.1.0**。

行业进入从原型到生产阶段，LangChain 重点转向**稳定性与可靠性**。

---

### 🔀 **2024-02**

发布 **LangGraph** —— 开源智能体编排框架。

LangGraph 提供了 LangChain 之前缺失的底层能力：

* 可控的节点式图结构执行
* 流式输出（Streaming）
* 可持久化执行（Durable Execution）
* 短期记忆（Memory）
* 人类干预机制（Human-in-the-loop）

---

### 🌍 **2024-06**

LangChain 集成数突破 **700+**。

为保持核心简洁：

* 核心集成拆分为独立包；
* 其余迁移至 `langchain-community`。

---

### ⚙️ **2024-10**

**LangGraph 成为构建复杂 AI 应用的首选方式。**

* Agent 与 Chain 等高层抽象逐步弃用；
* LangChain 迁移到基于 LangGraph 的统一 Agent 接口；
* 仍保留一个高层 Agent API（兼容旧版 ReAct 接口）。

---

### 🖼️ **2025-04**

模型 API 进入 **多模态时代（Multimodal）**。

* 支持输入文件、图片、视频等；
* LangChain-Core 更新消息格式以统一多模态输入。

---

### 🚀 **2025-10-20 — v1.0.0**

**LangChain 正式发布 1.0 版本！**

主要更新：

1. **全新 Agent 架构**

   * 所有旧版 Chains / Agents 被统一为一个高层 Agent 抽象；
   * 新 Agent 基于 LangGraph 构建；
   * 若需继续使用旧版，可安装 `langchain-classic`。

2. **标准化消息内容格式**

   * 模型输出从简单字符串进化为结构化内容块；
   * 支持 reasoning traces、citations、server-side tools；
   * LangChain 对所有 provider 输出进行统一封装。

---

## ✨ 总结：LangChain 的进化逻辑

| 阶段   | 关键变化                           | 目标            |
| ---- | ------------------------------ | ------------- |
| 2022 | LLM 抽象层 + 链式调用                 | 降低入门门槛        |
| 2023 | ReAct Agent + Function Calling | 实现智能体化        |
| 2024 | LangGraph + LangSmith          | 提升可控性与可靠性     |
| 2025 | LangChain v1 + 统一消息格式          | 构建生产级多模态智能体生态 |

---

> **LangChain 的哲学一句话总结：**
> “让开发者既能快速原型，也能稳定上线。”
> 从 *语言链* 到 *智能体图*，LangChain 代表了 AI 应用从**文本生成**走向**智能决策**的演进。
