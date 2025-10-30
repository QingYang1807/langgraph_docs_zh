# 短期记忆（Short-term memory）

## 概述

记忆系统用于记录先前交互的信息。
对于 AI 智能体而言，记忆极其重要——它能让智能体记住过去的对话，从反馈中学习，并根据用户偏好进行调整。随着智能体面对越来越复杂的任务和多轮交互，这项能力对效率与用户体验都至关重要。

**短期记忆（Short-term memory）** 让你的应用可以在单个会话或对话线程中保留上下文。

> 💡 **说明：**
> “线程（thread）”用于组织一次会话中的多轮交互，就像电子邮件会把多条消息归入同一个会话一样。

最常见的短期记忆形式是**对话历史（conversation history）**。
然而，较长的对话会给当今的 LLM 带来挑战：完整的历史可能无法装入模型的上下文窗口，从而造成信息丢失或错误。

即便模型支持较长上下文，处理过多历史内容依然会导致性能下降——模型容易被过时或无关内容“干扰”，响应变慢、成本增加。

在聊天应用中，模型通过 [messages](/oss/python/langchain/messages) 接收上下文，其中包含指令（系统消息）和输入（用户消息）。随着交互增加，消息列表会越来越长。
因此，许多应用需要使用“遗忘”或“裁剪”技术来移除陈旧内容。

---

## 用法（Usage）

要为智能体添加线程级的短期记忆，需要在创建时指定一个 **`checkpointer`**。

> ℹ️ **说明：**
> LangChain 智能体将短期记忆作为其状态（state）的一部分进行管理。
> 智能体会将状态存储在图的 state 中，以便在同一会话中共享上下文，同时不同会话保持隔离。
> 状态通过 `checkpointer` 保存到数据库或内存，可随时恢复。
> 智能体在执行或调用工具时更新状态，每一步执行开始时读取状态。

```python
from langchain.agents import create_agent
from langgraph.checkpoint.memory import InMemorySaver  # 高亮

agent = create_agent(
    "openai:gpt-5",
    [get_user_info],
    checkpointer=InMemorySaver(),  # 高亮
)

agent.invoke(
    {"messages": [{"role": "user", "content": "Hi! My name is Bob."}]},
    {"configurable": {"thread_id": "1"}},  # 高亮
)
```

### 生产环境使用

在生产环境中应使用数据库持久化的 checkpointer：

```shell
pip install langgraph-checkpoint-postgres
```

```python
from langchain.agents import create_agent
from langgraph.checkpoint.postgres import PostgresSaver  # 高亮

DB_URI = "postgresql://postgres:postgres@localhost:5442/postgres?sslmode=disable"
with PostgresSaver.from_conn_string(DB_URI) as checkpointer:
    checkpointer.setup()  # 自动创建表
    agent = create_agent(
        "openai:gpt-5",
        [get_user_info],
        checkpointer=checkpointer,
    )
```

---

## 自定义智能体记忆

