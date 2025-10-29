# 工具（Tools）

许多 AI 应用通过自然语言与用户交互，但某些场景下，模型需要直接与外部系统（如 API、数据库或文件系统）进行交互，并使用结构化输入。

**工具（Tools）** 是供 [智能体（Agents）](/oss/python/langchain/agents) 调用以执行动作的组件。它们通过定义良好的输入与输出，让模型能够与外部世界交互，从而扩展模型能力。工具封装了一个可调用函数及其输入模式（schema）。这些工具可以传递给兼容的 [聊天模型](/oss/python/langchain/models)，由模型自行决定是否调用工具以及使用什么参数。在此类场景中，“工具调用”让模型能生成符合指定输入模式的请求。

> 💡 **服务端工具调用**
>
> 某些聊天模型（如 [OpenAI](/oss/python/integrations/chat/openai)、[Anthropic](/oss/python/integrations/chat/anthropic)、[Gemini](/oss/python/integrations/chat/google_generative_ai)）自带服务端执行的 [内置工具](/oss/python/langchain/models#server-side-tool-use)，例如网页搜索与代码解释器。请参阅 [模型提供商概览](/oss/python/integrations/providers/overview) 了解如何在特定聊天模型中使用这些工具。

---

## 创建工具（Create Tools）

### 基础定义

创建工具的最简单方式是使用 [`@tool`](https://reference.langchain.com/python/langchain/tools/#langchain.tools.tool) 装饰器。默认情况下，函数的文档字符串（docstring）会成为工具的描述，帮助模型理解使用场景：

```python
from langchain.tools import tool

@tool
def search_database(query: str, limit: int = 10) -> str:
    """在客户数据库中搜索匹配查询的记录。

    Args:
        query: 搜索关键词
        limit: 返回的最大结果数
    """
    return f"Found {limit} results for '{query}'"
```

类型提示（type hints）是**必须的**，因为它们定义了工具的输入结构。文档字符串应简洁清晰，便于模型理解工具用途。

---

### 自定义工具属性

#### 自定义工具名称

默认名称来源于函数名，可通过参数自定义：

```python
@tool("web_search")  # 自定义名称
def search(query: str) -> str:
    """在网络上搜索信息。"""
    return f"Results for: {query}"

print(search.name)  # 输出 "web_search"
```

#### 自定义描述

可重写自动生成的描述，便于模型理解使用场景：

```python
@tool("calculator", description="执行算术计算，用于解决任何数学问题。")
def calc(expression: str) -> str:
    """计算数学表达式。"""
    return str(eval(expression))
```

---

### 高级输入结构定义

可使用 **Pydantic 模型** 或 **JSON Schema** 定义复杂输入结构。

**Pydantic 示例：**

```python
from pydantic import BaseModel, Field
from typing import Literal
from langchain.tools import tool

class WeatherInput(BaseModel):
    """天气查询输入参数"""
    location: str = Field(description="城市名或经纬度")
    units: Literal["celsius", "fahrenheit"] = Field(
        default="celsius", description="温度单位"
    )
    include_forecast: bool = Field(
        default=False, description="是否包含未来5天天气预报"
    )

@tool(args_schema=WeatherInput)
def get_weather(location: str, units: str = "celsius", include_forecast: bool = False) -> str:
    """获取当前天气及可选预报。"""
    temp = 22 if units == "celsius" else 72
    result = f"{location} 当前温度：{temp}°{units[0].upper()}"
    if include_forecast:
        result += "\n未来5天：晴"
    return result
```

**JSON Schema 示例：**

```python
weather_schema = {
    "type": "object",
    "properties": {
        "location": {"type": "string"},
        "units": {"type": "string"},
        "include_forecast": {"type": "boolean"}
    },
    "required": ["location", "units", "include_forecast"]
}

@tool(args_schema=weather_schema)
def get_weather(location: str, units: str = "celsius", include_forecast: bool = False) -> str:
    """获取当前天气及可选预报。"""
    temp = 22 if units == "celsius" else 72
    result = f"{location} 当前温度：{temp}°{units[0].upper()}"
    if include_forecast:
        result += "\n未来5天：晴"
    return result
```

---

## 访问上下文（Accessing Context）

> 🧠 **重要性说明**
> 工具的真正威力在于：能访问智能体状态、运行时上下文及长期记忆。这让工具能够做出上下文感知的决策、个性化响应并维持跨对话的一致性。

工具可通过 `ToolRuntime` 参数访问运行时信息，包括：

* **State（状态）**：执行过程中的可变数据（如消息、计数器、自定义字段）
* **Context（上下文）**：不可变的环境配置（如用户ID、会话信息、应用配置）
* **Store（存储）**：持久化的长期记忆
* **Stream Writer（流式输出）**：实时向用户输出更新信息
* **Config（配置）**：执行时的运行配置
* **Tool Call ID**：当前工具调用ID

---

### ToolRuntime 使用

`ToolRuntime` 提供对运行信息的统一访问，只需在函数参数中添加 `runtime: ToolRuntime`，系统会自动注入。

```python
from langchain.tools import tool, ToolRuntime

@tool
def summarize_conversation(runtime: ToolRuntime) -> str:
    """总结当前对话内容。"""
    messages = runtime.state["messages"]
    human_msgs = sum(1 for m in messages if m.__class__.__name__ == "HumanMessage")
    ai_msgs = sum(1 for m in messages if m.__class__.__name__ == "AIMessage")
    tool_msgs = sum(1 for m in messages if m.__class__.__name__ == "ToolMessage")
    return f"共有 {human_msgs} 条用户消息，{ai_msgs} 条AI回复，{tool_msgs} 条工具消息。"
```

> ⚠️ 注意：
> `runtime` 参数对模型是**不可见的**，即模型仅看到其他显式参数。

---

### 更新状态（Update State）

使用 [`Command`](https://reference.langchain.com/python/langgraph/types/#langgraph.types.Command) 更新智能体状态或控制执行流程：

```python
from langgraph.types import Command
from langchain.messages import RemoveMessage
from langgraph.graph.message import REMOVE_ALL_MESSAGES
from langchain.tools import tool

@tool
def clear_conversation() -> Command:
    """清空对话历史。"""
    return Command(update={"messages": [RemoveMessage(id=REMOVE_ALL_MESSAGES)]})

@tool
def update_user_name(new_name: str, runtime: ToolRuntime) -> Command:
    """更新用户名。"""
    return Command(update={"user_name": new_name})
```

---

### 访问上下文（Context）

工具可以通过 `runtime.context` 访问用户ID、会话配置等上下文信息。

```python
from dataclasses import dataclass
from langchain_openai import ChatOpenAI
from langchain.agents import create_agent
from langchain.tools import tool, ToolRuntime

USER_DATABASE = {
    "user123": {"name": "Alice", "balance": 5000},
    "user456": {"name": "Bob", "balance": 1200},
}

@dataclass
class UserContext:
    user_id: str

@tool
def get_account_info(runtime: ToolRuntime[UserContext]) -> str:
    """获取当前用户账户信息。"""
    user_id = runtime.context.user_id
    user = USER_DATABASE.get(user_id)
    return f"{user['name']} 的余额为 ${user['balance']}" if user else "用户不存在"
```

---

### 长期记忆（Store）

通过 `runtime.store` 访问持久化数据，实现跨会话记忆：

```python
from langgraph.store.memory import InMemoryStore
from langchain.tools import tool, ToolRuntime

@tool
def get_user_info(user_id: str, runtime: ToolRuntime) -> str:
    """获取用户信息。"""
    store = runtime.store
    user_info = store.get(("users",), user_id)
    return str(user_info.value) if user_info else "未知用户"

@tool
def save_user_info(user_id: str, user_info: dict, runtime: ToolRuntime) -> str:
    """保存用户信息。"""
    store = runtime.store
    store.put(("users",), user_id, user_info)
    return "用户信息已保存。"
```

---

### 流式输出（Stream Writer）

可通过 `runtime.stream_writer` 实时输出执行进度或状态更新：

```python
from langchain.tools import tool, ToolRuntime

@tool
def get_weather(city: str, runtime: ToolRuntime) -> str:
    """查询城市天气。"""
    writer = runtime.stream_writer
    writer(f"正在查询 {city} 的天气数据…")
    writer(f"已获取 {city} 天气信息。")
    return f"{city} 永远阳光明媚！"
```

> ⚠️ 提示：
> 若使用 `runtime.stream_writer`，需在 LangGraph 执行上下文中运行工具。详见 [Streaming](/oss/python/langchain/streaming)。

---

📄 [在 GitHub 上编辑此页面](https://github.com/langchain-ai/docs/edit/main/src/oss/langchain/tools.mdx)
🧰 [将此文档通过 MCP 实时连接到 Claude、VSCode 等环境](/use-these-docs)
