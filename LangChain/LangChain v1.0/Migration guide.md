# LangChain v1 迁移指南（中文详解）

本文是 **LangChain v1 与旧版本之间的完整迁移说明**。
LangChain v1 聚焦于“**Agent-first**” 核心能力，去除了大量历史包袱，使开发体验更轻量、更一致、更可控。

---

## 🧭 一、命名空间简化（Simplified Package）

v1 对包结构进行了大幅精简，只保留核心 Agent 构建模块。
许多旧的模块（如链、检索器、索引）被迁移到独立的 `langchain-classic` 包。

### ✅ 新版模块结构

| 模块                      | 可用功能                                  | 说明                    |
| ----------------------- | ------------------------------------- | --------------------- |
| `langchain.agents`      | `create_agent`, `AgentState`          | 核心智能体创建               |
| `langchain.messages`    | 消息类型、`content_blocks`、`trim_messages` | 重导出自 `langchain-core` |
| `langchain.tools`       | `@tool`, `BaseTool`                   | 工具定义与注入               |
| `langchain.chat_models` | `init_chat_model`, `BaseChatModel`    | 统一模型初始化接口             |
| `langchain.embeddings`  | `init_embeddings`, `Embeddings`       | 向量嵌入模型                |

### 🧱 旧模块迁移到 `langchain-classic`

旧功能包括：

* 传统链（`LLMChain`, `ConversationChain`）
* 检索器（`MultiQueryRetriever` 等）
* 索引 API
* Hub 模块（Prompt 管理）
* Embeddings 缓存与社区模型
* 旧版 `langchain-community` 导出内容

安装方式：

```bash
pip install langchain-classic
```

迁移示例：

```python
# v0 旧写法
from langchain.chains import LLMChain
from langchain import hub

# v1 新写法
from langchain_classic.chains import LLMChain
from langchain_classic import hub
```

---

## 🤖 二、从 `create_react_agent` 迁移到 `create_agent`

LangChain v1 统一使用：

```python
from langchain.agents import create_agent
```

取代旧的：

```python
from langgraph.prebuilt import create_react_agent
```

---

### 🧩 对比表

| 变更项           | v1 新版                                             | v0 旧版                                   |
| ------------- | ------------------------------------------------- | --------------------------------------- |
| **导入路径**      | `langchain.agents.create_agent`                   | `langgraph.prebuilt.create_react_agent` |
| **prompt 参数** | 改名为 `system_prompt`                               | 使用 `prompt`                             |
| **钩子机制**      | 统一改为 Middleware（如 `before_model` / `after_model`） | 使用 pre_model / post_model hooks         |
| **状态类型**      | 仅支持 `TypedDict`（`AgentState`）                     | 支持 `Pydantic` / `dataclass`             |
| **模型切换**      | 通过 Middleware 动态控制                                | 函数返回模型                                  |
| **工具机制**      | 仅支持函数或 BaseTool，不再支持 ToolNode                     | 可用 ToolNode                             |
| **结构化输出**     | 使用 `ToolStrategy` / `ProviderStrategy`            | 使用响应格式或自定义 prompt                       |
| **上下文传入**     | `context=` 参数传递                                   | `config["configurable"]`                |
| **命名空间**      | 极简核心结构                                            | 包含大量链与组件                                |

---

### 🧠 新接口示例

```python
from langchain.agents import create_agent

agent = create_agent(
    model="anthropic:claude-sonnet-4-5",
    tools=[search_web, analyze_data],
    system_prompt="You are a helpful assistant."
)

result = agent.invoke({
    "messages": [{"role": "user", "content": "Research AI safety"}]
})
```

---

## ⚙️ 三、Middleware（中间件）取代旧 Hook

中间件统一了 `pre_model` / `post_model` / `tool_error` 等旧钩子，
并提供 **可组合、可复用、可扩展** 的执行管线。

### 示例：自动摘要中间件

```python
from langchain.agents import create_agent
from langchain.agents.middleware import SummarizationMiddleware

agent = create_agent(
    model="anthropic:claude-sonnet-4-5",
    tools=tools,
    middleware=[
        SummarizationMiddleware(
            model="anthropic:claude-sonnet-4-5",
            max_tokens_before_summary=1000
        )
    ]
)
```

---

### 示例：人工审批中间件

```python
from langchain.agents.middleware import HumanInTheLoopMiddleware

agent = create_agent(
    model="anthropic:claude-sonnet-4-5",
    tools=[send_email],
    middleware=[
        HumanInTheLoopMiddleware(
            interrupt_on={"send_email": True}
        )
    ]
)
```

---

## 💾 四、自定义状态（Custom State）

v1 仅支持 `TypedDict` 作为状态定义。

### ✅ 推荐写法

```python
from langchain.agents import create_agent, AgentState
from langchain.tools import tool, ToolRuntime

class CustomState(AgentState):
    user_name: str

@tool
def greet(runtime: ToolRuntime[CustomState]) -> str:
    name = runtime.state.get("user_name", "Unknown")
    return f"Hello {name}!"

agent = create_agent(
    model="anthropic:claude-sonnet-4-5",
    tools=[greet],
    state_schema=CustomState
)
```

