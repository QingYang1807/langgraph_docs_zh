# 中间件（Middleware）

> **在每一步精确控制与定制智能体执行流程**

中间件为你提供了更强大的方式来控制智能体的内部运行逻辑。

---

## 🧠 智能体核心循环

智能体的核心执行流程是：
调用模型 → 模型选择并执行工具 → 当没有更多工具调用时结束。

<div align="center">
  <img src="https://mintcdn.com/langchain-5e9cc07a/Tazq8zGc0yYUYrDl/oss/images/core_agent_loop.png" alt="核心智能体循环示意图" width="300"/>
</div>

中间件允许在每个关键步骤的**前后**插入钩子（hooks）：

<div align="center">
  <img src="https://mintcdn.com/langchain-5e9cc07a/RAP6mjwE5G00xYsA/oss/images/middleware_final.png" alt="中间件流程示意图" width="500"/>
</div>

---

## 🌟 中间件能做什么？

| 类型              | 说明                      |
| --------------- | ----------------------- |
| **Monitor（监控）** | 通过日志、分析和调试跟踪智能体行为       |
| **Modify（修改）**  | 转换提示词、工具选择或输出格式         |
| **Control（控制）** | 添加重试、回退或提前终止逻辑          |
| **Enforce（约束）** | 应用限流、防护策略、PII（个人隐私信息）检测 |

---

## 🚀 如何添加中间件

通过 `create_agent` 传入中间件列表即可：

```python
from langchain.agents import create_agent
from langchain.agents.middleware import SummarizationMiddleware, HumanInTheLoopMiddleware

agent = create_agent(
    model="openai:gpt-4o",
    tools=[...],
    middleware=[SummarizationMiddleware(), HumanInTheLoopMiddleware()],
)
```

---

## 🧩 内置中间件

LangChain 提供了多种常用场景下的内置中间件：

---

### ✍️ Summarization（摘要中间件）

当对话接近上下文长度上限时，自动对历史对话进行摘要。

**适用场景：**

* 长会话超过上下文窗口
* 多轮对话
* 需要保留完整上下文的应用

```python
from langchain.agents.middleware import SummarizationMiddleware

agent = create_agent(
    model="openai:gpt-4o",
    tools=[weather_tool, calculator_tool],
    middleware=[
        SummarizationMiddleware(
            model="openai:gpt-4o-mini",
            max_tokens_before_summary=4000,
            messages_to_keep=20,
            summary_prompt="自定义摘要提示词..."
        )
    ]
)
```

主要配置项：

* `model`: 用于生成摘要的模型
* `max_tokens_before_summary`: 触发摘要的token阈值
* `messages_to_keep`: 摘要后保留的最近消息数

---

### 🧍 Human-in-the-loop（人类参与中间件）

在工具执行前暂停智能体运行，等待人工**批准 / 编辑 / 拒绝**。

**适用场景：**

* 高风险操作（数据库写入、金融交易）
* 合规场景（需人工审批）
* 长会话中需人工指导

```python
from langchain.agents.middleware import HumanInTheLoopMiddleware
from langgraph.checkpoint.memory import InMemorySaver

agent = create_agent(
    model="openai:gpt-4o",
    tools=[read_email_tool, send_email_tool],
    checkpointer=InMemorySaver(),
    middleware=[
        HumanInTheLoopMiddleware(
            interrupt_on={
                "send_email_tool": {"allowed_decisions": ["approve", "edit", "reject"]},
                "read_email_tool": False
            }
        )
    ]
)
```

> ⚠️ 需搭配 `checkpointer` 使用，用于在中断间保存状态。

---

### 🧠 Anthropic Prompt Caching（提示缓存）

缓存重复的提示前缀，减少调用成本（适用于 Anthropic 模型）。

---

### 🔢 Model Call Limit（模型调用限制）

限制模型调用次数，防止死循环或超额成本。

---

### 🧰 Tool Call Limit（工具调用限制）

限制工具调用次数，可针对全局或特定工具。

---

### 🔄 Model Fallback（模型回退）

主模型出错时自动回退到备用模型，增强鲁棒性。

---

### 🕵️ PII Detection（个人隐私信息检测）

检测并处理输入输出中的敏感信息。

示例：

```python
from langchain.agents.middleware import PIIMiddleware

agent = create_agent(
    model="openai:gpt-4o",
    middleware=[
        PIIMiddleware("email", strategy="redact", apply_to_input=True),
        PIIMiddleware("credit_card", strategy="mask", apply_to_input=True),
        PIIMiddleware("api_key", detector=r"sk-[a-zA-Z0-9]{32}", strategy="block"),
    ]
)
```

---

### 🗂 TodoListMiddleware（任务计划中间件）

为复杂任务添加自动的待办事项（ToDo）管理功能。

---

### ⚙️ LLM Tool Selector（智能工具选择）

使用LLM自动筛选最相关工具，降低token消耗并提升效率。

---

### 🔁 Tool Retry（工具重试机制）

对失败的工具调用进行自动重试，支持指数退避与抖动。

---

### 🧪 LLM Tool Emulator（工具模拟器）

用LLM模拟工具执行结果，适合测试或原型验证。

---

### 🧹 Context Editing（上下文编辑）

通过清理、摘要或裁剪上下文，控制对话长度与质量。

---

## 🛠 自定义中间件

你可以通过以下两种方式自定义中间件：

1. **装饰器式（Decorator-based）**：轻量、快速，用于单钩子逻辑
2. **类式（Class-based）**：结构化、可复用，适合复杂场景

---

### 装饰器式中间件示例

```python
from langchain.agents.middleware import before_model, after_model, wrap_model_call

@before_model
def log_before(state, runtime):
    print(f"准备调用模型，共 {len(state['messages'])} 条消息")

@after_model
def validate(state, runtime):
    if "BLOCKED" in state["messages"][-1].content:
        return {"messages": ["我无法回应该请求"], "jump_to": "end"}
```

---

### 类式中间件示例

```python
from langchain.agents.middleware import AgentMiddleware

class RetryMiddleware(AgentMiddleware):
    def wrap_model_call(self, request, handler):
        for attempt in range(3):
            try:
                return handler(request)
            except Exception as e:
                if attempt == 2: raise
                print(f"重试 {attempt+1}/3 次，错误：{e}")
```

---

## ⚖️ 执行顺序

当有多个中间件时：

1. `before_*` 按顺序执行
2. `wrap_*` 嵌套执行（外层包裹内层）
3. `after_*` 逆序执行

---

## 🧭 最佳实践

1. **单一职责**：每个中间件只做一件事
2. **优雅失败**：中间件异常不应导致智能体崩溃
3. **正确使用 Hook 类型**：

   * Node-style → 顺序逻辑（如日志、验证）
   * Wrap-style → 控制流（如重试、缓存）
4. **明确定义自定义状态字段**
5. **独立单测后再集成**
6. **关键中间件放在最前**
7. **优先使用官方中间件**

---

## 🔍 示例：动态选择工具

```python
from langchain.agents.middleware import AgentMiddleware

class ToolSelectorMiddleware(AgentMiddleware):
    def wrap_model_call(self, request, handler):
        request.tools = select_relevant_tools(request.state, request.runtime)
        return handler(request)
```

---

## 📚 延伸阅读

* [Middleware API 文档](https://reference.langchain.com/python/langchain/middleware/)
* [Human-in-the-loop 指南](/oss/python/langchain/human-in-the-loop)
* [Agent 测试方法](/oss/python/langchain/test)

---

💡 **提示**：
你可以通过 MCP 将这些文档连接至 Claude、VSCode 等工具中，实现实时文档问答。
