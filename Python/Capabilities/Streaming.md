# 流式处理

LangGraph 实现了一个流式处理系统，用于展示实时更新。流式处理对于增强基于 LLM 构建的应用程序的响应能力至关重要。通过在完整响应就绪之前逐步显示输出，流式处理显著改善了用户体验（UX），特别是在处理 LLM 延迟时。

LangGraph 流式处理可以实现的功能：

* <Icon icon="share-nodes" size={16} /> [**流式图状态**](#stream-graph-state) — 通过 `updates` 和 `values` 模式获取状态更新/值。
* <Icon icon="square-poll-horizontal" size={16} /> [**流式子图输出**](#stream-subgraph-outputs) — 包含父图和任何嵌套子图的输出。
* <Icon icon="square-binary" size={16} /> [**流式 LLM 令牌**](#messages) — 从任何位置捕获令牌流：节点内、子图内或工具内。
* <Icon icon="table" size={16} /> [**流式自定义数据**](#stream-custom-data) — 直接从工具函数发送自定义更新或进度信号。
* <Icon icon="layer-plus" size={16} /> [**使用多种流式模式**](#stream-multiple-modes) — 从 `values`（完整状态）、`updates`（状态增量）、`messages`（LLM 令牌 + 元数据）、`custom`（任意用户数据）或 `debug`（详细跟踪）中选择。

## 支持的流式模式

将以下一个或多个流式模式作为列表传递给 [`stream`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.CompiledStateGraph.stream) 或 [`astream`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.CompiledStateGraph.astream) 方法：

| 模式 | 描述 |
| ---------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `values` | 在图的每个步骤之后流式传输状态的完整值。 |
| `updates` | 在图的每个步骤之后流式传输状态的更新。如果在同一步骤中进行了多次更新（例如，运行多个节点），这些更新将分别流式传输。 |
| `custom` | 从图节点内部流式传输自定义数据。 |
| `messages` | 从调用 LLM 的任何图节点流式传输 2 元组（LLM 令牌，元数据）。 |
| `debug` | 在图的执行过程中尽可能多地流式传输信息。 |

## 基本使用示例

LangGraph 图提供了 [`stream`](https://reference.langchain.com/python/langgraph/pregel/#langgraph.pregel.Pregel.stream)（同步）和 [`astream`](https://reference.langchain.com/python/langgraph/pregel/#langgraph.pregel.Pregel.astream)（异步）方法，以迭代器形式生成流式输出。

```python  theme={null}
for chunk in graph.stream(inputs, stream_mode="updates"):
    print(chunk)
```

<Accordion title="扩展示例：流式更新">
  ```python  theme={null}
  from typing import TypedDict
  from langgraph.graph import StateGraph, START, END

  class State(TypedDict):
      topic: str
      joke: str

  def refine_topic(state: State):
      return {"topic": state["topic"] + " and cats"}

  def generate_joke(state: State):
      return {"joke": f"This is a joke about {state['topic']}"}

  graph = (
      StateGraph(State)
      .add_node(refine_topic)
      .add_node(generate_joke)
      .add_edge(START, "refine_topic")
      .add_edge("refine_topic", "generate_joke")
      .add_edge("generate_joke", END)
      .compile()
  )

  # stream() 方法返回一个迭代器，生成流式输出
  for chunk in graph.stream(  # [!code highlight]
      {"topic": "ice cream"},
      # 设置 stream_mode="updates" 以仅在每次节点后流式传输图状态的更新
      # 其他流式模式也可用。详情请参见支持的流式模式
      stream_mode="updates",  # [!code highlight]
  ):
      print(chunk)
  ```

  ```output  theme={null}
  {'refineTopic': {'topic': 'ice cream and cats'}}
  {'generateJoke': {'joke': 'This is a joke about ice cream and cats'}}
  ```
</Accordion>

## 多模式流式传输

您可以传递一个列表作为 `stream_mode` 参数，同时流式传输多种模式。

流式输出将是 `(mode, chunk)` 的元组，其中 `mode` 是流式模式的名称，`chunk` 是该模式流式传输的数据。

```python  theme={null}
for mode, chunk in graph.stream(inputs, stream_mode=["updates", "custom"]):
    print(chunk)
```

## 流式图状态

使用流式模式 `updates` 和 `values` 来流式传输图的执行状态。

* `updates` 流式传输图每个步骤后状态的**更新**。
* `values` 流式传输图每个步骤后状态的**完整值**。

```python  theme={null}
from typing import TypedDict
from langgraph.graph import StateGraph, START, END


class State(TypedDict):
  topic: str
  joke: str


def refine_topic(state: State):
    return {"topic": state["topic"] + " and cats"}


def generate_joke(state: State):
    return {"joke": f"This is a joke about {state['topic']}"}

graph = (
  StateGraph(State)
  .add_node(refine_topic)
  .add_node(generate_joke)
  .add_edge(START, "refine_topic")
  .add_edge("refine_topic", "generate_joke")
  .add_edge("generate_joke", END)
  .compile()
)
```

<Tabs>
  <Tab title="updates">
    使用此选项流式传输节点在每个步骤后返回的**状态更新**。流式输出包括节点名称和更新内容。

    ```python  theme={null}
    for chunk in graph.stream(
        {"topic": "ice cream"},
        stream_mode="updates",  # [!code highlight]
    ):
        print(chunk)
    ```
  </Tab>

  <Tab title="values">
    使用此选项流式传输图每个步骤后的**完整状态**。

    ```python  theme={null}
    for chunk in graph.stream(
        {"topic": "ice cream"},
        stream_mode="values",  # [!code highlight]
    ):
        print(chunk)
    ```
  </Tab>
</Tabs>

## 流式子图输出

要在流式输出中包含来自[子图](/oss/python/langgraph/use-subgraphs)的输出，可以在父图的 `.stream()` 方法中设置 `subgraphs=True`。这将流式传输父图和任何子图的输出。

输出将作为 `(namespace, data)` 元组流式传输，其中 `namespace` 是调用子图的节点的路径元组，例如 `("parent_node:<task_id>", "child_node:<task_id>")`。

```python  theme={null}
for chunk in graph.stream(
    {"foo": "foo"},
    # 设置 subgraphs=True 以流式传输子图的输出
    subgraphs=True,  # [!code highlight]
    stream_mode="updates",
):
    print(chunk)
```

<Accordion title="扩展示例：从子图流式传输">
  ```python  theme={null}
  from langgraph.graph import START, StateGraph
  from typing import TypedDict

  # 定义子图
  class SubgraphState(TypedDict):
      foo: str  # 注意此键与父图状态共享
      bar: str

  def subgraph_node_1(state: SubgraphState):
      return {"bar": "bar"}

  def subgraph_node_2(state: SubgraphState):
      return {"foo": state["foo"] + state["bar"]}

  subgraph_builder = StateGraph(SubgraphState)
  subgraph_builder.add_node(subgraph_node_1)
  subgraph_builder.add_node(subgraph_node_2)
  subgraph_builder.add_edge(START, "subgraph_node_1")
  subgraph_builder.add_edge("subgraph_node_1", "subgraph_node_2")
  subgraph = subgraph_builder.compile()

  # 定义父图
  class ParentState(TypedDict):
      foo: str

  def node_1(state: ParentState):
      return {"foo": "hi! " + state["foo"]}

  builder = StateGraph(ParentState)
  builder.add_node("node_1", node_1)
  builder.add_node("node_2", subgraph)
  builder.add_edge(START, "node_1")
  builder.add_edge("node_1", "node_2")
  graph = builder.compile()

  for chunk in graph.stream(
      {"foo": "foo"},
      stream_mode="updates",
      # 设置 subgraphs=True 以流式传输子图的输出
      subgraphs=True,  # [!code highlight]
  ):
      print(chunk)
  ```

  ```
  ((), {'node_1': {'foo': 'hi! foo'}})
  (('node_2:dfddc4ba-c3c5-6887-5012-a243b5b377c2',), {'subgraph_node_1': {'bar': 'bar'}})
  (('node_2:dfddc4ba-c3c5-6887-5012-a243b5b377c2',), {'subgraph_node_2': {'foo': 'hi! foobar'}})
  ((), {'node_2': {'foo': 'hi! foobar'}})
  ```

  **注意**我们不仅接收节点更新，还接收命名空间，它告诉我们我们正在从哪个图（或子图）流式传输。
</Accordion>

<a id="debug" />

### 调试

使用 `debug` 流式模式在图的执行过程中尽可能多地流式传输信息。流式输出包括节点名称和完整状态。

```python  theme={null}
for chunk in graph.stream(
    {"topic": "ice cream"},
    stream_mode="debug",  # [!code highlight]
):
    print(chunk)
```

<a id="messages" />

## LLM 令牌

使用 `messages` 流式模式从图的任何部分**逐个令牌**流式传输大型语言模型（LLM）的输出，包括节点、工具、子图或任务。

来自 [`messages` 模式](#supported-stream-modes) 的流式输出是一个元组 `(message_chunk, metadata)`，其中：

* `message_chunk`：来自 LLM 的令牌或消息片段。
* `metadata`：包含有关图节点和 LLM 调用详细信息的字典。

> 如果您的 LLM 不可用作 LangChain 集成，您可以使用 `custom` 模式流式传输其输出。详情请参见[与任何 LLM 一起使用](#use-with-any-llm)。

<Warning>
  **Python < 3.11 中异步需要手动配置**
  在 Python < 3.11 上使用异步代码时，必须显式将 [`RunnableConfig`](https://reference.langchain.com/python/langchain_core/runnables/#langchain_core.runnables.RunnableConfig) 传递给 `ainvoke()` 以启用正确的流式处理。详情请参见[Python < 3.11 异步](#async)或升级到 Python 3.11+。
</Warning>

```python  theme={null}
from dataclasses import dataclass

from langchain.chat_models import init_chat_model
from langgraph.graph import StateGraph, START


@dataclass
class MyState:
    topic: str
    joke: str = ""


model = init_chat_model(model="openai:gpt-4o-mini")

def call_model(state: MyState):
    """调用 LLM 生成关于主题的笑话"""
    # 注意即使在使用 .invoke 而不是 .stream 运行 LLM 时也会发出消息事件
    model_response = model.invoke(  # [!code highlight]
        [
            {"role": "user", "content": f"Generate a joke about {state.topic}"}
        ]
    )
    return {"joke": model_response.content}

graph = (
    StateGraph(MyState)
    .add_node(call_model)
    .add_edge(START, "call_model")
    .compile()
)

# "messages" 流式模式返回一个元组迭代器 (message_chunk, metadata)
# 其中 message_chunk 是 LLM 流式传输的令牌，metadata 是一个包含有关调用 LLM 的图节点信息的字典
for message_chunk, metadata in graph.stream(
    {"topic": "ice cream"},
    stream_mode="messages",  # [!code highlight]
):
    if message_chunk.content:
        print(message_chunk.content, end="|", flush=True)
```

#### 按 LLM 调用过滤

您可以将 `tags` 与 LLM 调用关联，以根据 LLM 调用过滤流式传输的令牌。

```python  theme={null}
from langchain.chat_models import init_chat_model

# model_1 标记为 "joke"
model_1 = init_chat_model(model="openai:gpt-4o-mini", tags=['joke'])
# model_2 标记为 "poem"
model_2 = init_chat_model(model="openai:gpt-4o-mini", tags=['poem'])

graph = ... # 定义一个使用这些 LLM 的图

# stream_mode 设置为 "messages" 以流式传输 LLM 令牌
# metadata 包含有关 LLM 调用的信息，包括标记
async for msg, metadata in graph.astream(
    {"topic": "cats"},
    stream_mode="messages",  # [!code highlight]
):
    # 按 metadata 中的 tags 字段过滤流式传输的令牌，仅包含
    # 标记为 "joke" 的 LLM 调用的令牌
    if metadata["tags"] == ["joke"]:
        print(msg.content, end="|", flush=True)
```

<Accordion title="扩展示例：按标记过滤">
  ```python  theme={null}
  from typing import TypedDict

  from langchain.chat_models import init_chat_model
  from langgraph.graph import START, StateGraph

  # joke_model 标记为 "joke"
  joke_model = init_chat_model(model="openai:gpt-4o-mini", tags=["joke"])
  # poem_model 标记为 "poem"
  poem_model = init_chat_model(model="openai:gpt-4o-mini", tags=["poem"])


  class State(TypedDict):
        topic: str
        joke: str
        poem: str


  async def call_model(state, config):
        topic = state["topic"]
        print("Writing joke...")
        # 注意：在 python < 3.11 中显式传递配置是必需的
        # 因为那时没有添加上下文变量支持：https://docs.python.org/3/library/asyncio-task.html#creating-tasks
        # 显式传递配置以确保正确传播上下文变量
        # 在 Python < 3.11 上使用异步代码时这是必需的。更多详情请参见异步部分
        joke_response = await joke_model.ainvoke(
              [{"role": "user", "content": f"Write a joke about {topic}"}],
              config,
        )
        print("\n\nWriting poem...")
        poem_response = await poem_model.ainvoke(
              [{"role": "user", "content": f"Write a short poem about {topic}"}],
              config,
        )
        return {"joke": joke_response.content, "poem": poem_response.content}


  graph = (
        StateGraph(State)
        .add_node(call_model)
        .add_edge(START, "call_model")
        .compile()
  )

  # stream_mode 设置为 "messages" 以流式传输 LLM 令牌
  # metadata 包含有关 LLM 调用的信息，包括标记
  async for msg, metadata in graph.astream(
        {"topic": "cats"},
        stream_mode="messages",
  ):
      if metadata["tags"] == ["joke"]:
          print(msg.content, end="|", flush=True)
  ```
</Accordion>

#### 按节点过滤

要仅从特定节点流式传输令牌，请使用 `stream_mode="messages"` 并根据流式传输的元数据中的 `langgraph_node` 字段过滤输出：

```python  theme={null}
# "messages" 流式模式返回一个元组 (message_chunk, metadata)
# 其中 message_chunk 是 LLM 流式传输的令牌，metadata 是一个包含有关调用 LLM 的图节点信息的字典
for msg, metadata in graph.stream(
    inputs,
    stream_mode="messages",  # [!code highlight]
):
    # 按 metadata 中的 langgraph_node 字段过滤流式传输的令牌
    # 仅包含来自指定节点的令牌
    if msg.content and metadata["langgraph_node"] == "some_node_name":
        ...
```

<Accordion title="扩展示例：从特定节点流式传输 LLM 令牌">
  ```python  theme={null}
  from typing import TypedDict
  from langgraph.graph import START, StateGraph
  from langchain_openai import ChatOpenAI

  model = ChatOpenAI(model="gpt-4o-mini")


  class State(TypedDict):
        topic: str
        joke: str
        poem: str


  def write_joke(state: State):
        topic = state["topic"]
        joke_response = model.invoke(
              [{"role": "user", "content": f"Write a joke about {topic}"}]
        )
        return {"joke": joke_response.content}


  def write_poem(state: State):
        topic = state["topic"]
        poem_response = model.invoke(
              [{"role": "user", "content": f"Write a short poem about {topic}"}]
        )
        return {"poem": poem_response.content}


  graph = (
        StateGraph(State)
        .add_node(write_joke)
        .add_node(write_poem)
        # 并发编写笑话和诗
        .add_edge(START, "write_joke")
        .add_edge(START, "write_poem")
        .compile()
  )

  # "messages" 流式模式返回一个元组 (message_chunk, metadata)
  # 其中 message_chunk 是 LLM 流式传输的令牌，metadata 是一个包含有关调用 LLM 的图节点信息的字典
  for msg, metadata in graph.stream(
      {"topic": "cats"},
      stream_mode="messages",  # [!code highlight]
  ):
      # 按 metadata 中的 langgraph_node 字段过滤流式传输的令牌
      # 仅包含来自 write_poem 节点的令牌
      if msg.content and metadata["langgraph_node"] == "write_poem":
          print(msg.content, end="|", flush=True)
  ```
</Accordion>

## 流式自定义数据

要从 LangGraph 节点或工具内部发送**自定义用户定义数据**，请按照以下步骤操作：

1. 使用 [`get_stream_writer`](https://reference.langchain.com/python/langgraph/config/#langgraph.config.get_stream_writer) 访问流式写入器并发出自定义数据。
2. 在调用 `.stream()` 或 `.astream()` 时设置 `stream_mode="custom"` 以在流中获取自定义数据。您可以组合多种模式（例如 `["updates", "custom"]`），但至少必须有一个是 `"custom"`。

<Warning>
  **Python < 3.11 的异步中没有 [`get_stream_writer`](https://reference.langchain.com/python/langgraph/config/#langgraph.config.get_stream_writer)**
  在 Python < 3.11 上运行的异步代码中，[`get_stream_writer`](https://reference.langchain.com/python/langgraph/config/#langgraph.config.get_stream_writer) 将不起作用。
  相反，请在您的节点或工具中添加一个 `writer` 参数并手动传递它。
  详情请参见[Python < 3.11 异步](#async)的使用示例。
</Warning>

<Tabs>
  <Tab title="node">
    ```python  theme={null}
    from typing import TypedDict
    from langgraph.config import get_stream_writer
    from langgraph.graph import StateGraph, START

    class State(TypedDict):
        query: str
        answer: str

    def node(state: State):
        # 获取流式写入器以发送自定义数据
        writer = get_stream_writer()
        # 发出自定义键值对（例如进度更新）
        writer({"custom_key": "在节点内生成自定义数据"})
        return {"answer": "一些数据"}

    graph = (
        StateGraph(State)
        .add_node(node)
        .add_edge(START, "node")
        .compile()
    )

    inputs = {"query": "示例"}

    # 设置 stream_mode="custom" 以在流中接收自定义数据
    for chunk in graph.stream(inputs, stream_mode="custom"):
        print(chunk)
    ```
  </Tab>

  <Tab title="tool">
    ```python  theme={null}
    from langchain.tools import tool
    from langgraph.config import get_stream_writer

    @tool
    def query_database(query: str) -> str:
        """查询数据库。"""
        # 访问流式写入器以发送自定义数据
        writer = get_stream_writer()  # [!code highlight]
        # 发出自定义键值对（例如进度更新）
        writer({"data": "已检索 0/100 条记录", "type": "progress"})  # [!code highlight]
        # 执行查询
        # 发出另一个自定义键值对
        writer({"data": "已检索 100/100 条记录", "type": "progress"})
        return "一些答案"


    graph = ... # 定义一个使用此工具的图

    # 设置 stream_mode="custom" 以在流中接收自定义数据
    for chunk in graph.stream(inputs, stream_mode="custom"):
        print(chunk)
    ```
  </Tab>
</Tabs>

## 与任何 LLM 一起使用

您可以使用 `stream_mode="custom"` 来流式传输**任何 LLM API** 的数据 — 即使该 API**没有**实现 LangChain 聊天模型接口。

这让您可以集成原始的 LLM 客户端或提供自己的流式接口的外部服务，使 LangGraph 对于自定义设置具有高度的灵活性。

```python  theme={null}
from langgraph.config import get_stream_writer

def call_arbitrary_model(state):
    """调用任意模型并流式传输输出的示例节点"""
    # 获取流式写入器以发送自定义数据
    writer = get_stream_writer()  # [!code highlight]
    # 假设您有一个生成块的流式客户端
    # 使用您的自定义流式客户端生成 LLM 令牌
    for chunk in your_custom_streaming_client(state["topic"]):
        # 使用写入器将自定义数据发送到流
        writer({"custom_llm_chunk": chunk})  # [!code highlight]
    return {"result": "已完成"}

graph = (
    StateGraph(State)
    .add_node(call_arbitrary_model)
    # 根据需要添加其他节点和边
    .compile()
)
# 设置 stream_mode="custom" 以在流中接收自定义数据
for chunk in graph.stream(
    {"topic": "cats"},
    stream_mode="custom",  # [!code highlight]

):
    # 块将包含从 llm 流式传输的自定义数据
    print(chunk)
```

<Accordion title="扩展示例：流式传输任意聊天模型">
  ```python  theme={null}
  import operator
  import json

  from typing import TypedDict
  from typing_extensions import Annotated
  from langgraph.graph import StateGraph, START

  from openai import AsyncOpenAI

  openai_client = AsyncOpenAI()
  model_name = "gpt-4o-mini"


  async def stream_tokens(model_name: str, messages: list[dict]):
      response = await openai_client.chat.completions.create(
          messages=messages, model=model_name, stream=True
      )
      role = None
      async for chunk in response:
          delta = chunk.choices[0].delta

          if delta.role is not None:
              role = delta.role

          if delta.content:
              yield {"role": role, "content": delta.content}


  # 这是我们的工具
  async def get_items(place: str) -> str:
      """使用此工具列出您询问的地方可能找到的物品。"""
      writer = get_stream_writer()
      response = ""
      async for msg_chunk in stream_tokens(
          model_name,
          [
              {
                  "role": "user",
                  "content": (
                      "你能告诉我我可能在以下地方找到什么样的物品吗？"
                      f"'{place}'。"
                      "列出至少 3 个这样的物品，用逗号分隔。"
                      "并包含每个物品的简要描述。"
                  ),
              }
          ],
      ):
          response += msg_chunk["content"]
          writer(msg_chunk)

      return response


  class State(TypedDict):
      messages: Annotated[list[dict], operator.add]


  # 这是工具调用图节点
  async def call_tool(state: State):
      ai_message = state["messages"][-1]
      tool_call = ai_message["tool_calls"][-1]

      function_name = tool_call["function"]["name"]
      if function_name != "get_items":
          raise ValueError(f"不支持工具 {function_name}")

      function_arguments = tool_call["function"]["arguments"]
      arguments = json.loads(function_arguments)

      function_response = await get_items(**arguments)
      tool_message = {
          "tool_call_id": tool_call["id"],
          "role": "tool",
          "name": function_name,
          "content": function_response,
      }
      return {"messages": [tool_message]}


  graph = (
      StateGraph(State)
      .add_node(call_tool)
      .add_edge(START, "call_tool")
      .compile()
  )
  ```

  让我们使用包含工具调用的 [`AIMessage`](https://reference.langchain.com/python/langchain/messages/#langchain.messages.AIMessage) 调用图：

  ```python  theme={null}
  inputs = {
      "messages": [
          {
              "content": None,
              "role": "assistant",
              "tool_calls": [
                  {
                      "id": "1",
                      "function": {
                          "arguments": '{"place":"bedroom"}',
                          "name": "get_items",
                      },
                      "type": "function",
                  }
              ],
          }
      ]
  }

  async for chunk in graph.astream(
      inputs,
      stream_mode="custom",
  ):
      print(chunk["content"], end="|", flush=True)
  ```
</Accordion>

## 为特定聊天模型禁用流式处理

如果您的应用程序混合了支持流式处理的模型和不支持的模型，您可能需要为不支持的模型显式禁用流式处理。

在初始化模型时设置 `disable_streaming=True`。

<Tabs>
  <Tab title="init_chat_model">
    ```python  theme={null}
    from langchain.chat_models import init_chat_model

    model = init_chat_model(
        "anthropic:claude-sonnet-4-5",
        # 设置 disable_streaming=True 以禁用聊天模型的流式处理
        disable_streaming=True  # [!code highlight]

    )
    ```
  </Tab>

  <Tab title="chat model interface">
    ```python  theme={null}
    from langchain_openai import ChatOpenAI

    # 设置 disable_streaming=True 以禁用聊天模型的流式处理
    model = ChatOpenAI(model="o1-preview", disable_streaming=True)
    ```
  </Tab>
</Tabs>

<a id="async" />

### Python < 3.11 异步

在 Python < 3.11 版本中，[asyncio 任务](https://docs.python.org/3/library/asyncio-task.html#asyncio.create_task)不支持 `context` 参数。
这限制了 LangGraph 自动传播上下文的能力，并以两种关键方式影响了 LangGraph 的流式处理机制：

1. 您**必须**显式将 [`RunnableConfig`](https://python.langchain.com/docs/concepts/runnables/#runnableconfig) 传递给异步 LLM 调用（例如 `ainvoke()`），因为回调不会自动传播。
2. 您**不能**在异步节点或工具中使用 [`get_stream_writer`](https://reference.langchain.com/python/langgraph/config/#langgraph.config.get_stream_writer) — 您必须直接传递 `writer` 参数。

<Accordion title="扩展示例：异步 LLM 调用与手动配置">
  ```python  theme={null}
  from typing import TypedDict
  from langgraph.graph import START, StateGraph
  from langchain.chat_models import init_chat_model

  model = init_chat_model(model="openai:gpt-4o-mini")

  class State(TypedDict):
      topic: str
      joke: str

  # 在异步节点函数中将 config 作为参数接受
  async def call_model(state, config):
      topic = state["topic"]
      print("生成笑话...")
      # 将 config 传递给 model.ainvoke() 以确保正确的上下文传播
      joke_response = await model.ainvoke(  # [!code highlight]
          [{"role": "user", "content": f"Write a joke about {topic}"}],
          config,
      )
      return {"joke": joke_response.content}

  graph = (
      StateGraph(State)
      .add_node(call_model)
      .add_edge(START, "call_model")
      .compile()
  )

  # 设置 stream_mode="messages" 以流式传输 LLM 令牌
  async for chunk, metadata in graph.astream(
      {"topic": "ice cream"},
      stream_mode="messages",  # [!code highlight]
  ):
      if chunk.content:
          print(chunk.content, end="|", flush=True)
  ```
</Accordion>

<Accordion title="扩展示例：异步自定义流式处理与流式写入器">
  ```python  theme={null}
  from typing import TypedDict
  from langgraph.types import StreamWriter

  class State(TypedDict):
        topic: str
        joke: str

  # 在异步节点或工具的函数签名中将 writer 添加为参数
  # LangGraph 将自动将流式写入器传递给函数
  async def generate_joke(state: State, writer: StreamWriter):  # [!code highlight]
        writer({"custom_key": "生成笑话时流式传输自定义数据"})
        return {"joke": f"This is a joke about {state['topic']}"}

  graph = (
        StateGraph(State)
        .add_node(generate_joke)
        .add_edge(START, "generate_joke")
        .compile()
  )

  # 设置 stream_mode="custom" 以在流中接收自定义数据  # [!code highlight]
  async for chunk in graph.astream(
        {"topic": "ice cream"},
        stream_mode="custom",
  ):
        print(chunk)
  ```
</Accordion>

***

<Callout icon="pen-to-square" iconType="regular">
  [在 GitHub 上编辑此页面源代码。](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/streaming.mdx)
</Callout>

<Tip icon="terminal" iconType="regular">
  [通过 MCP 以编程方式连接这些文档](/use-these-docs) 到 Claude、VSCode 等以获取实时答案。
</Tip>