> ❌ 不再支持 `pydantic.BaseModel` 或 `dataclass`。

---

## 🔄 五、动态模型与工具控制

旧版本通过函数返回不同模型；
v1 改为通过 `Middleware.wrap_model_call` 实现。

```python
from langchain.agents.middleware import AgentMiddleware, ModelRequest, ModelRequestHandler
from langchain_openai import ChatOpenAI

basic = ChatOpenAI(model="gpt-5-nano")
advanced = ChatOpenAI(model="gpt-5")

class DynamicModelMiddleware(AgentMiddleware):
    def wrap_model_call(self, req: ModelRequest, handler: ModelRequestHandler):
        model = advanced if len(req.state.messages) > 10 else basic
        return handler(req.replace(model=model))

agent = create_agent(
    model=basic,
    tools=tools,
    middleware=[DynamicModelMiddleware()]
)
```

---

## 📦 六、结构化输出（Structured Output）

### 新策略

| 策略                 | 说明             |
| ------------------ | -------------- |
| `ToolStrategy`     | 利用伪工具调用生成结构化输出 |
| `ProviderStrategy` | 利用模型原生结构化输出能力  |

示例：

```python
from langchain.agents import create_agent
from langchain.agents.structured_output import ToolStrategy
from pydantic import BaseModel

class Sentiment(BaseModel):
    summary: str
    polarity: str

agent = create_agent(
    "openai:gpt-4o-mini",
    tools=[],
    response_format=ToolStrategy(Sentiment)
)
```

> 🚫 `response_format=("prompt", schema)` 的方式已被移除。

---

## 💬 七、标准化内容块（Standard Content Blocks）

v1 引入 `content_blocks` 属性，为多模态与推理结果提供统一格式。

```python
from langchain.chat_models import init_chat_model

model = init_chat_model("openai:gpt-5-nano")
response = model.invoke("Explain AI")

for block in response.content_blocks:
    if block["type"] == "reasoning":
        print(block["reasoning"])
    elif block["type"] == "text":
        print(block["text"])
```

支持的块类型：

* `text`
* `reasoning`
* `image`
* `tool_call`
* `citation`

启用兼容序列化：

```bash
export LC_OUTPUT_VERSION=v1
```

或

```python
model = init_chat_model("openai:gpt-5-nano", output_version="v1")
```

---

## ⚠️ 八、破坏性变更（Breaking Changes）

| 变更项                    | 新规则                                 |
| ---------------------- | ----------------------------------- |
| 🐍 Python 版本           | 仅支持 ≥ 3.10                          |
| 💬 模型输出类型              | 统一为 `AIMessage`（不再使用 `BaseMessage`） |
| 🧠 `.text()`           | 改为属性访问 `.text`，旧调用方式将发出警告           |
| 📦 `langchain-classic` | 所有旧链、检索、索引功能迁移                      |
| 🧩 `ToolNode`          | 不再支持，使用函数或 BaseTool                 |
| 🧮 `max_tokens`        | `langchain-anthropic` 默认值根据模型自适应    |

---

## ✅ 九、迁移总结表

| 功能领域      | v0 旧版                              | v1 新版                               |
| --------- | ---------------------------------- | ----------------------------------- |
| 智能体创建     | `create_react_agent`               | `create_agent`                      |
| Prompt 参数 | `prompt=`                          | `system_prompt=`                    |
| 动态 Prompt | 函数回调                               | `@dynamic_prompt` 中间件               |
| Hook 机制   | pre/post hooks                     | Middleware 链式调用                     |
| 状态定义      | `Pydantic` / `dataclass`           | `TypedDict`                         |
| 模型动态切换    | 函数返回                               | Middleware                          |
| 工具封装      | ToolNode                           | 函数 / BaseTool                       |
| 结构化输出     | `response_format=(prompt, schema)` | `ToolStrategy` / `ProviderStrategy` |
| 多模态消息     | Provider 自定义                       | 标准 `content_blocks`                 |
| 上下文注入     | `config["configurable"]`           | `context=`                          |
| 命名空间      | 混杂                                 | 精简为 5 个核心模块                         |

---

## 🔗 推荐阅读

* 🚀 [LangChain v1 新特性总览](https://blog.langchain.com/langchain-langchain-1-0-alpha-releases/)
* 🧩 [Middleware 设计指南](https://blog.langchain.com/agent-middleware/)
* 📚 [Agents 文档](https://python.langchain.com/docs/agents)
* 🧱 [Content Blocks 规范](https://python.langchain.com/docs/messages#content-blocks)
* 🔄 [迁移指南原文](https://python.langchain.com/docs/migrate/langchain-v1)
* 🐙 [LangChain GitHub 仓库](https://github.com/langchain-ai/langchain)

---

🪄 **一句话总结：**

> LangChain v1 = LangGraph + Middleware + 标准内容块 + 极简命名空间
> —— 一切为了更纯粹、更可控的智能体构建体验。
