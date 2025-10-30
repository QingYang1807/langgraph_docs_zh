# LangChain v1 新特性

**LangChain v1 是面向生产环境的全新智能体（Agent）开发核心版本。**
它以稳定、简洁、模块化为目标，围绕以下三大改进进行了全面升级：

---

## 🧩 三大核心改进

| 模块                            | 描述                                                                  |
| ----------------------------- | ------------------------------------------------------------------- |
| 🤖 **`create_agent`**         | LangChain 构建智能体的新标准接口，取代旧版 `langgraph.prebuilt.create_react_agent`。 |
| 🧱 **标准化内容块（content blocks）** | 新增 `content_blocks` 属性，实现跨厂商统一的消息内容访问方式。                            |
| 🧭 **精简命名空间（namespace）**      | `langchain` 命名空间专注于智能体核心功能，其余旧功能迁移至 `langchain-classic`。            |

---

## ⚙️ 升级到最新版本

```bash
pip install -U langchain
```

或使用 `uv`：

```bash
uv add langchain
```

查看完整迁移指南 👉 [LangChain v1 迁移指南](/oss/python/migrate/langchain-v1)

---

## 🧠 新一代智能体构建：`create_agent`

[`create_agent`](https://reference.langchain.com/python/langchain/agents/#langchain.agents.create_agent)
是 LangChain v1 的核心方法，用于快速构建可扩展、可定制的智能体。

相比旧版 `create_react_agent`，它的接口更简洁、功能更强，并支持通过 **Middleware 中间件** 进行深度定制。

```python
from langchain.agents import create_agent

agent = create_agent(
    model="anthropic:claude-sonnet-4-5",
    tools=[search_web, analyze_data, send_email],
    system_prompt="You are a helpful research assistant."
)

result = agent.invoke({
    "messages": [
        {"role": "user", "content": "Research AI safety trends"}
    ]
})
```

该智能体内部执行循环如下图所示：

🌀 **核心执行循环（Agent Loop）**

1. 模型生成思考与决策；
2. 选择并调用工具；
3. 若无更多工具调用，则结束执行。

---

## 🧩 中间件（Middleware）

Middleware 是 `create_agent` 的核心增强点，提供了强大的扩展机制。

它可用于：

* 动态提示词生成（Context Engineering）
* 对话总结与裁剪
* 工具权限控制
* 人机协作审批
* 输出安全校验与防护

---

### 🏗️ 内置中间件（Prebuilt Middleware）

LangChain 提供多种常用中间件：

| 中间件                        | 功能                 |
| -------------------------- | ------------------ |
| `PIIMiddleware`            | 屏蔽或替换敏感信息（如邮箱、手机号） |
| `SummarizationMiddleware`  | 自动摘要长对话内容          |
| `HumanInTheLoopMiddleware` | 在执行敏感操作前暂停等待人工批准   |

示例：

```python
from langchain.agents import create_agent
from langchain.agents.middleware import (
    PIIMiddleware,
    SummarizationMiddleware,
    HumanInTheLoopMiddleware
)

agent = create_agent(
    model="anthropic:claude-sonnet-4-5",
    tools=[read_email, send_email],
    middleware=[
        PIIMiddleware("email", strategy="redact", apply_to_input=True),
        PIIMiddleware("phone_number", strategy="block"),
        SummarizationMiddleware(model="anthropic:claude-sonnet-4-5", max_tokens_before_summary=500),
        HumanInTheLoopMiddleware(interrupt_on={"send_email": {"allowed_decisions": ["approve", "edit", "reject"]}})
    ]
)
```

---

### 🔧 自定义中间件（Custom Middleware）

继承 `AgentMiddleware` 类即可编写自定义中间件，
可在智能体生命周期的任意阶段插入逻辑：

| Hook              | 触发时机       | 应用场景            |
| ----------------- | ---------- | --------------- |
| `before_agent`    | 调用 Agent 前 | 载入记忆、输入校验       |
| `before_model`    | 每次 LLM 调用前 | 修改 prompt、裁剪上下文 |
| `wrap_model_call` | 包裹模型调用     | 修改输入输出、调试注入     |
| `wrap_tool_call`  | 包裹工具调用     | 监控与拦截工具执行       |
| `after_model`     | LLM 调用后    | 输出过滤、应用防护规则     |
| `after_agent`     | Agent 结束后  | 存储结果、清理状态       |

示例：

```python
from dataclasses import dataclass
from typing import Callable
from langchain.agents.middleware import AgentMiddleware, ModelRequest
from langchain.agents.middleware.types import ModelResponse
from langchain_openai import ChatOpenAI

@dataclass
class Context:
    user_expertise: str = "beginner"

class ExpertiseMiddleware(AgentMiddleware):
    def wrap_model_call(self, request: ModelRequest, handler: Callable[[ModelRequest], ModelResponse]) -> ModelResponse:
        user_level = request.runtime.context.user_expertise
        if user_level == "expert":
            request.model = ChatOpenAI(model="openai:gpt-5")
            request.tools = [advanced_search, data_analysis]
        else:
            request.model = ChatOpenAI(model="openai:gpt-5-nano")
            request.tools = [simple_search, basic_calculator]
        return handler(request)

agent = create_agent(
    model="anthropic:claude-sonnet-4-5",
    tools=[simple_search, advanced_search, data_analysis],
    middleware=[ExpertiseMiddleware()],
    context_schema=Context
)
```

---

## ⚙️ LangGraph 内置能力

`create_agent` 基于 [LangGraph](/oss/python/langgraph)，因此自动继承以下运行时特性：

| 特性                            | 说明                       |
| ----------------------------- | ------------------------ |
| 💾 **持久化（Persistence）**       | 对话自动保存与恢复（Checkpoint 支持） |
| 🌊 **流式输出（Streaming）**        | 支持实时 token、工具调用和推理追踪     |
| ✋ **人工干预（Human-in-the-loop）** | 支持中断并等待人工确认              |
| 🕰️ **时间回溯（Time travel）**     | 可回放任意时刻对话并重演路径           |

> 🚀 无需额外配置，即可开箱使用。

---

## 🧮 结构化输出（Structured Output）

`create_agent` 在结构化输出上也进行了全面改进：

* 内置于主循环中，无需额外 LLM 调用；
* 支持模型端的结构化输出生成；
* 显著降低成本与延迟。

```python
from langchain.agents import create_agent
from langchain.agents.structured_output import ToolStrategy
from pydantic import BaseModel

class Weather(BaseModel):
    temperature: float
    condition: str

def weather_tool(city: str) -> str:
    return f"it's sunny and 70 degrees in {city}"

agent = create_agent(
    "openai:gpt-4o-mini",
    tools=[weather_tool],
    response_format=ToolStrategy(Weather)
)

result = agent.invoke({
    "messages": [{"role": "user", "content": "What's the weather in SF?"}]
})

print(result["structured_response"])
# 输出：Weather(temperature=70.0, condition='sunny')
```

> 错误处理可通过 `ToolStrategy(handle_errors=...)` 参数控制。

---

## 🧱 标准化内容块（Content Blocks）

新属性 [`content_blocks`](https://reference.langchain.com/python/langchain_core/language_models/#langchain_core.messages.BaseMessage.content_blocks)
提供统一的跨厂商消息结构，支持 OpenAI、Anthropic、Google、AWS、Ollama 等模型。

```python
from langchain_anthropic import ChatAnthropic

model = ChatAnthropic(model="claude-sonnet-4-5")
response = model.invoke("What's the capital of France?")

for block in response.content_blocks:
    if block["type"] == "reasoning":
        print(f"Model reasoning: {block['reasoning']}")
    elif block["type"] == "text":
        print(f"Response: {block['text']}")
    elif block["type"] == "tool_call":
        print(f"Tool call: {block['name']}({block['args']})")
```

优势：

* 🚀 **跨厂商统一接口**（如推理过程、工具调用、引用等）
* 🧩 **类型安全**（完整的类型提示）
* 🧱 **向后兼容**（支持懒加载，不破坏旧逻辑）

---

## 🧭 精简后的命名空间

LangChain v1 对命名空间进行了重构，保留核心构建模块：

| 模块                      | 功能                                        |
| ----------------------- | ----------------------------------------- |
| `langchain.agents`      | 智能体构建（`create_agent`, `AgentState`）       |
| `langchain.messages`    | 消息结构与内容块（`ContentBlock`, `trim_messages`） |
| `langchain.tools`       | 工具系统（`@tool`, `BaseTool`）                 |
| `langchain.chat_models` | 模型初始化（`init_chat_model`）                  |
| `langchain.embeddings`  | 向量嵌入（`init_embeddings`）                   |

示例：

```python
from langchain.agents import create_agent
from langchain.messages import AIMessage, HumanMessage
from langchain.tools import tool
from langchain.chat_models import init_chat_model
from langchain.embeddings import init_embeddings
```

---

## 🧩 `langchain-classic`（旧版兼容包）

为保持核心库精简，旧版功能已迁移至
[`langchain-classic`](https://pypi.org/project/langchain-classic)。

包含：

* 旧版链（Chains）
* 检索器（Retrievers）
* 索引 API
* Hub 模块
* `langchain-community` 导出内容

安装方式：

```bash
pip install langchain-classic
```

并更新导入路径：

```python
from langchain import ...  # ❌
from langchain_classic import ...  # ✅
```

---

## 🧭 更多资源

| 主题                                                                                        | 链接 |
| ----------------------------------------------------------------------------------------- | -- |
| 🚀 [LangChain 1.0 公告](https://blog.langchain.com/langchain-langchain-1-0-alpha-releases/) |    |
| 🧩 [Middleware 深度解析](https://blog.langchain.com/agent-middleware/)                        |    |
| 🤖 [智能体文档](/oss/python/langchain/agents)                                                  |    |
| 💬 [消息与内容块 API](/oss/python/langchain/messages#message-content)                           |    |
| 🔄 [迁移指南](/oss/python/migrate/langchain-v1)                                               |    |
| 🐙 [GitHub 仓库](https://github.com/langchain-ai/langchain)                                 |    |

---

✏️ [编辑本页源码](https://github.com/langchain-ai/docs/edit/main/src/oss/python/releases/langchain-v1.mdx)

💻 [通过 MCP 接入 Claude、VSCode 等工具，实现实时问答](/use-these-docs)
