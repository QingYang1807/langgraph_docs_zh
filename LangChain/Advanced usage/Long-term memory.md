# 长期记忆

## 概述

LangChain 代理使用 [LangGraph 持久化](/oss/python/langgraph/persistence#memory-store) 来启用长期记忆。这是一个更高级的主题，需要具备 LangGraph 知识才能使用。

## 内存存储

LangGraph 将长期记忆作为 JSON 文档存储在 [存储](/oss/python/langgraph/persistence#memory-store) 中。

每个记忆都在自定义的 `命名空间`（类似于文件夹）和唯一的 `键`（类似于文件名）下组织。命名空间通常包含用户或组织 ID 或其他标签，以便更轻松地组织信息。

这种结构支持记忆的分层组织。然后可以通过内容过滤器实现跨命名空间搜索。

```python  theme={null}
from langgraph.store.memory import InMemoryStore


def embed(texts: list[str]) -> list[list[float]]:
    # 替换为实际的嵌入函数或 LangChain 嵌入对象
    return [[1.0, 2.0] * len(texts)]


# InMemoryStore 将数据保存到内存字典中。在生产环境中使用基于数据库的存储。
store = InMemoryStore(index={"embed": embed, "dims": 2}) # [!code highlight]
user_id = "my-user"
application_context = "chitchat"
namespace = (user_id, application_context) # [!code highlight]
store.put( # [!code highlight]
    namespace,
    "a-memory",
    {
        "rules": [
            "用户喜欢简短直接的语言",
            "用户只说英语和python",
        ],
        "my-key": "my-value",
    },
)
# 通过 ID 获取"记忆"
item = store.get(namespace, "a-memory") # [!code highlight]
# 在此命名空间内搜索"记忆"，基于内容等效性过滤，按向量相似度排序
items = store.search( # [!code highlight]
    namespace, filter={"my-key": "my-value"}, query="语言偏好"
)
```

有关内存存储的更多信息，请参阅 [持久化](/oss/python/langgraph/persistence#memory-store) 指南。

## 在工具中读取长期记忆

```python 代理可用于查找用户信息的工具 theme={null}
from dataclasses import dataclass

from langchain_core.runnables import RunnableConfig
from langchain.agents import create_agent
from langchain.tools import tool, ToolRuntime
from langgraph.store.memory import InMemoryStore


@dataclass
class Context:
    user_id: str

# InMemoryStore 将数据保存到内存字典中。在生产环境中使用基于数据库的存储。
store = InMemoryStore() # [!code highlight]

# 使用 put 方法将示例数据写入存储
store.put( # [!code highlight]
    ("users",),  # 用于分组相关数据的命名空间（用户数据的用户命名空间）
    "user_123",  # 命名空间内的键（用户 ID 作为键）
    {
        "name": "John Smith",
        "language": "English",
    }  # 为给定用户存储的数据
)

@tool
def get_user_info(runtime: ToolRuntime[Context]) -> str:
    """查找用户信息。"""
    # 访问存储 - 与提供给 `create_agent` 的存储相同
    store = runtime.store # [!code highlight]
    user_id = runtime.context.user_id
    # 从存储中检索数据 - 返回包含值和元数据的 StoreValue 对象
    user_info = store.get(("users",), user_id) # [!code highlight]
    return str(user_info.value) if user_info else "未知用户"

agent = create_agent(
    model="anthropic:claude-sonnet-4-5",
    tools=[get_user_info],
    # 将存储传递给代理 - 使代理在运行工具时能够访问存储
    store=store, # [!code highlight]
    context_schema=Context
)

# 运行代理
agent.invoke(
    {"messages": [{"role": "user", "content": "查找用户信息"}]},
    context=Context(user_id="user_123") # [!code highlight]
)
```

<a id="write-long-term" />

## 从工具写入长期记忆

```python 更新用户信息的工具示例 theme={null}
from dataclasses import dataclass
from typing_extensions import TypedDict

from langchain.agents import create_agent
from langchain.tools import tool, ToolRuntime
from langgraph.store.memory import InMemoryStore


# InMemoryStore 将数据保存到内存字典中。在生产环境中使用基于数据库的存储。
store = InMemoryStore() # [!code highlight]

@dataclass
class Context:
    user_id: str

# TypedDict 定义用户信息的结构，供 LLM 使用
class UserInfo(TypedDict):
    name: str

# 允许代理更新用户信息的工具（对聊天应用程序很有用）
@tool
def save_user_info(user_info: UserInfo, runtime: ToolRuntime[Context]) -> str:
    """保存用户信息。"""
    # 访问存储 - 与提供给 `create_agent` 的存储相同
    store = runtime.store # [!code highlight]
    user_id = runtime.context.user_id # [!code highlight]
    # 将数据存储在存储中（命名空间，键，数据）
    store.put(("users",), user_id, user_info) # [!code highlight]
    return "成功保存用户信息。"

agent = create_agent(
    model="anthropic:claude-sonnet-4-5",
    tools=[save_user_info],
    store=store, # [!code highlight]
    context_schema=Context
)

# 运行代理
agent.invoke(
    {"messages": [{"role": "user", "content": "我的名字是 John Smith"}]},
    # 在上下文中传递 user_id 以标识正在更新谁的信息
    context=Context(user_id="user_123") # [!code highlight]
)

# 您可以直接访问存储来获取值
store.get(("users",), "user_123").value
```

***

<Callout icon="pen-to-square" iconType="regular">
  [在 GitHub 上编辑此页面的源代码。](https://github.com/langchain-ai/docs/edit/main/src/oss/langchain/long-term-memory.mdx)
</Callout>

<Tip icon="terminal" iconType="regular">
  [通过 MCP 以编程方式连接这些文档](/use-these-docs)，与 Claude、VSCode 等连接，以获取实时答案。
</Tip>