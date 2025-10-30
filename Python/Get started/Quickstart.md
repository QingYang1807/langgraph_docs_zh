# 快速入门

本快速入门演示如何使用 LangGraph Graph API 或 Functional API 构建一个计算器代理。

* 如果你更喜欢将代理定义为节点和边的图，请[使用 Graph API](#use-the-graph-api)。
* 如果你更喜欢将代理定义为单个函数，请[使用 Functional API](#use-the-functional-api)。


有关概念信息，请参阅[Graph API 概览](/oss/python/langgraph/graph-api)和[Functional API 概览](/oss/python/langgraph/functional-api)。



在此示例中，你需要设置一个[Claude (Anthropic)](https://www.anthropic.com/)账户并获取 API 密钥。然后，在终端中设置 `ANTHROPIC_API_KEY` 环境变量。

## 使用 Graph API
## 1. 定义工具和模型

在本示例中，我们将使用 Claude Sonnet 4.5 模型，并定义加法、乘法和除法的工具。

```python
from langchain.tools import tool
from langchain.chat_models import init_chat_model


model = init_chat_model(
    "anthropic:claude-sonnet-4-5",
    temperature=0
)


# 定义工具
@tool
def multiply(a: int, b: int) -> int:
    """将 `a` 和 `b` 相乘。

    Args:
    a: 第一个整数
    b: 第二个整数
    """
    return a * b


@tool
def add(a: int, b: int) -> int:
    """将 `a` 和 `b` 相加。

    Args:
    a: 第一个整数
    b: 第二个整数
    """
    return a + b


@tool
def divide(a: int, b: int) -> float:
    """将 `a` 除以 `b`。

    Args:
    a: 第一个整数
    b: 第二个整数
    """
    return a / b


# 增强 LLM 的工具能力
tools = [add, multiply, divide]
tools_by_name = {tool.name: tool for tool in tools}
model_with_tools = model.bind_tools(tools)
```

## 2. 定义状态

图的状态用于存储消息和 LLM 调用次数。

LangGraph 中的状态在代理执行期间会持续存在。

使用带有 `operator.add` 的 `Annotated` 类型确保新消息附加到现有列表中，而不是替换它们。

```python
from langchain.messages import AnyMessage
from typing_extensions import TypedDict, Annotated
import operator


class MessagesState(TypedDict):
    messages: Annotated[list[AnyMessage], operator.add]
    llm_calls: int
```

## 3. 定义模型节点

模型节点用于调用 LLM 并决定是否调用工具。

    ```python
from langchain.messages import SystemMessage


def llm_call(state: dict):
    """LLM 决定是否调用工具"""

    return {
    "messages": [
    model_with_tools.invoke(
    [
    SystemMessage(
    content="你是一个有帮助的助手，负责在一组输入上执行算术运算。"
    )
    ]
    + state["messages"]
    )
    ],
    "llm_calls": state.get('llm_calls', 0) + 1
    }
```

## 4. 定义工具节点

工具节点用于调用工具并返回结果。

```python
from langchain.messages import ToolMessage


def tool_node(state: dict):
    """执行工具调用"""

    result = []
    for tool_call in state["messages"][-1].tool_calls:
        tool = tools_by_name[tool_call["name"]]
        observation = tool.invoke(tool_call["args"])
        result.append(ToolMessage(content=observation, tool_call_id=tool_call["id"]))
    return {"messages": result}
```

## 5. 定义结束逻辑

条件边函数用于根据 LLM 是否进行了工具调用，路由到工具节点或结束。

```python
from typing import Literal
from langgraph.graph import StateGraph, START, END


def should_continue(state: MessagesState) -> Literal["tool_node", END]:
    """根据 LLM 是否进行工具调用决定是否继续循环或停止"""

    messages = state["messages"]
    last_message = messages[-1]

    # 如果 LLM 进行了工具调用，则执行操作
    if last_message.tool_calls:
        return "tool_node"

    # 否则，我们停止（回复用户）
    return END
```

## 6. 构建并编译代理

代理使用 [`StateGraph`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.StateGraph) 类构建，并使用 [`compile`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.StateGraph.compile) 方法编译。

```python
# 构建工作流
agent_builder = StateGraph(MessagesState)

# 添加节点
agent_builder.add_node("llm_call", llm_call)
agent_builder.add_node("tool_node", tool_node)

# 添加边以连接节点
agent_builder.add_edge(START, "llm_call")
agent_builder.add_conditional_edges(
    "llm_call",
    should_continue,
    ["tool_node", END]
)
agent_builder.add_edge("tool_node", "llm_call")

# 编译代理
agent = agent_builder.compile()

# 显示代理
from IPython.display import Image, display
display(Image(agent.get_graph(xray=True).draw_mermaid_png()))

# 调用
from langchain.messages import HumanMessage
messages = [HumanMessage(content="将 3 和 4 相加。")]
messages = agent.invoke({"messages": messages})
for m in messages["messages"]:
    m.pretty_print()
```

恭喜！你已经使用 LangGraph Graph API 构建了你的第一个代理。

## 完整代码示例
```python
# 步骤 1: 定义工具和模型

from langchain.tools import tool
from langchain.chat_models import init_chat_model


model = init_chat_model(
    "anthropic:claude-sonnet-4-5",
          temperature=0
)


# 定义工具
@tool
def multiply(a: int, b: int) -> int:
   """将 `a` 和 `b` 相乘。

   Args:
      a: 第一个整数
      b: 第二个整数
   """
  return a * b


@tool
def add(a: int, b: int) -> int:
    """将 `a` 和 `b` 相加。

    Args:
        a: 第一个整数
        b: 第二个整数
    """
    return a + b


@tool
def divide(a: int, b: int) -> float:
      """将 `a` 除以 `b`。

      Args:
      a: 第一个整数
      b: 第二个整数
      """
      return a / b


      # 增强 LLM 的工具能力
      tools = [add, multiply, divide]
      tools_by_name = {tool.name: tool for tool in tools}
      model_with_tools = model.bind_tools(tools)

      # 步骤 2: 定义状态

      from langchain.messages import AnyMessage
      from typing_extensions import TypedDict, Annotated
      import operator


      class MessagesState(TypedDict):
          messages: Annotated[list[AnyMessage], operator.add]
          llm_calls: int

      # 步骤 3: 定义模型节点
      from langchain.messages import SystemMessage


      def llm_call(state: dict):
          """LLM 决定是否调用工具"""

          return {
              "messages": [
                  model_with_tools.invoke(
                      [
                          SystemMessage(
                              content="你是一个有帮助的助手，负责在一组输入上执行算术运算。"
                          )
                      ]
                      + state["messages"]
                  )
              ],
              "llm_calls": state.get('llm_calls', 0) + 1
          }


      # 步骤 4: 定义工具节点

      from langchain.messages import ToolMessage


      def tool_node(state: dict):
          """执行工具调用"""

          result = []
          for tool_call in state["messages"][-1].tool_calls:
              tool = tools_by_name[tool_call["name"]]
              observation = tool.invoke(tool_call["args"])
              result.append(ToolMessage(content=observation, tool_call_id=tool_call["id"]))
          return {"messages": result}

      # 步骤 5: 定义决定是否结束的逻辑

      from typing import Literal
      from langgraph.graph import StateGraph, START, END


      # 条件边函数，根据 LLM 是否进行工具调用路由到工具节点或结束
      def should_continue(state: MessagesState) -> Literal["tool_node", END]:
          """根据 LLM 是否进行工具调用决定是否继续循环或停止"""

          messages = state["messages"]
          last_message = messages[-1]

          # 如果 LLM 进行了工具调用，则执行操作
          if last_message.tool_calls:
              return "tool_node"

          # 否则，我们停止（回复用户）
          return END

      # 步骤 6: 构建代理

      # 构建工作流
      agent_builder = StateGraph(MessagesState)

      # 添加节点
      agent_builder.add_node("llm_call", llm_call)
      agent_builder.add_node("tool_node", tool_node)

      # 添加边以连接节点
      agent_builder.add_edge(START, "llm_call")
      agent_builder.add_conditional_edges(
          "llm_call",
          should_continue,
          ["tool_node", END]
      )
      agent_builder.add_edge("tool_node", "llm_call")

      # 编译代理
      agent = agent_builder.compile()


      from IPython.display import Image, display
      # 显示代理
      display(Image(agent.get_graph(xray=True).draw_mermaid_png()))

      # 调用
      from langchain.messages import HumanMessage
      messages = [HumanMessage(content="将 3 和 4 相加。")]
      messages = agent.invoke({"messages": messages})
      for m in messages["messages"]:
          m.pretty_print()

```

## 使用 Functional API
    ## 1. 定义工具和模型

    在本示例中，我们将使用 Claude Sonnet 4.5 模型，并定义加法、乘法和除法的工具。

    ```python
    from langchain.tools import tool
    from langchain.chat_models import init_chat_model


    model = init_chat_model(
        "anthropic:claude-sonnet-4-5",
        temperature=0
    )


    # 定义工具
    @tool
    def multiply(a: int, b: int) -> int:
        """将 `a` 和 `b` 相乘。

        Args:
            a: 第一个整数
            b: 第二个整数
        """
        return a * b


    @tool
    def add(a: int, b: int) -> int:
        """将 `a` 和 `b` 相加。

        Args:
            a: 第一个整数
            b: 第二个整数
        """
        return a + b


    @tool
    def divide(a: int, b: int) -> float:
        """将 `a` 除以 `b`。

        Args:
            a: 第一个整数
            b: 第二个整数
        """
        return a / b


    # 增强 LLM 的工具能力
    tools = [add, multiply, divide]
      tools_by_name = {tool.name: tool for tool in tools}
      model_with_tools = model.bind_tools(tools)

      from langgraph.graph import add_messages
      from langchain.messages import (
          SystemMessage,
          HumanMessage,
          ToolCall,
      )
      from langchain_core.messages import BaseMessage
      from langgraph.func import entrypoint, task
    ```

    ## 2. 定义模型节点

    模型节点用于调用 LLM 并决定是否调用工具。

> [`@task`](https://reference.langchain.com/python/langgraph/func/#langgraph.func.task) 装饰器将函数标记为可以作为代理一部分执行的任务。任务可以在入口点函数中同步或异步调用。

    ```python
    @task
    def call_llm(messages: list[BaseMessage]):
        """LLM 决定是否调用工具"""
        return model_with_tools.invoke(
            [
                SystemMessage(
                    content="你是一个有帮助的助手，负责在一组输入上执行算术运算。"
                )
            ]
            + messages
        )
    ```

    ## 3. 定义工具节点

    工具节点用于调用工具并返回结果。

    ```python
    @task
    def call_tool(tool_call: ToolCall):
        """执行工具调用"""
        tool = tools_by_name[tool_call["name"]]
        return tool.invoke(tool_call)

    ```

    ## 4. 定义代理

    代理使用 [`@entrypoint`](https://reference.langchain.com/python/langgraph/func/#langgraph.func.entrypoint) 函数构建。

    <Note>
      在 Functional API 中，你不需要显式定义节点和边，而是在单个函数内编写标准的控制流逻辑（循环、条件语句）。
    </Note>

    ```python
    @entrypoint()
    def agent(messages: list[BaseMessage]):
        model_response = call_llm(messages).result()

        while True:
            if not model_response.tool_calls:
                break

            # 执行工具
            tool_result_futures = [
                call_tool(tool_call) for tool_call in model_response.tool_calls
            ]
            tool_results = [fut.result() for fut in tool_result_futures]
            messages = add_messages(messages, [model_response, *tool_results])
            model_response = call_llm(messages).result()

        messages = add_messages(messages, model_response)
        return messages

    # 调用
    messages = [HumanMessage(content="将 3 和 4 相加。")]
    for chunk in agent.stream(messages, stream_mode="updates"):
        print(chunk)
        print("\n")
    ```

    恭喜！你已经使用 LangGraph Functional API 构建了你的第一个代理。

## 完整代码示例
      ```python
      # 步骤 1: 定义工具和模型

      from langchain.tools import tool
      from langchain.chat_models import init_chat_model


      model = init_chat_model(
          "anthropic:claude-sonnet-4-5",
          temperature=0
      )


      # 定义工具
      @tool
      def multiply(a: int, b: int) -> int:
          """将 `a` 和 `b` 相乘。

          Args:
              a: 第一个整数
              b: 第二个整数
          """
          return a * b


      @tool
      def add(a: int, b: int) -> int:
          """将 `a` 和 `b` 相加。

          Args:
              a: 第一个整数
              b: 第二个整数
          """
          return a + b


      @tool
      def divide(a: int, b: int) -> float:
          """将 `a` 除以 `b`。

          Args:
              a: 第一个整数
              b: 第二个整数
          """
          return a / b


      # 增强 LLM 的工具能力
      tools = [add, multiply, divide]
      tools_by_name = {tool.name: tool for tool in tools}
      model_with_tools = model.bind_tools(tools)

      from langgraph.graph import add_messages
      from langchain.messages import (
          SystemMessage,
          HumanMessage,
          ToolCall,
      )
      from langchain_core.messages import BaseMessage
      from langgraph.func import entrypoint, task


      # 步骤 2: 定义模型节点

      @task
      def call_llm(messages: list[BaseMessage]):
          """LLM 决定是否调用工具"""
          return model_with_tools.invoke(
              [
                  SystemMessage(
                      content="你是一个有帮助的助手，负责在一组输入上执行算术运算。"
                  )
              ]
              + messages
          )


      # 步骤 3: 定义工具节点

      @task
      def call_tool(tool_call: ToolCall):
          """执行工具调用"""
          tool = tools_by_name[tool_call["name"]]
          return tool.invoke(tool_call)


      # 步骤 4: 定义代理

      @entrypoint()
      def agent(messages: list[BaseMessage]):
          model_response = call_llm(messages).result()

          while True:
              if not model_response.tool_calls:
                  break

              # 执行工具
              tool_result_futures = [
                  call_tool(tool_call) for tool_call in model_response.tool_calls
              ]
              tool_results = [fut.result() for fut in tool_result_futures]
              messages = add_messages(messages, [model_response, *tool_results])
              model_response = call_llm(messages).result()

          messages = add_messages(messages, model_response)
          return messages

      # 调用
      messages = [HumanMessage(content="将 3 和 4 相加。")]
      for chunk in agent.stream(messages, stream_mode="updates"):
          print(chunk)
          print("\n")
```

***

<Callout icon="pen-to-square" iconType="regular">
  [在 GitHub 上编辑此页面的源代码。](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/quickstart.mdx)
</Callout>

<Tip icon="terminal" iconType="regular">
  [通过 MCP 以编程方式连接这些文档](/use-these-docs)到 Claude、VSCode 等，获取实时答案。
</Tip>