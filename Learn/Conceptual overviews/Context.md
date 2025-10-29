# 上下文概览

**上下文工程**是构建动态系统的实践，这些系统以正确的格式提供正确的信息和工具，以便AI应用程序能够完成任务。上下文可以根据两个关键维度进行特征描述：

1. 按**可变性**：

* **静态上下文**：执行期间不会改变的不可变数据（例如，用户元数据、数据库连接、工具）
* **动态上下文**：随着应用程序运行而演变的可变数据（例如，对话历史、中间结果、工具调用观察）

2. 按**生命周期**：

* **运行时上下文**：仅限于单个运行或调用的数据
* **跨对话上下文**：跨多个对话或会话持久存在的数据

<提示>
  运行时上下文是指本地上下文：代码运行所需的数据和依赖项。它**不**指：

  * 传递给LLM提示的LLM上下文数据。
  * "上下文窗口"，即可以传递给LLM的最大token数量。

  运行时上下文可用于优化LLM上下文。例如，您可以在运行时上下文中使用用户元数据来获取用户偏好，并将其输入到上下文窗口中。
</提示>

LangGraph提供三种管理上下文的方法，这些方法结合了可变性和生命周期维度：

| 上下文类型                                                                                                       | 描述                                                 | 可变性     | 生命周期           | 访问方法                               |
| ---------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------- | ---------- | ------------------ | --------------------------------------- |
| [**静态运行时上下文**](#static-runtime-context)                                                                  | 在启动时传递的用户元数据、工具、数据库连接           | 静态       | 单次运行           | `invoke`/`stream`的`context`参数       |
| [**动态运行时上下文（状态）**](#dynamic-runtime-context-state)                                                   | 在单次运行过程中演变的可变数据                       | 动态       | 单次运行           | LangGraph状态对象                      |
| [**动态跨对话上下文（存储）**](#dynamic-cross-conversation-context-store)                                         | 跨多个对话共享的持久数据                             | 动态       | 跨对话             | LangGraph存储                          |

## 静态运行时上下文

**静态运行时上下文**表示不可变数据，如用户元数据、工具和数据库连接，这些数据在运行开始时通过`invoke`/`stream`的`context`参数传递给应用程序。这些数据在执行期间不会改变。

```python  theme={null}
@dataclass
class ContextSchema:
    user_name: str

graph.invoke(
    {"messages": [{"role": "user", "content": "hi!"}]},
    context={"user_name": "John Smith"}  # [!code highlight]
)
```

<选项卡>
  <选项卡标题="代理提示">
    ```python  theme={null}
    from dataclasses import dataclass
    from langchain.agents import create_agent
    from langchain.agents.middleware import dynamic_prompt, ModelRequest


    @dataclass
    class ContextSchema:
        user_name: str

    @dynamic_prompt  # [!code highlight]
    def personalized_prompt(request: ModelRequest) -> str:  # [!code highlight]
        user_name = request.runtime.context.user_name
        return f"You are a helpful assistant. Address the user as {user_name}."

    agent = create_agent(
        model="anthropic:claude-sonnet-4-5",
        tools=[get_weather],
        middleware=[personalized_prompt],
        context_schema=ContextSchema
    )

    agent.invoke(
        {"messages": [{"role": "user", "content": "what is the weather in sf"}]},
        context=ContextSchema(user_name="John Smith")  # [!code highlight]
    )
    ```

    有关详细信息，请参见[代理](/oss/python/langchain/agents)。
  </选项卡>

  <选项卡标题="工作流节点">
    ```python  theme={null}
    from langgraph.runtime import Runtime

    def node(state: State, runtime: Runtime[ContextSchema]):  # [!code highlight]
        user_name = runtime.context.user_name
        ...
    ```

    * 有关详细信息，请参见[图API](/oss/python/langgraph/graph-api#add-runtime-configuration)。
  </选项卡>

  <选项卡标题="在工具中">
    ```python  theme={null}
    from langchain.tools import tool, ToolRuntime

    @tool
    def get_user_email(runtime: ToolRuntime[ContextSchema]) -> str:
        """Retrieve user information based on user ID."""
        # simulate fetching user info from a database
        email = get_user_email_from_db(runtime.context.user_name)  # [!code highlight]
        return email
    ```

    有关详细信息，请参见[工具调用指南](/oss/python/langchain/tools#configuration)。
  </选项卡>
</选项卡>

<提示>
  可以使用`Runtime`对象访问静态上下文以及其他实用工具，如活动存储和流写入器。
  有关详细信息，请参见[@[`Runtime`]\[langgraph.runtime.Runtime]]文档。
</提示>

<a id="state" />

## 动态运行时上下文

**动态运行时上下文**表示可变数据，这些数据可以在单次运行期间演变，并通过LangGraph状态对象进行管理。这包括对话历史、中间结果以及从工具或LLM输出派生的值。在LangGraph中，状态对象在运行期间充当[短期记忆](/oss/python/concepts/memory)。

<选项卡>
  <选项卡标题="在代理中">
    示例展示了如何将状态合并到代理**提示**中。
  
    代理的**工具**也可以访问状态，这些工具可以根据需要读取或更新状态。有关详细信息，请参见[工具调用指南](/oss/python/langchain/tools#short-term-memory)。

    ```python  theme={null}
    from langchain.agents import create_agent
    from langchain.agents.middleware import dynamic_prompt, ModelRequest
    from langchain.agents import AgentState


    class CustomState(AgentState):  # [!code highlight]
        user_name: str

    @dynamic_prompt  # [!code highlight]
    def personalized_prompt(request: ModelRequest) -> str:  # [!code highlight]
        user_name = request.state.get("user_name", "User")
        return f"You are a helpful assistant. User's name is {user_name}"

    agent = create_agent(
        model="anthropic:claude-sonnet-4-5",
        tools=[...],
        state_schema=CustomState,  # [!code highlight]
        middleware=[personalized_prompt],  # [!code highlight]
    )

    agent.invoke({
        "messages": "hi!",
        "user_name": "John Smith"
    })
    ```
  </选项卡>

  <选项卡标题="在工作流中">
    ```python  theme={null}
    from typing_extensions import TypedDict
    from langchain.messages import AnyMessage
    from langgraph.graph import StateGraph

    class CustomState(TypedDict):  # [!code highlight]
        messages: list[AnyMessage]
        extra_field: int

    def node(state: CustomState):  # [!code highlight]
        messages = state["messages"]
        ...
        return {  # [!code highlight]
            "extra_field": state["extra_field"] + 1  # [!code highlight]
        }

    builder = StateGraph(State)
    builder.add_node(node)
    builder.set_entry_point("node")
    graph = builder.compile()
    ```
  </选项卡>
</选项卡>

<提示>
  **开启内存**

  有关如何启用内存的更多详细信息，请参见[内存指南](/oss/python/langgraph/add-memory)。这是一个强大的功能，允许跨多次调用持久化代理的状态。否则，状态仅限于单次运行。
</提示>

<a id="store" />

## 动态跨对话上下文

**动态跨对话上下文**表示持久且可变的数据，这些数据跨越多个对话或会话，并通过LangGraph存储进行管理。这包括用户配置文件、偏好设置和历史交互。LangGraph存储充当[长期记忆](/oss/python/concepts/memory#long-term-memory)，跨越多次运行。这可用于读取或更新持久性事实（例如，用户配置文件、偏好设置、先前的交互）。

## 另请参见

* [内存概念概述](/oss/python/concepts/memory)
* [LangChain中的短期记忆](/oss/python/langchain/short-term-memory)
* [LangChain中的长期记忆](/oss/python/langchain/long-term-memory)
* [LangGraph中的内存](/oss/python/langgraph/add-memory)

***

<插图 icon="pen-to-square" iconType="regular">
  [在GitHub上编辑此页面的源代码。](https://github.com/langchain-ai/docs/edit/main/src/oss/concepts/context.mdx)
</插图>

<提示 icon="terminal" iconType="regular">
  通过MCP将[这些文档以编程方式连接](/use-these-docs)到Claude、VSCode等，以获取实时答案。
</提示>