默认情况下，智能体使用 [`AgentState`](https://reference.langchain.com/python/langchain/agents/#langchain.agents.AgentState) 管理短期记忆，通过 `messages` 键存储对话历史。

可以通过继承 `AgentState` 添加自定义字段，并在创建智能体时通过 `state_schema` 参数传入。

```python
from langchain.agents import create_agent, AgentState
from langgraph.checkpoint.memory import InMemorySaver

class CustomAgentState(AgentState):
    user_id: str
    preferences: dict

agent = create_agent(
    "openai:gpt-5",
    [get_user_info],
    state_schema=CustomAgentState,
    checkpointer=InMemorySaver(),
)

result = agent.invoke(
    {
        "messages": [{"role": "user", "content": "Hello"}],
        "user_id": "user_123",
        "preferences": {"theme": "dark"}
    },
    {"configurable": {"thread_id": "1"}}
)
```

---

## 常见模式

当开启短期记忆后，长对话可能超出模型的上下文窗口。常见的处理方式包括：

| 操作方式                            | 描述               |
| ------------------------------- | ---------------- |
| ✂️ **裁剪消息（Trim messages）**      | 移除最早或最后 N 条消息    |
| 🗑️ **删除消息（Delete messages）**   | 永久删除部分消息         |
| 🧩 **总结消息（Summarize messages）** | 将旧消息总结为简短摘要替代原文  |
| ⚙️ **自定义策略（Custom strategies）** | 例如基于过滤、重要度、情境等规则 |

这些方法可帮助智能体在不超出上下文长度的前提下保持对话连贯。

---

### 裁剪消息（Trim messages）

多数 LLM 有最大上下文窗口（以 token 计）。
可通过计算消息历史的 token 数，在接近上限时裁剪。LangChain 提供了裁剪工具，可指定保留 token 数量及策略（如保留最新消息）。

以下示例展示如何通过 `@before_model` 中间件裁剪消息：

```python
@before_model
def trim_messages(state: AgentState, runtime: Runtime):
    """仅保留最近几条消息"""
    messages = state["messages"]
    if len(messages) <= 3:
        return None
    first_msg = messages[0]
    recent_messages = messages[-3:] if len(messages) % 2 == 0 else messages[-4:]
    return {
        "messages": [RemoveMessage(id=REMOVE_ALL_MESSAGES), *([first_msg] + recent_messages)]
    }
```

---

### 删除消息（Delete messages）

可通过 `RemoveMessage` 删除特定或全部消息。
默认的 `AgentState` 已包含支持 `add_messages` 的 reducer。

删除全部消息示例：

```python
from langgraph.graph.message import REMOVE_ALL_MESSAGES

def delete_messages(state):
    return {"messages": [RemoveMessage(id=REMOVE_ALL_MESSAGES)]}
```

> ⚠️ 注意：删除后需确保消息历史仍符合模型要求。
>
> * 某些模型要求首条为用户消息；
> * 有工具调用的消息需紧随对应结果消息。

---

### 总结消息（Summarize messages）

直接删除或裁剪消息可能丢失信息，因此可使用模型自动总结历史内容并替换。

LangChain 提供了 `SummarizationMiddleware` 实现：

```python
from langchain.agents.middleware import SummarizationMiddleware

agent = create_agent(
    model="openai:gpt-4o",
    middleware=[
        SummarizationMiddleware(
            model="openai:gpt-4o-mini",
            max_tokens_before_summary=4000,
            messages_to_keep=20
        )
    ],
    checkpointer=InMemorySaver(),
)
```

---

## 访问与修改记忆

可通过以下方式访问或更新智能体的短期记忆：

### 1. 在工具中读取记忆

使用 `ToolRuntime` 参数读取状态：

```python
@tool
def get_user_info(runtime: ToolRuntime) -> str:
    user_id = runtime.state["user_id"]
    return "User is John Smith" if user_id == "user_123" else "Unknown user"
```

### 2. 在工具中写入记忆

工具可以返回状态更新，使信息在会话中持久化：

```python
@tool
def update_user_info(runtime: ToolRuntime[CustomContext, CustomState]) -> Command:
    user_id = runtime.context.user_id
    name = "John Smith" if user_id == "user_123" else "Unknown user"
    return Command(update={
        "user_name": name,
        "messages": [ToolMessage("User info updated", tool_call_id=runtime.tool_call_id)]
    })
```

---

### 3. 在 Prompt 中动态访问记忆

可在中间件中根据用户状态动态生成系统提示：

```python
@dynamic_prompt
def dynamic_system_prompt(request: ModelRequest) -> str:
    user_name = request.runtime.context["user_name"]
    return f"You are a helpful assistant. Address the user as {user_name}."
```

---

### 4. 在模型调用前处理（before_model）

可在模型调用前裁剪或调整消息，如：

```python
@before_model
def trim_messages(state: AgentState, runtime: Runtime):
    ...
```

### 5. 在模型调用后处理（after_model）

可在模型输出后执行过滤、校验等逻辑：

```python
@after_model
def validate_response(state: AgentState, runtime: Runtime):
    STOP_WORDS = ["password", "secret"]
    last_message = state["messages"][-1]
    if any(word in last_message.content for word in STOP_WORDS):
        return {"messages": [RemoveMessage(id=last_message.id)]}
```

---

> ✏️ [在 GitHub 上编辑本文档](https://github.com/langchain-ai/docs/edit/main/src/oss/langchain/short-term-memory.mdx)
> 💡 [通过 MCP 将本文档连接至 Claude、VSCode 等工具，实现实时问答](/use-these-docs)
