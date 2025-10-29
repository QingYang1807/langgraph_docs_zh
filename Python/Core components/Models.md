# 模型（Models）

[大型语言模型（LLM）](https://en.wikipedia.org/wiki/Large_language_model) 是强大的人工智能工具，能够像人类一样理解和生成文本。它们具备极高的通用性：无需针对每个任务单独训练，就可以编写内容、翻译语言、总结文本以及回答问题。

除了文本生成外，许多模型还支持以下功能：

* <Icon icon="hammer" size={16} /> **[工具调用（Tool calling）](#tool-calling)** —— 调用外部工具（例如数据库查询或 API 调用），并将结果融入模型输出。
* <Icon icon="shapes" size={16} /> **[结构化输出（Structured output）](#structured-outputs)** —— 模型输出会被约束在特定的格式中。
* <Icon icon="image" size={16} /> **[多模态（Multimodality）](#multimodal)** —— 处理和生成除文本外的数据，如图片、音频、视频等。
* <Icon icon="brain" size={16} /> **[推理（Reasoning）](#reasoning)** —— 模型通过多步推理过程得出结论。

模型是 [智能体（Agents）](/oss/python/langchain/agents) 的“推理引擎”。它驱动智能体的决策流程：决定调用哪些工具、如何解释结果，以及何时生成最终答案。

选择的模型质量与能力将直接影响智能体的可靠性与性能。不同模型擅长的方向不同——有的更善于遵循复杂指令，有的在结构化推理上表现更好，还有的能支持更大的上下文窗口，从而处理更多信息。

LangChain 提供标准化的模型接口，整合了多家提供商的接入方式，让你可以轻松试验、切换并找到最适合自身业务场景的模型。

<Info>
  各模型提供商的特定集成信息，请参见对应的 [聊天模型集成页面](/oss/python/integrations/chat)。
</Info>

---

## 基础用法

模型有两种主要使用方式：

1. **与智能体配合使用** —— 在创建 [Agent](/oss/python/langchain/agents#model) 时动态指定模型。
2. **独立调用** —— 直接调用模型（不依赖 Agent 框架）完成如文本生成、分类、抽取等任务。

同一模型接口可在两种场景下通用，这种一致性让你能从简单任务起步，逐步扩展到复杂的基于 Agent 的工作流。

### 初始化模型

在 LangChain 中初始化独立模型最简单的方法是使用 [`init_chat_model`](https://reference.langchain.com/python/langchain/models/#langchain.chat_models.init_chat_model)，从你选择的 [聊天模型提供商](/oss/python/integrations/chat) 中创建模型，例如：

（以下示例展示了 OpenAI、Anthropic、Azure、Google Gemini、AWS Bedrock 等多家提供商的用法）

```python
response = model.invoke("Why do parrots talk?")
```

更多详细说明（包括参数设置）请参阅 [`init_chat_model`](https://reference.langchain.com/python/langchain/models/#langchain.chat_models.init_chat_model)。

---

## 核心方法

* **Invoke（调用）**：输入消息，输出完整生成的响应。
* **Stream（流式输出）**：实时返回生成的内容片段。
* **Batch（批量处理）**：批量并行处理多个请求，提高效率。

<Info>
  除聊天模型外，LangChain 还支持嵌入模型与向量数据库等技术，详见 [集成页面](/oss/python/integrations/providers/overview)。
</Info>

---

## 参数说明

聊天模型接受一组参数，用于控制其行为。常见参数包括：

| 参数            | 类型     | 说明                        |
| ------------- | ------ | ------------------------- |
| `model`       | string | 要使用的具体模型名称或标识。            |
| `api_key`     | string | 与模型提供商认证的密钥。              |
| `temperature` | number | 控制输出的随机性。值越高越具创造性，越低越确定性。 |
| `timeout`     | number | 响应超时时间（秒）。                |
| `max_tokens`  | number | 限制响应的最大 token 数。          |
| `max_retries` | number | 请求失败后的最大重试次数。             |

这些参数可以通过 `**kwargs` 直接传入 `init_chat_model`：

```python
model = init_chat_model(
    "anthropic:claude-sonnet-4-5",
    temperature=0.7,
    timeout=30,
    max_tokens=1000,
)
```

<Info>
  不同提供商还可能支持额外参数，例如 `ChatOpenAI` 的 `use_responses_api`。
  所有模型参数请参考 [聊天模型集成页面](/oss/python/integrations/chat)。
</Info>

---

## 调用模型（Invocation）

模型必须被调用才能生成输出。主要有三种方式：

### 1. `invoke()` —— 单次调用

传入字符串或消息列表，获取完整响应。

### 2. `stream()` —— 流式输出

实时接收模型生成的内容，常用于改善长文本生成体验。

### 3. `batch()` —— 批量调用

并行处理多条输入，节省时间和成本。

---

## 工具调用（Tool Calling）

模型可调用外部工具执行任务，如访问数据库、调用 API、执行代码等。
可通过 `bind_tools()` 绑定自定义工具，使模型能在推理过程中调用它们。
绑定后模型的响应中会包含调用请求，若非使用 Agent，需要手动执行这些请求并将结果返回给模型。
（详细示例略）

---

## 结构化输出（Structured Outputs）

模型可以根据指定的结构化模式输出结果，方便解析与后续处理。LangChain 支持：

* **Pydantic 模型** —— 提供验证与描述功能。
* **TypedDict** —— Python 原生类型提示方式。
* **JSON Schema** —— 适合跨系统使用。

---

## 支持的模型

LangChain 支持主流模型提供商，包括 OpenAI、Anthropic、Google、Azure、AWS Bedrock 等，完整列表见 [集成概览页](/oss/python/integrations/providers/overview)。

---

## 高级主题

### 多模态（Multimodal）

部分模型可处理与生成非文本数据，如图像、音频、视频。
例如：

```python
response = model.invoke("Create a picture of a cat")
print(response.content_blocks)
# 输出中包含 {"type": "image", "base64": "..."}
```

---

### 推理（Reasoning）

新一代模型具备多步推理能力，能展示思考过程，帮助理解其结论来源。

---

### 本地模型（Local Models）

LangChain 支持在本地运行模型，如通过 [Ollama](/oss/python/integrations/chat/ollama) 部署，适用于隐私或成本敏感场景。

---

### 提示缓存（Prompt Caching）

部分提供商（如 OpenAI、Gemini、Anthropic）提供提示缓存功能，用于降低重复请求的成本与延迟。

---

### 服务器端工具使用

部分模型（如 GPT-4.1）支持在云端直接调用工具（例如网页搜索），无需手动执行。

---

### 速率限制（Rate Limiting）

通过 `rate_limiter` 参数或内置的 `InMemoryRateLimiter` 控制请求速率，防止触发限流。

---

### 基础 URL / 代理配置

可自定义 API 基础 URL 或设置代理，用于兼容第三方 OpenAI 接口或内网部署。

---

### 词元概率（Log Probabilities）

部分模型可返回 token 级别的概率信息，方便分析生成置信度。

---

### Token 使用统计

模型返回的消息中包含 token 使用统计信息，可用于成本与性能分析。

---

### 调用配置（Invocation Config）

可通过 `config` 参数传入运行时配置（如名称、标签、回调、元数据等），用于日志追踪、监控、并发控制等。

---

### 可配置模型（Configurable Models）

可创建在运行时切换模型或参数的动态模型实例，实现跨模型执行。

---

<Callout icon="pen-to-square" iconType="regular">
  [在 GitHub 上编辑本页面源码](https://github.com/langchain-ai/docs/edit/main/src/oss/langchain/models.mdx)
</Callout>

<Tip icon="terminal" iconType="regular">
  [通过 MCP 将本文档接入 Claude、VSCode 等工具，实现实时问答](/use-these-docs)
</Tip>