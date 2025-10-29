# LangChain 快速上手（Quickstart 中文版）

本教程将带你在几分钟内，从零搭建一个 **可运行的 AI 智能体（Agent）**。
你将学会如何构建、配置、运行并持续交互一个“具备记忆与工具调用能力”的 LangChain 智能体。

---

## 🚀 1. 构建基础智能体

下面示例展示了最基础的智能体：

* 模型：Claude Sonnet 4.5
* 工具：天气查询函数
* 系统提示词：定义 Agent 行为

```python
from langchain.agents import create_agent

def get_weather(city: str) -> str:
    """获取指定城市天气"""
    return f"It's always sunny in {city}!"

agent = create_agent(
    model="anthropic:claude-sonnet-4-5",
    tools=[get_weather],
    system_prompt="You are a helpful assistant",
)

# 调用智能体
agent.invoke(
    {"messages": [{"role": "user", "content": "what is the weather in sf"}]}
)
```

> 💡 提示：
> 本例需在系统环境中设置 Anthropic API Key
>
> ```bash
> export ANTHROPIC_API_KEY="your_api_key_here"
> ```

---

## 🌦️ 2. 构建真实世界智能体（Weather Agent）

接下来，我们将构建一个更完整的天气预报智能体，它具备：

1. **角色提示（System Prompt）**：定义语气与行为；
2. **外部工具调用（Tools）**：从外部获取数据；
3. **模型配置（Model Config）**：控制输出一致性；
4. **结构化输出（Structured Output）**：让响应更可解析；
5. **记忆能力（Memory）**：让对话具备上下文；
6. **上下文注入（Context）**：支持用户级个性化。

---

### 🧱 Step 1：定义系统提示（System Prompt）

```python
SYSTEM_PROMPT = """You are an expert weather forecaster, who speaks in puns.

You have access to two tools:

- get_weather_for_location: use this to get the weather for a specific location
- get_user_location: use this to get the user's location

If a user asks you for the weather, make sure you know the location.
If you can tell from the question that they mean wherever they are,
use the get_user_location tool to find their location."""
```

---

### 🔧 Step 2：定义工具（Tools）

工具允许模型执行函数调用，与外部系统交互。

```python
from dataclasses import dataclass
from langchain.tools import tool, ToolRuntime

@tool
def get_weather_for_location(city: str) -> str:
    """获取指定城市天气"""
    return f"It's always sunny in {city}!"

@dataclass
class Context:
    """运行时上下文（自定义结构）"""
    user_id: str

@tool
def get_user_location(runtime: ToolRuntime[Context]) -> str:
    """根据用户ID获取位置"""
    user_id = runtime.context.user_id
    return "Florida" if user_id == "1" else "SF"
```

> 💡 工具文档（函数名、参数名、说明）会自动被注入模型提示中。

---

### 🧠 Step 3：配置模型（Model Configuration）

```python
from langchain.chat_models import init_chat_model

model = init_chat_model(
    "anthropic:claude-sonnet-4-5",
    temperature=0.5,
    timeout=10,
    max_tokens=1000
)
```

---

### 📄 Step 4：定义响应结构（Response Format）

```python
from dataclasses import dataclass

@dataclass
class ResponseFormat:
    """智能体响应结构"""
    punny_response: str  # 双关语式回应（必填）
    weather_conditions: str | None = None  # 可选天气详情
```

---

### 💾 Step 5：添加记忆（Memory）

使用 `InMemorySaver` 在短期内保存上下文对话。
生产环境中可改为数据库持久化。

```python
from langgraph.checkpoint.memory import InMemorySaver

checkpointer = InMemorySaver()
```

---

### 🧠 Step 6：创建并运行智能体（Create & Run Agent）

```python
from langchain.agents import create_agent

agent = create_agent(
    model=model,
    system_prompt=SYSTEM_PROMPT,
    tools=[get_user_location, get_weather_for_location],
    context_schema=Context,
    response_format=ResponseFormat,
    checkpointer=checkpointer
)

# 唯一线程ID，用于维持对话
config = {"configurable": {"thread_id": "1"}}

response = agent.invoke(
    {"messages": [{"role": "user", "content": "what is the weather outside?"}]},
    config=config,
    context=Context(user_id="1")
)

print(response['structured_response'])
```

输出示例：

```python
ResponseFormat(
    punny_response="Florida is still having a 'sun-derful' day!...",
    weather_conditions="It's always sunny in Florida!"
)
```

---

### 🔁 Step 7：连续对话（Conversation Continuity）

使用相同 `thread_id` 可保持上下文记忆：

```python
response = agent.invoke(
    {"messages": [{"role": "user", "content": "thank you!"}]},
    config=config,
    context=Context(user_id="1")
)

print(response['structured_response'])
# ResponseFormat(
#     punny_response="You're 'thund-erfully' welcome!...",
#     weather_conditions=None
# )
```

---

## 🧩 完整示例代码

```python
from dataclasses import dataclass
from langchain.agents import create_agent
from langchain.chat_models import init_chat_model
from langchain.tools import tool, ToolRuntime
from langgraph.checkpoint.memory import InMemorySaver

SYSTEM_PROMPT = """You are an expert weather forecaster, who speaks in puns.

You have access to two tools:
- get_weather_for_location
- get_user_location
"""

@dataclass
class Context:
    user_id: str

@tool
def get_weather_for_location(city: str) -> str:
    return f"It's always sunny in {city}!"

@tool
def get_user_location(runtime: ToolRuntime[Context]) -> str:
    return "Florida" if runtime.context.user_id == "1" else "SF"

model = init_chat_model("anthropic:claude-sonnet-4-5", temperature=0)
checkpointer = InMemorySaver()

@dataclass
class ResponseFormat:
    punny_response: str
    weather_conditions: str | None = None

agent = create_agent(
    model=model,
    system_prompt=SYSTEM_PROMPT,
    tools=[get_user_location, get_weather_for_location],
    context_schema=Context,
    response_format=ResponseFormat,
    checkpointer=checkpointer
)

config = {"configurable": {"thread_id": "1"}}
response = agent.invoke(
    {"messages": [{"role": "user", "content": "what is the weather outside?"}]},
    config=config,
    context=Context(user_id="1")
)
print(response['structured_response'])
```

---

## 🎯 你现在已经掌握了一个完整可用的 LangChain 智能体！

它可以：

✅ 理解上下文、维持对话状态
✅ 调用多个外部工具
✅ 返回结构化输出格式
✅ 基于用户上下文动态响应
✅ 支持持久化记忆与多轮交互

---

✏️ [在 GitHub 上编辑本页](https://github.com/langchain-ai/docs/edit/main/src/oss/langchain/quickstart.mdx)
💻 [通过 MCP 接入 Claude、VSCode 等工具，实现实时问答](/use-these-docs)
