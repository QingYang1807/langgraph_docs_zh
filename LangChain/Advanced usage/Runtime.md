# 运行时（Runtime）

## 概述

LangChain 的 [`create_agent`](https://reference.langchain.com/python/langchain/agents/#langchain.agents.create_agent) 在底层运行于 LangGraph 的运行时环境之上。

LangGraph 提供了一个 [`Runtime`](https://reference.langchain.com/python/langgraph/runtime/#langgraph.runtime.Runtime) 对象，包含以下信息：

1. **Context（上下文）**：静态信息，如用户 ID、数据库连接或其他用于代理调用的依赖项
2. **Store（存储）**：一个 [BaseStore](https://reference.langchain.com/python/langgraph/store/#langgraph.store.base.BaseStore) 实例，用于 [长期记忆](/oss/python/langchain/long-term-memory)
3. **Stream writer（流写入器）**：一个用于通过 `"custom"` 流模式进行信息流式传输的对象

你可以在 [工具（tools）](#inside-tools) 和 [中间件（middleware）](#inside-middleware) 内访问运行时信息。

---

## 访问方式

当使用 [`create_agent`](https://reference.langchain.com/python/langchain/agents/#langchain.agents.create_agent) 创建代理时，你可以通过 `context_schema` 参数定义 `Runtime` 中 `context` 的结构。

在调用代理时，使用 `context` 参数传入本次运行的相关配置信息：

```python
from dataclasses import dataclass
from langchain.agents import create_agent

@dataclass
class Context:
    user_name: str

agent = create_agent(
    model="openai:gpt-5-nano",
    tools=[...],
    context_schema=Context  # [!code highlight]
)

agent.invoke(
    {"messages": [{"role": "user", "content": "我的名字是什么？"}]},
    context=Context(user_name="John Smith")  # [!code highlight]
)
```

---

### 在工具内部（Inside tools）

你可以在工具内部访问运行时信息，用于：

* 访问上下文
* 读取或写入长期记忆
* 向 [自定义流](/oss/python/langchain/streaming#custom-updates) 写入信息（例如工具进度或实时更新）

使用 `ToolRuntime` 参数即可在工具中访问 [`Runtime`](https://reference.langchain.com/python/langgraph/runtime/#langgraph.runtime.Runtime) 对象。

```python
from dataclasses import dataclass
from langchain.tools import tool, ToolRuntime  # [!code highlight]

@dataclass
class Context:
    user_id: str

@tool
def fetch_user_email_preferences(runtime: ToolRuntime[Context]) -> str:  # [!code highlight]
    """从存储中获取用户的邮件偏好设置。"""
    user_id = runtime.context.user_id  # [!code highlight]

    preferences: str = "该用户偏好简短而礼貌的邮件。"
    if runtime.store:  # [!code highlight]
        if memory := runtime.store.get(("users",), user_id):  # [!code highlight]
            preferences = memory.value["preferences"]

    return preferences
```

---

### 在中间件内部（Inside middleware）

你可以在中间件中访问运行时信息，用于动态创建提示词、修改消息，或基于用户上下文控制代理的行为。

使用 `request.runtime` 即可在中间件装饰器中访问 [`Runtime`](https://reference.langchain.com/python/langgraph/runtime/#langgraph.runtime.Runtime) 对象。
该运行时对象存在于传递给中间件函数的 [`ModelRequest`](https://reference.langchain.com/python/langchain/middleware/#langchain.agents.middleware.ModelRequest) 参数中。

```python
from dataclasses import dataclass

from langchain.messages import AnyMessage
from langchain.agents import create_agent, AgentState
from langchain.agents.middleware import dynamic_prompt, ModelRequest, before_model, after_model
from langgraph.runtime import Runtime

@dataclass
class Context:
    user_name: str

# 动态提示词
@dynamic_prompt
def dynamic_system_prompt(request: ModelRequest) -> str:
    user_name = request.runtime.context.user_name  # [!code highlight]
    system_prompt = f"你是一名乐于助人的助手，请称呼用户为 {user_name}。"
    return system_prompt

# 模型执行前钩子
@before_model
def log_before_model(state: AgentState, runtime: Runtime[Context]) -> dict | None:  # [!code highlight]
    print(f"正在处理用户请求：{runtime.context.user_name}")  # [!code highlight]
    return None

# 模型执行后钩子
@after_model
def log_after_model(state: AgentState, runtime: Runtime[Context]) -> dict | None:  # [!code highlight]
    print(f"已完成用户请求：{runtime.context.user_name}")  # [!code highlight]
    return None

agent = create_agent(
    model="openai:gpt-5-nano",
    tools=[...],
    middleware=[dynamic_system_prompt, log_before_model, log_after_model],  # [!code highlight]
    context_schema=Context
)

agent.invoke(
    {"messages": [{"role": "user", "content": "我的名字是什么？"}]},
    context=Context(user_name="John Smith")
)
```

---

<Callout icon="pen-to-square" iconType="regular">
  [在 GitHub 上编辑此页面的源文件。](https://github.com/langchain-ai/docs/edit/main/src/oss/langchain/runtime.mdx)
</Callout>

<Tip icon="terminal" iconType="regular">
  [通过 MCP 连接这些文档](/use-these-docs)，在 Claude、VSCode 等工具中实时获取答案。
</Tip>
