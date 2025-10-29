# 使用图API

本指南演示了LangGraph图API的基础知识。它涵盖了[状态](#define-and-update-state)的定义和更新，以及构建常见图结构的方法，如[序列](#create-a-sequence-of-steps)、[分支](#create-branches)和[循环](#create-and-control-loops)。此外，还介绍了LangGraph的控制功能，包括用于map-reduce工作流的[Send API](#map-reduce-and-the-send-api)和用于将状态更新与节点间"跳跃"相结合的[Command API](#combine-control-flow-and-state-updates-with-command)。

## 安装

安装`langgraph`：

<CodeGroup>
  ```bash pip theme={null}
  pip install -U langgraph
  ```

  ```bash uv theme={null}
  uv add langgraph
  ```
</CodeGroup>

<Tip>
  **设置LangSmith以获得更好的调试体验**
  注册[LangSmith](https://smith.langchain.com)以快速发现问题并改进您LangGraph项目的性能。LangGraph让您可以使用跟踪数据来调试、测试和监控使用LangGraph构建的LLM应用 - 更多关于如何入门的信息，请参阅[文档](https://docs.smith.langchain.com)。
</Tip>

## 定义和更新状态

这里我们展示如何在LangGraph中定义和更新[状态](/oss/python/langgraph/graph-api#state)。我们将演示：

1. 如何使用状态定义图的[模式](/oss/python/langgraph/graph-api#schema)
2. 如何使用[约简器(reducers)](/oss/python/langgraph/graph-api#reducers)控制状态更新的处理方式。

### 定义状态

LangGraph中的[状态](/oss/python/langgraph/graph-api#state)可以是`TypedDict`、`Pydantic`模型或数据类。下面我们将使用`TypedDict`。有关使用Pydantic的详细信息，请参阅[本节](#use-pydantic-models-for-graph-state)。

默认情况下，图将具有相同的输入和输出模式，而状态决定了该模式。有关如何定义不同的输入和输出模式的信息，请参阅[本节](#define-input-and-output-schemas)。

让我们考虑一个使用[消息](/oss/python/langgraph/graph-api#messagesstate)的简单示例。这是许多LLM应用程序状态的通用表述形式。有关更多详细信息，请参阅我们的[概念页面](/oss/python/langgraph/graph-api#working-with-messages-in-graph-state)。

```python  theme={null}
from langchain.messages import AnyMessage
from typing_extensions import TypedDict

class State(TypedDict):
    messages: list[AnyMessage]
    extra_field: int
```

此状态跟踪一个[消息](https://python.langchain.com/docs/concepts/messages/)对象列表，以及一个额外的整数字段。

### 更新状态

让我们构建一个包含单个节点的示例图。我们的[节点](/oss/python/langgraph/graph-api#nodes)只是一个Python函数，它读取图的状态并对其进行更新。该函数的第一个参数始终是状态：

```python  theme={null}
from langchain.messages import AIMessage

def node(state: State):
    messages = state["messages"]
    new_message = AIMessage("Hello!")
    return {"messages": messages + [new_message], "extra_field": 10}
```

此节点只是将一个消息附加到我们的消息列表中，并填充一个额外字段。

<Warning>
  节点应该直接返回对状态的更新，而不是修改状态。
</Warning>

接下来，我们定义一个包含此节点的简单图。我们使用[`StateGraph`](/oss/python/langgraph/graph-api#stategraph)定义一个操作此状态的图。然后使用[`add_node`](/oss/python/langgraph/graph-api#nodes)填充我们的图。

```python  theme={null}
from langgraph.graph import StateGraph

builder = StateGraph(State)
builder.add_node(node)
builder.set_entry_point("node")
graph = builder.compile()
```

LangGraph提供了内置工具来可视化您的图。让我们检查一下我们的图。有关可视化的详细信息，请参阅[本节](#visualize-your-graph)。

```python  theme={null}
from IPython.display import Image, display

display(Image(graph.get_graph().draw_mermaid_png()))
```

<img src="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_1.png?fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=cf3d978b707847e166d5ed15bc7cbbe4" alt="具有单个节点的简单图" data-og-width="107" width="107" data-og-height="134" height="134" data-path="oss/images/graph_api_image_1.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_1.png?w=280&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=498bbdb0192eb26ab115d51b53fcb64c 280w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_1.png?w=560&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=94cbad4b92d5b887dff2bfbb6f8e0c6c 560w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_1.png?w=840&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=d90d58640d49e3fd4e558ab56acf4817 840w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_1.png?w=1100&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=cad59990b0c551a2aa96b684b102b953 1100w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_1.png?w=1650&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=318736f22c69f66c48f4189db3e39235 1650w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_1.png?w=2500&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=6740141ec001a9a4275cecfac67b9c55 2500w" />

在这种情况下，我们的图只执行单个节点。让我们继续一个简单的调用：

```python  theme={null}
from langchain.messages import HumanMessage

result = graph.invoke({"messages": [HumanMessage("Hi")]})
result
```

```
{'messages': [HumanMessage(content='Hi'), AIMessage(content='Hello!')], 'extra_field': 10}
```

请注意：

* 我们通过更新状态的单个键来启动调用。
* 我们在调用结果中接收到整个状态。

为方便起见，我们经常通过漂亮打印[消息对象](https://python.langchain.com/docs/concepts/messages/)的内容来检查：

```python  theme={null}
for message in result["messages"]:
    message.pretty_print()
```

```
================================ Human Message ================================

Hi
================================== Ai Message ==================================

Hello!
```

### 使用约简器处理状态更新

状态中的每个键都可以有自己的独立[约简器(reducer)](/oss/python/langgraph/graph-api#reducers)函数，该函数控制如何处理来自节点的更新。如果没有明确指定约简器函数，则假定对该键的所有更新都将覆盖它。

对于`TypedDict`状态模式，我们可以通过用约简器函数注释状态的相应字段来定义约简器。

在前面的示例中，我们的节点通过向消息列表附加消息来更新状态中的`"messages"`键。下面，我们为该键添加一个约简器，以便更新自动附加：

```python  theme={null}
from typing_extensions import Annotated

def add(left, right):
    """也可以从内置的`operator`模块导入`add`。"""
    return left + right

class State(TypedDict):
    messages: Annotated[list[AnyMessage], add]  # [!code highlight]
    extra_field: int
```

现在我们的节点可以简化为：

```python  theme={null}
def node(state: State):
    new_message = AIMessage("Hello!")
    return {"messages": [new_message], "extra_field": 10}  # [!code highlight]
```

```python  theme={null}
from langgraph.graph import START

graph = StateGraph(State).add_node(node).add_edge(START, "node").compile()

result = graph.invoke({"messages": [HumanMessage("Hi")]})

for message in result["messages"]:
    message.pretty_print()
```

```
================================ Human Message ================================

Hi
================================== Ai Message ==================================

Hello!
```

#### MessagesState

实际上，更新消息列表还有其他考虑因素：

* 我们可能希望更新状态中的现有消息。
* 我们可能想接受[消息格式](/oss/python/langgraph/graph-api#using-messages-in-your-graph)的简写，例如[OpenAI格式](https://python.langchain.com/docs/concepts/messages/#openai-format)。

LangGraph包含一个内置约简器[`add_messages`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.message.add_messages)，可处理这些考虑因素：

```python  theme={null}
from langgraph.graph.message import add_messages

class State(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]  # [!code highlight]
    extra_field: int

def node(state: State):
    new_message = AIMessage("Hello!")
    return {"messages": [new_message], "extra_field": 10}

graph = StateGraph(State).add_node(node).set_entry_point("node").compile()
```

```python  theme={null}
input_message = {"role": "user", "content": "Hi"}  # [!code highlight]

result = graph.invoke({"messages": [input_message]})

for message in result["messages"]:
    message.pretty_print()
```

```
================================ Human Message ================================

Hi
================================== Ai Message ==================================

Hello!
```

这是涉及[聊天模型](https://python.langchain.com/docs/concepts/chat_models/)的应用程序状态的通用表示形式。为方便起见，LangGraph包含一个预构建的`MessagesState`，这样我们可以：

```python  theme={null}
from langgraph.graph import MessagesState

class State(MessagesState):
    extra_field: int
```

### 定义输入和输出模式

默认情况下，`StateGraph`使用单个模式操作，所有节点都期望使用该模式进行通信。但是，也可以为图定义不同的输入和输出模式。

当指定不同的模式时，仍将使用内部模式在节点之间进行通信。输入模式确保提供的输入与预期结构匹配，而输出模式根据定义的输出模式筛选内部数据以仅返回相关信息。

下面，我们将看到如何定义不同的输入和输出模式。

```python  theme={null}
from langgraph.graph import StateGraph, START, END
from typing_extensions import TypedDict

# 定义输入的模式
class InputState(TypedDict):
    question: str

# 定义输出的模式
class OutputState(TypedDict):
    answer: str

# 定义整体模式，结合输入和输出
class OverallState(InputState, OutputState):
    pass

# 定义处理输入并生成答案的节点
def answer_node(state: InputState):
    # 示例答案和一个额外键
    return {"answer": "bye", "question": state["question"]}

# 构建具有输入和输出模式指定的图
builder = StateGraph(OverallState, input_schema=InputState, output_schema=OutputState)
builder.add_node(answer_node)  # 添加答案节点
builder.add_edge(START, "answer_node")  # 定义起始边
builder.add_edge("answer_node", END)  # 定义结束边
graph = builder.compile()  # 编译图

# 使用输入调用图并打印结果
print(graph.invoke({"question": "hi"}))
```

```
{'answer': 'bye'}
```

注意，调用的输出仅包含输出模式。

### 在节点之间传递私有状态

在某些情况下，您可能希望节点交换对中间逻辑至关重要但不需要成为图主模式一部分的信息。此私有数据与图的总体输入/输出无关，应该仅在特定节点之间共享。

下面，我们将创建一个包含三个节点（node_1、node_2和node_3）的顺序示例图，其中私有数据在前两个步骤（node_1和node_2）之间传递，而第三个步骤（node_3）只能访问公共整体状态。

```python  theme={null}
from langgraph.graph import StateGraph, START, END
from typing_extensions import TypedDict

# 图的整体状态（这是跨节点共享的公共状态）
class OverallState(TypedDict):
    a: str

# node_1的输出包含不属于整体状态的私有数据
class Node1Output(TypedDict):
    private_data: str

# 私有数据仅在node_1和node_2之间共享
def node_1(state: OverallState) -> Node1Output:
    output = {"private_data": "set by node_1"}
    print(f"进入节点`node_1`:\n\t输入: {state}.\n\t返回: {output}")
    return output

# Node 2输入仅请求node_1之后可用的私有数据
class Node2Input(TypedDict):
    private_data: str

def node_2(state: Node2Input) -> OverallState:
    output = {"a": "set by node_2"}
    print(f"进入节点`node_2`:\n\t输入: {state}.\n\t返回: {output}")
    return output

# Node 3只能访问整体状态（无法访问来自node_1的私有数据）
def node_3(state: OverallState) -> OverallState:
    output = {"a": "set by node_3"}
    print(f"进入节点`node_3`:\n\t输入: {state}.\n\t返回: {output}")
    return output

# 按顺序连接节点
# node_2接受来自node_1的私有数据，而
# node_3看不到私有数据。
builder = StateGraph(OverallState).add_sequence([node_1, node_2, node_3])
builder.add_edge(START, "node_1")
graph = builder.compile()

# 使用初始状态调用图
response = graph.invoke(
    {
        "a": "set at start",
    }
)

print()
print(f"图调用的输出: {response}")
```

```
进入节点`node_1`:
    输入: {'a': 'set at start'}.
    返回: {'private_data': 'set by node_1'}
进入节点`node_2`:
    输入: {'private_data': 'set by node_1'}.
    返回: {'a': 'set by node_2'}
进入节点`node_3`:
    输入: {'a': 'set by node_2'}.
    返回: {'a': 'set by node_3'}

图调用的输出: {'a': 'set by node_3'}
```

### 使用Pydantic模型作为图状态

[StateGraph](https://langchain-ai.github.io/langgraph/reference/graphs.md#langgraph.graph.StateGraph)接受一个[`state_schema`](https://reference.langchain.com/python/langchain/middleware/#langchain.agents.middleware.AgentMiddleware.state_schema)参数，该参数指定图中节点可以访问和更新的状态的"形状"。

在我们的示例中，我们通常使用Python原生的`TypedDict`或[`dataclass`](https://docs.python.org/3/library/dataclasses.html)作为`state_schema`，但[`state_schema`](https://reference.langchain.com/python/langchain/middleware/#langchain.agents.middleware.AgentMiddleware.state_schema)可以是任何[类型](https://docs.python.org/3/library/stdtypes.html#type-objects)。

这里，我们将看到如何使用[Pydantic BaseModel](https://docs.pydantic.dev/latest/api/base_model/)作为[`state_schema`](https://reference.langchain.com/python/langchain/middleware/#langchain.agents.middleware.AgentMiddleware.state_schema)，以在**输入**上添加运行时验证。

<Note>
  **已知限制**

  * 目前，图的输出**不会**是pydantic模型的实例。
  * 运行时验证仅发生在节点的输入上，而不是输出上。
  * Pydantic的验证错误跟踪不会显示错误 arising in 的节点。
  * Pydantic的递归验证可能很慢。对于性能敏感的应用程序，您可能需要考虑使用`dataclass`。
</Note>

```python  theme={null}
from langgraph.graph import StateGraph, START, END
from typing_extensions import TypedDict
from pydantic import BaseModel

# 图的整体状态（这是跨节点共享的公共状态）
class OverallState(BaseModel):
    a: str

def node(state: OverallState):
    return {"a": "goodbye"}

# 构建状态图
builder = StateGraph(OverallState)
builder.add_node(node)  # node_1是第一个节点
builder.add_edge(START, "node")  # 从node_1启动图
builder.add_edge("node", END)  # 在node_1之后结束图
graph = builder.compile()

# 使用有效输入测试图
graph.invoke({"a": "hello"})
```

使用**无效**输入调用图

```python  theme={null}
try:
    graph.invoke({"a": 123})  # 应该是字符串
except Exception as e:
    print("因为`a`是整数而不是字符串而引发了异常。")
    print(e)
```

```
因为`a`是整数而不是字符串而引发了异常。
1 validation error for OverallState
a
  Input should be a valid string [type=string_type, input_value=123, input_type=int]
    For further information visit https://errors.pydantic.dev/2.9/v/string_type
```

有关Pydantic模型状态的附加功能，请参见下文：

<Accordion title="序列化行为">
  当使用Pydantic模型作为状态模式时，了解序列化的工作原理非常重要，特别是在以下情况下：

  * 将Pydantic对象作为输入传递
  * 接收来自图的输出
  * 处理嵌套的Pydantic模型

  让我们看看这些行为的实际应用。

  ```python  theme={null}
  from langgraph.graph import StateGraph, START, END
  from pydantic import BaseModel

  class NestedModel(BaseModel):
      value: str

  class ComplexState(BaseModel):
      text: str
      count: int
      nested: NestedModel

  def process_node(state: ComplexState):
      # 节点接收经过验证的Pydantic对象
      print(f"输入状态类型: {type(state)}")
      print(f"嵌套类型: {type(state.nested)}")
      # 返回字典更新
      return {"text": state.text + " processed", "count": state.count + 1}

  # 构建图
  builder = StateGraph(ComplexState)
  builder.add_node("process", process_node)
  builder.add_edge(START, "process")
  builder.add_edge("process", END)
  graph = builder.compile()

  # 为输入创建Pydantic实例
  input_state = ComplexState(text="hello", count=0, nested=NestedModel(value="test"))
  print(f"输入对象类型: {type(input_state)}")

  # 使用Pydantic实例调用图
  result = graph.invoke(input_state)
  print(f"输出类型: {type(result)}")
  print(f"输出内容: {result}")

  # 如果需要，转换回Pydantic模型
  output_model = ComplexState(**result)
  print(f"转换回Pydantic: {type(output_model)}")
  ```
</Accordion>

<Accordion title="运行时类型强制转换">
  Pydantic对某些数据类型执行运行时类型强制转换。这可能很有用，但如果您不知道它，也可能导致意外行为。

  ```python  theme={null}
  from langgraph.graph import StateGraph, START, END
  from pydantic import BaseModel

  class CoercionExample(BaseModel):
      # Pydantic会将字符串数字强制转换为整数
      number: int
      # Pydantic会将字符串布尔值解析为布尔值
      flag: bool

  def inspect_node(state: CoercionExample):
      print(f"number: {state.number} (类型: {type(state.number)})")
      print(f"flag: {state.flag} (类型: {type(state.flag)})")
      return {}

  builder = StateGraph(CoercionExample)
  builder.add_node("inspect", inspect_node)
  builder.add_edge(START, "inspect")
  builder.add_edge("inspect", END)
  graph = builder.compile()

  # 演示使用将被转换的字符串输入的强制转换
  result = graph.invoke({"number": "42", "flag": "true"})

  # 这将因验证错误而失败
  try:
      graph.invoke({"number": "not-a-number", "flag": "true"})
  except Exception as e:
      print(f"\n预期的验证错误: {e}")
  ```
</Accordion>

<Accordion title="使用消息模型">
  在状态模式中使用LangChain消息类型时，序列化有重要注意事项。您应该使用`AnyMessage`（而不是`BaseMessage`）来实现消息对象的正确序列化/反序列化，尤其是在通过网络传输消息对象时。

  ```python  theme={null}
  from langgraph.graph import StateGraph, START, END
  from pydantic import BaseModel
  from langchain.messages import HumanMessage, AIMessage, AnyMessage
  from typing import List

  class ChatState(BaseModel):
      messages: List[AnyMessage]
      context: str

  def add_message(state: ChatState):
      return {"messages": state.messages + [AIMessage(content="Hello there!")]}

  builder = StateGraph(ChatState)
  builder.add_node("add_message", add_message)
  builder.add_edge(START, "add_message")
  builder.add_edge("add_message", END)
  graph = builder.compile()

  # 创建带消息的输入
  initial_state = ChatState(
      messages=[HumanMessage(content="Hi")], context="Customer support chat"
  )

  result = graph.invoke(initial_state)
  print(f"输出: {result}")

  # 转换回Pydantic模型以查看消息类型
  output_model = ChatState(**result)
  for i, msg in enumerate(output_model.messages):
      print(f"消息 {i}: {type(msg).__name__} - {msg.content}")
  ```
</Accordion>

## 添加运行时配置

有时您希望能够配置调用图时的图。例如，您可能希望能够指定在运行时使用什么LLM或系统提示，*而不用这些参数污染图状态*。

要添加运行时配置：

1. 为您的配置指定一个模式
2. 将配置添加到节点或条件边的函数签名中
3. 将配置传递给图。

下面是一个简单的示例：

```python  theme={null}
from langgraph.graph import END, StateGraph, START
from langgraph.runtime import Runtime
from typing_extensions import TypedDict

# 1. 指定配置模式
class ContextSchema(TypedDict):
    my_runtime_value: str

# 2. 定义一个在节点中访问配置的图
class State(TypedDict):
    my_state_value: str

def node(state: State, runtime: Runtime[ContextSchema]):  # [!code highlight]
    if runtime.context["my_runtime_value"] == "a":  # [!code highlight]
        return {"my_state_value": 1}
    elif runtime.context["my_runtime_value"] == "b":  # [!code highlight]
        return {"my_state_value": 2}
    else:
        raise ValueError("未知值。")

builder = StateGraph(State, context_schema=ContextSchema)  # [!code highlight]
builder.add_node(node)
builder.add_edge(START, "node")
builder.add_edge("node", END)

graph = builder.compile()

# 3. 在运行时传入配置：
print(graph.invoke({}, context={"my_runtime_value": "a"}))  # [!code highlight]
print(graph.invoke({}, context={"my_runtime_value": "b"}))  # [!code highlight]
```

```
{'my_state_value': 1}
{'my_state_value': 2}
```

<Accordion title="扩展示例：在运行时指定LLM">
  下面我们演示一个实际示例，我们在其中配置在运行时使用什么LLM。我们将同时使用OpenAI和Anthropic模型。

  ```python  theme={null}
  from dataclasses import dataclass

  from langchain.chat_models import init_chat_model
  from langgraph.graph import MessagesState, END, StateGraph, START
  from langgraph.runtime import Runtime
  from typing_extensions import TypedDict

  @dataclass
  class ContextSchema:
      model_provider: str = "anthropic"

  MODELS = {
      "anthropic": init_chat_model("anthropic:claude-3-5-haiku-latest"),
      "openai": init_chat_model("openai:gpt-4.1-mini"),
  }

  def call_model(state: MessagesState, runtime: Runtime[ContextSchema]):
      model = MODELS[runtime.context.model_provider]
      response = model.invoke(state["messages"])
      return {"messages": [response]}

  builder = StateGraph(MessagesState, context_schema=ContextSchema)
  builder.add_node("model", call_model)
  builder.add_edge(START, "model")
  builder.add_edge("model", END)

  graph = builder.compile()

  # 使用
  input_message = {"role": "user", "content": "hi"}
  # 没有配置时，使用默认值（Anthropic）
  response_1 = graph.invoke({"messages": [input_message]}, context=ContextSchema())["messages"][-1]
  # 或者，可以设置OpenAI
  response_2 = graph.invoke({"messages": [input_message]}, context={"model_provider": "openai"})["messages"][-1]

  print(response_1.response_metadata["model_name"])
  print(response_2.response_metadata["model_name"])
  ```

  ```
  claude-3-5-haiku-20241022
  gpt-4.1-mini-2025-04-14
  ```
</Accordion>

<Accordion title="扩展示例：在运行时指定模型和系统消息">
  下面我们演示一个实际示例，我们在其中配置两个参数：在运行时使用的LLM和系统消息。

  ```python  theme={null}
  from dataclasses import dataclass
  from langchain.chat_models import init_chat_model
  from langchain.messages import SystemMessage
  from langgraph.graph import END, MessagesState, StateGraph, START
  from langgraph.runtime import Runtime
  from typing_extensions import TypedDict

  @dataclass
  class ContextSchema:
      model_provider: str = "anthropic"
      system_message: str | None = None

  MODELS = {
      "anthropic": init_chat_model("anthropic:claude-3-5-haiku-latest"),
      "openai": init_chat_model("openai:gpt-4.1-mini"),
  }

  def call_model(state: MessagesState, runtime: Runtime[ContextSchema]):
      model = MODELS[runtime.context.model_provider]
      messages = state["messages"]
      if (system_message := runtime.context.system_message):
          messages = [SystemMessage(system_message)] + messages
      response = model.invoke(messages)
      return {"messages": [response]}

  builder = StateGraph(MessagesState, context_schema=ContextSchema)
  builder.add_node("model", call_model)
  builder.add_edge(START, "model")
  builder.add_edge("model", END)

  graph = builder.compile()

  # 使用
  input_message = {"role": "user", "content": "hi"}
  response = graph.invoke({"messages": [input_message]}, context={"model_provider": "openai", "system_message": "Respond in Italian."})
  for message in response["messages"]:
      message.pretty_print()
  ```

  ```
  ================================ Human Message ================================

  hi
  ================================== Ai Message ==================================

  Ciao! Come posso aiutarti oggi?
  ```
</Accordion>

## 添加重试策略

有许多用例可能需要您的节点具有自定义重试策略，例如如果您正在调用API、查询数据库或调用LLM等。LangGraph允许您向节点添加重试策略。

要配置重试策略，请将`retry_policy`参数传递给[`add_node`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.StateGraph.add_node)。`retry_policy`参数接受一个`RetryPolicy`命名元组对象。下面我们实例化一个具有默认参数的`RetryPolicy`对象，并将其与一个节点关联：

```python  theme={null}
from langgraph.types import RetryPolicy

builder.add_node(
    "node_name",
    node_function,
    retry_policy=RetryPolicy(),
)
```

默认情况下，`retry_on`参数使用`default_retry_on`函数，该函数会重试任何异常，以下情况除外：

* `ValueError`
* `TypeError`
* `ArithmeticError`
* `ImportError`
* `LookupError`
* `NameError`
* `SyntaxError`
* `RuntimeError`
* `ReferenceError`
* `StopIteration`
* `StopAsyncIteration`
* `OSError`

此外，对于来自流行http请求库（如`requests`和`httpx`）的异常，它仅重试5xx状态码。

<Accordion title="扩展示例：自定义重试策略">
  考虑一个我们从SQL数据库读取的示例。下面我们将两个不同的重试策略传递给节点：

  ```python  theme={null}
  import sqlite3
  from typing_extensions import TypedDict
  from langchain.chat_models import init_chat_model
  from langgraph.graph import END, MessagesState, StateGraph, START
  from langgraph.types import RetryPolicy
  from langchain_community.utilities import SQLDatabase
  from langchain.messages import AIMessage

  db = SQLDatabase.from_uri("sqlite:///:memory:")
  model = init_chat_model("anthropic:claude-3-5-haiku-latest")

  def query_database(state: MessagesState):
      query_result = db.run("SELECT * FROM Artist LIMIT 10;")
      return {"messages": [AIMessage(content=query_result)]}

  def call_model(state: MessagesState):
      response = model.invoke(state["messages"])
      return {"messages": [response]}

  # 定义一个新图
  builder = StateGraph(MessagesState)
  builder.add_node(
      "query_database",
      query_database,
      retry_policy=RetryPolicy(retry_on=sqlite3.OperationalError),
  )
  builder.add_node("model", call_model, retry_policy=RetryPolicy(max_attempts=5))
  builder.add_edge(START, "model")
  builder.add_edge("model", "query_database")
  builder.add_edge("query_database", END)
  graph = builder.compile()
  ```
</Accordion>

## 添加节点缓存

节点缓存在您想要避免重复操作时很有用，例如在进行昂贵操作（无论是时间还是成本方面）时。LangGraph允许您向图中的节点添加个性化缓存策略。

要配置缓存策略，请将`cache_policy`参数传递给[`add_node`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.StateGraph.add_node)函数。在以下示例中，实例化了一个具有120秒生存时间和默认`key_func`生成器的[`CachePolicy`](https://reference.langchain.com/python/langgraph/types/#langgraph.types.CachePolicy)对象。然后将其与一个节点关联：

```python  theme={null}
from langgraph.types import CachePolicy

builder.add_node(
    "node_name",
    node_function,
    cache_policy=CachePolicy(ttl=120),
)
```

然后，要为图启用节点级缓存，请在编译图时设置`cache`参数。下面的示例使用`InMemoryCache`设置具有内存缓存的图，但也可使用`SqliteCache`。

```python  theme={null}
from langgraph.cache.memory import InMemoryCache

graph = builder.compile(cache=InMemoryCache())
```

## 创建步骤序列

<Info>
  **先决条件**
  本指南假设您熟悉上述关于[状态](#define-and-update-state)的部分。
</Info>

这里我们演示如何构建简单的步骤序列。我们将展示：

1. 如何构建顺序图
2. 用于构建类似图的内置简写。

要添加节点序列，我们使用[图](/oss/python/langgraph/graph-api#stategraph)的[`add_node`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.StateGraph.add_node)和[`add_edge`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.StateGraph.add_edge)方法：

```python  theme={null}
from langgraph.graph import START, StateGraph

builder = StateGraph(State)

# 添加节点
builder.add_node(step_1)
builder.add_node(step_2)
builder.add_node(step_3)

# 添加边
builder.add_edge(START, "step_1")
builder.add_edge("step_1", "step_2")
builder.add_edge("step_2", "step_3")
```

我们也可以使用内置简写`.add_sequence`：

```python  theme={null}
builder = StateGraph(State).add_sequence([step_1, step_2, step_3])
builder.add_edge(START, "step_1")
```

<Accordion title="为什么使用LangGraph将应用程序步骤拆分为序列？">
  LangGraph使得为您的应用程序添加底层持久层变得容易。
  这允许在节点执行之间检查点状态，因此您的LangGraph节点控制：

  * 状态更新如何[检查点](/oss/python/langgraph/persistence)
  * 如何在[人在环路中](/oss/python/langgraph/interrupts)工作流中恢复中断
  * 如何使用LangGraph的[时间旅行](/oss/python/langgraph/use-time-travel)功能"倒带"和分支执行

  它们还确定执行步骤如何[流式传输](/oss/python/langgraph/streaming)，以及如何使用[Studio](/langsmith/studio)可视化调试您的应用程序。

  让我们演示一个端到端的示例。我们将创建一个包含三个步骤的序列：

  1. 在状态键中填充一个值
  2. 更新相同的值
  3. 填充一个不同的值

  首先，定义我们的[状态](/oss/python/langgraph/graph-api#state)。这控制图的[模式](/oss/python/langgraph/graph-api#schema)，也可以指定如何应用更新。有关更多详细信息，请参阅[本节](#process-state-updates-with-reducers)。

  在我们的例子中，我们将只跟踪两个值：

  ```python  theme={null}
  from typing_extensions import TypedDict

  class State(TypedDict):
      value_1: str
      value_2: int
  ```

  我们的[节点](/oss/python/langgraph/graph-api#nodes)只是Python函数，它们读取图的状态并对其进行更新。该函数的第一个参数始终是状态：

  ```python  theme={null}
  def step_1(state: State):
      return {"value_1": "a"}

  def step_2(state: State):
      current_value_1 = state["value_1"]
      return {"value_1": f"{current_value_1} b"}

  def step_3(state: State):
      return {"value_2": 10}
  ```

  <Note>
    注意，当发出对状态的更新时，每个节点可以只指定它希望更新的键的值。

    默认情况下，这将**覆盖**相应键的值。您还可以使用[约简器](/oss/python/langgraph/graph-api#reducers)控制如何处理更新—例如，您可以连续附加对键的更新，而不是覆盖。有关更多详细信息，请参阅[本节](#process-state-updates-with-reducers)。
  </Note>

  最后，我们定义图。我们使用[StateGraph](/oss/python/langgraph/graph-api#stategraph)定义一个操作此状态的图。

  然后使用[`add_node`](/oss/python/langgraph/graph-api#messagesstate)和[`add_edge`](/oss/python/langgraph/graph-api#edges)填充我们的图并定义其控制流。

  ```python  theme={null}
  from langgraph.graph import START, StateGraph

  builder = StateGraph(State)

  # 添加节点
  builder.add_node(step_1)
  builder.add_node(step_2)
  builder.add_node(step_3)

  # 添加边
  builder.add_edge(START, "step_1")
  builder.add_edge("step_1", "step_2")
  builder.add_edge("step_2", "step_3")
  ```

  <Tip>
    **指定自定义名称**
    您可以使用[`add_node`](https://reference.langchain.com/python/langchain/core/runnables/langchain_core.runnables.base.RunnableConfig.html)为节点指定自定义名称：

    ```python  theme={null}
    builder.add_node("my_node", step_1)
    ```
  </Tip>

  注意：

  * [`add_edge`](https://reference.langchain.com/python/langchain/core/runnables/langchain_core.runnables.base.RunnableConfig.html)采用节点的名称，对于函数，默认为`node.__name__`。
  * 我们必须指定图的入口点。为此，我们添加一条与[START节点](/oss/python/langgraph/graph-api#start-node)相连的边。
  * 当没有更多节点要执行时，图停止。

  我们接下来[编译](/oss/python/langgraph/graph-api#compiling-your-graph)我们的图。这会对图的结构提供一些基本检查（例如，识别孤立节点）。如果我们通过[检查点器](/oss/python/langgraph/persistence)向应用程序添加持久性，它也会在这里传递。

  ```python  theme={null}
  graph = builder.compile()
  ```

  LangGraph提供内置工具来可视化您的图。让我们检查我们的序列。有关可视化的详细信息，请参阅[本指南](#visualize-your-graph)。

  ```python  theme={null}
  from IPython.display import Image, display

  display(Image(graph.get_graph().draw_mermaid_png()))
  ```

    <img src="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_2.png?fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=fa0376786cc89d704a5435abba178804" alt="步骤序列图" data-og-width="107" width="107" data-og-height="333" height="333" data-path="oss/images/graph_api_image_2.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_2.png?w=280&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=e2d4ec28fa1b03fab44cbcfccd19aa16 280w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_2.png?w=560&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=5ab128ae8f12f766384f48e03fa2c35c 560w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_2.png?w=840&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=db4260bece32ab8f5045ea7b9b151c45 840w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_2.png?w=1100&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=8a93a6970742a83f06fb1a5288668eef 1100w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_2.png?w=1650&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=269956fccda17f64def8a69db847d4aa 1650w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_2.png?w=2500&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=40f495cb5fbca4aa2c960083a50af52e 2500w" />

  让我们继续一个简单的调用：

  ```python  theme={null}
  graph.invoke({"value_1": "c"})
  ```

  ```
  {'value_1': 'a b', 'value_2': 10}
  ```

  注意：

  * 我们通过为状态的单个键提供值来启动调用。我们总是必须为至少一个键提供值。
  * 我们传入的值被第一个节点覆盖。
  * 第二个节点更新了该值。
  * 第三个节点填充了一个不同的值。

  <Tip>
    **内置简写**
    `langgraph>=0.2.46`包含一个内置简写`add_sequence`，用于添加节点序列。您可以按如下方式编译相同的图：

    ```python  theme={null}
    builder = StateGraph(State).add_sequence([step_1, step_2, step_3])  # [!code highlight]
    builder.add_edge(START, "step_1")

    graph = builder.compile()

    graph.invoke({"value_1": "c"})
    ```
  </Tip>
</Accordion>

## 创建分支

并行执行节点对于加速整体图操作至关重要。LangGraph提供对节点并行执行的原生支持，这可以显著增强基于图的性能。这种并行化是通过扇出(fan-out)和扇入(fan-in)机制实现的，同时使用标准边和[条件边(conditional_edges)](https://langchain-ai.github.io/langgraph/reference/graphs.md#langgraph.graph.MessageGraph.add_conditional_edges)。下面是一些示例，展示如何创建为您工作的分支数据流。

### 并行运行图节点

在这个示例中，我们从`Node A`扇出到`B和C`，然后扇入到`D`。在我们的状态中，[我们指定了add约简器](/oss/python/langgraph/graph-api#reducers)。这将组合或累积状态中特定键的值，而不是简单地覆盖现有值。对于列表，这意味着将新列表与现有列表连接起来。有关使用约简器更新状态的更多详细信息，请参阅上述关于[状态约简器](#process-state-updates-with-reducers)的部分。

```python  theme={null}
import operator
from typing import Annotated, Any
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END

class State(TypedDict):
    # operator.add 约简器函数使此键成为仅附加
    aggregate: Annotated[list, operator.add]

def a(state: State):
    print(f'向 {state["aggregate"]} 添加"A"')
    return {"aggregate": ["A"]}

def b(state: State):
    print(f'向 {state["aggregate"]} 添加"B"')
    return {"aggregate": ["B"]}

def c(state: State):
    print(f'向 {state["aggregate"]} 添加"C"')
    return {"aggregate": ["C"]}

def d(state: State):
    print(f'向 {state["aggregate"]} 添加"D"')
    return {"aggregate": ["D"]}

builder = StateGraph(State)
builder.add_node(a)
builder.add_node(b)
builder.add_node(c)
builder.add_node(d)
builder.add_edge(START, "a")
builder.add_edge("a", "b")
builder.add_edge("a", "c")
builder.add_edge("b", "d")
builder.add_edge("c", "d")
builder.add_edge("d", END)
graph = builder.compile()
```

```python  theme={null}
from IPython.display import Image, display

display(Image(graph.get_graph().draw_mermaid_png()))
```

<img src="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_3.png?fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=8359f2e8d9dde03d7cc25f9d755a428d" alt="并行执行图" data-og-width="143" width="143" data-og-height="432" height="432" data-path="oss/images/graph_api_image_3.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_3.png?w=280&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=75695e23f3e5e7eddb985785376108c4 280w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_3.png?w=560&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=cf45dc47fcfcf30ef39922a44119d815 560w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_3.png?w=840&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=92b3e0a7d06b07becf4deab660ff3717 840w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_3.png?w=1100&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=8c0e296783bde688d32b36e7e8fb669c 1100w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_3.png?w=1650&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=a4ff2db4eea2ab57343b329f6e21949c 1650w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_3.png?w=2500&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=99b0250accefffa610c67662ca4be2a2 2500w" />

使用约简器，您可以看到每个节点中添加的值是如何累积的。

```python  theme={null}
graph.invoke({"aggregate": []}, {"configurable": {"thread_id": "foo"}})
```

```
向 [] 添加"A"
向 ['A'] 添加"B"
向 ['A'] 添加"C"
向 ['A', 'B', 'C'] 添加"D"
```

<Note>
  在上面的示例中，节点`"b"`和`"c"`在同一个[超级步](/oss/python/langgraph/graph-api#graphs)中并行执行。因为它们在同一步骤中，所以节点`"d"`在`"b"`和`"c"`都完成后才执行。

  重要的是，来自并行超级步的更新可能不会以一致的顺序应用。如果您需要来自并行超级步的更新具有一致、预定的顺序，您应该将输出与用于对它们排序的值一起写入状态中的单独字段。
</Note>

<Accordion title="异常处理？">
  LangGraph在[超级步](/oss/python/langgraph/graph-api#graphs)内执行节点，这意味着虽然并行分支并行执行，但整个超级步是**事务性的**。如果这些分支中的任何一个引发异常，**没有**更新会被应用到状态（整个超级步出错）。

  重要的是，当使用[检查点器](/oss/python/langgraph/persistence)时，成功节点内的超级步结果会被保存，并且在恢复时不会重复执行。

  如果您有容易出错的节点（可能是处理不可靠的API调用），LangGraph提供了两种解决方法：

  1. 您可以在节点内编写常规Python代码来捕获和处理异常。
  2. 您可以设置**[重试策略](https://langchain-ai.github.io/langgraph/reference/types/#langgraph.types.RetryPolicy)**，指示图重试引发某些类型异常的节点。只有失败的分支才会重试，因此您不必担心执行冗余工作。

  这些方法结合使用可以让您执行并行执行并完全控制异常处理。
</Accordion>

<Tip>
  **设置最大并发数**
  您可以通过在调用图时设置`max_concurrency`来控制并发任务的最大数量。

  ```python  theme={null}
  graph.invoke({"value_1": "c"}, {"configurable": {"max_concurrency": 10}})
  ```
</Tip>

### 延迟节点执行

当您希望延迟节点的执行直到所有其他待处理任务完成时，延迟节点执行很有用。这在分支长度不同的情况下特别相关，这在map-reduce流等工作流中很常见。

上面的示例展示了当每个路径只有一个步骤时如何扇出和扇入。但如果一个分支有多个步骤怎么办？让我们在`"b"`分支中添加一个节点`"b_2"`：

```python  theme={null}
import operator
from typing import Annotated, Any
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END

class State(TypedDict):
    # operator.add 约简器函数使此键成为仅附加
    aggregate: Annotated[list, operator.add]

def a(state: State):
    print(f'向 {state["aggregate"]} 添加"A"')
    return {"aggregate": ["A"]}

def b(state: State):
    print(f'向 {state["aggregate"]} 添加"B"')
    return {"aggregate": ["B"]}

def b_2(state: State):
    print(f'向 {state["aggregate"]} 添加"B_2"')
    return {"aggregate": ["B_2"]}

def c(state: State):
    print(f'向 {state["aggregate"]} 添加"C"')
    return {"aggregate": ["C"]}

def d(state: State):
    print(f'向 {state["aggregate"]} 添加"D"')
    return {"aggregate": ["D"]}

builder = StateGraph(State)
builder.add_node(a)
builder.add_node(b)
builder.add_node(b_2)
builder.add_node(c)
builder.add_node(d, defer=True)  # [!code highlight]
builder.add_edge(START, "a")
builder.add_edge("a", "b")
builder.add_edge("a", "c")
builder.add_edge("b", "b_2")
builder.add_edge("b_2", "d")
builder.add_edge("c", "d")
builder.add_edge("d", END)
graph = builder.compile()
```

```python  theme={null}
from IPython.display import Image, display

display(Image(graph.get_graph().draw_mermaid_png()))
```

<img src="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_4.png?fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=44cd97f020dfefeaffbe2b012514f343" alt="延迟执行图" data-og-width="161" width="161" data-og-height="531" height="531" data-path="oss/images/graph_api_image_4.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_4.png?w=280&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=645690182cd1ed41151da17c7d103d47 280w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_4.png?w=560&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=51cdd5ba95c2285baa2b7dc5236c8b63 560w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_4.png?w=840&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=e99de6c886526afdb2e7a538e3d23705 840w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_4.png?w=1100&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=92aba13b5bbc8428e42f2ad50ba7b607 1100w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_4.png?w=1650&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=14fda3686ef277c3f72a3ed8618c5e58 1650w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_4.png?w=2500&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=65c543b4b79c53b9224c74631b959e0b 2500w" />

```python  theme={null}
graph.invoke({"aggregate": []})
```

```
向 [] 添加"A"
向 ['A'] 添加"B"
向 ['A'] 添加"C"
向 ['A', 'B', 'C'] 添加"B_2"
向 ['A', 'B', 'C', 'B_2'] 添加"D"
```

在上面的示例中，节点`"b"`和`"c"`在同一个超级步中并行执行。我们在节点`d`上设置了`defer=True`，因此它不会执行直到所有待处理任务完成。在这种情况下，这意味着`"d"`等待直到整个`"b"`分支完成。

### 条件分支

如果您的扇出应该在运行时基于状态而变化，您可以使用[`add_conditional_edges`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.StateGraph.add_conditional_edges)使用图状态选择一个或多个路径。请参见下面的示例，其中节点`a`生成确定后续节点的状态更新。

```python  theme={null}
import operator
from typing import Annotated, Literal, Sequence
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END

class State(TypedDict):
    aggregate: Annotated[list, operator.add]
    # 向状态添加一个键。我们将设置此键以确定
    # 我们如何分支。
    which: str

def a(state: State):
    print(f'向 {state["aggregate"]} 添加"A"')
    return {"aggregate": ["A"], "which": "c"}  # [!code highlight]

def b(state: State):
    print(f'向 {state["aggregate"]} 添加"B"')
    return {"aggregate": ["B"]}

def c(state: State):
    print(f'向 {state["aggregate"]} 添加"C"')
    return {"aggregate": ["C"]}

builder = StateGraph(State)
builder.add_node(a)
builder.add_node(b)
builder.add_node(c)
builder.add_edge(START, "a")
builder.add_edge("b", END)
builder.add_edge("c", END)

def conditional_edge(state: State) -> Literal["b", "c"]:
    # 在这里填写任意逻辑，使用状态
    # 确定下一个节点
    return state["which"]

builder.add_conditional_edges("a", conditional_edge)  # [!code highlight]

graph = builder.compile()
```

```python  theme={null}
from IPython.display import Image, display

display(Image(graph.get_graph().draw_mermaid_png()))
```

<img src="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_5.png?fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=3373a383d5acc3e4d6a4d1575e849146" alt="条件分支图" data-og-width="143" width="143" data-og-height="333" height="333" data-path="oss/images/graph_api_image_5.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_5.png?w=280&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=addc707d8e23e088279d93e61cd4429c 280w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_5.png?w=560&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=9b0779c2c5444a984a67617640449b26 560w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_5.png?w=840&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=77a82cd36bc56637b4c3bdd0bccc656a 840w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_5.png?w=1100&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=fd83ca7056bb93a4a72187b4aeed3873 1100w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_5.png?w=1650&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=5c57aebb9c69aa7bce3f77adcaee11a4 1650w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_5.png?w=2500&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=0e256ff324997275e003ee62809e030d 2500w" />

```python  theme={null}
result = graph.invoke({"aggregate": []})
print(result)
```

```
向 [] 添加"A"
向 ['A'] 添加"C"
{'aggregate': ['A', 'C'], 'which': 'c'}
```

<Tip>
  您的条件边可以路由到多个目标节点。例如：

  ```python  theme={null}
  def route_bc_or_cd(state: State) -> Sequence[str]:
  if state["which"] == "cd":
  return ["c", "d"]
  return ["b", "c"]
  ```
</Tip>

## Map-Reduce和Send API

LangGraph使用Send API支持map-reduce和其他高级分支模式。以下是使用它的示例：

```python  theme={null}
from langgraph.graph import StateGraph, START, END
from langgraph.types import Send
from typing_extensions import TypedDict, Annotated
import operator

class OverallState(TypedDict):
    topic: str
    subjects: list[str]
    jokes: Annotated[list[str], operator.add]
    best_selected_joke: str

def generate_topics(state: OverallState):
    return {"subjects": ["lions", "elephants", "penguins"]}

def generate_joke(state: OverallState):
    joke_map = {
        "lions": "Why don't lions like fast food? Because they can't catch it!",
        "elephants": "Why don't elephants use computers? They're afraid of the mouse!",
        "penguins": "Why don't penguins like talking to strangers at parties? Because they find it hard to break the ice."
    }
    return {"jokes": [joke_map[state["subject"]]]}

def continue_to_jokes(state: OverallState):
    return [Send("generate_joke", {"subject": s}) for s in state["subjects"]]

def best_joke(state: OverallState):
    return {"best_selected_joke": "penguins"}

builder = StateGraph(OverallState)
builder.add_node("generate_topics", generate_topics)
builder.add_node("generate_joke", generate_joke)
builder.add_node("best_joke", best_joke)
builder.add_edge(START, "generate_topics")
builder.add_conditional_edges("generate_topics", continue_to_jokes, ["generate_joke"])
builder.add_edge("generate_joke", "best_joke")
builder.add_edge("best_joke", END)
graph = builder.compile()
```

```python  theme={null}
from IPython.display import Image, display

display(Image(graph.get_graph().draw_mermaid_png()))
```

<img src="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_6.png?fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=48249d2085e8bfc63a142ccfba5082f5" alt="具有扇出的Map-Reduce图" data-og-width="160" width="160" data-og-height="432" height="432" data-path="oss/images/graph_api_image_6.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_6.png?w=280&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=f37fee0612923f1363e110025a9b9727 280w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_6.png?w=560&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=83f39ecd3959718bbe11e2a3eaa6d8ef 560w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_6.png?w=840&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=9edacf5d4a433e39922b4bc003906b9d 840w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_6.png?w=1100&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=3627608cc06068c975bff51e98247889 1100w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_6.png?w=1650&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=70d18d5cb2ed9e706aea7792723d6891 1650w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_6.png?w=2500&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=03f4b27152e455d84d589c0c46c2324d 2500w" />

```python  theme={null}
# 调用图：这里我们调用它来生成笑话列表
for step in graph.stream({"topic": "animals"}):
    print(step)
```

```
{'generate_topics': {'subjects': ['lions', 'elephants', 'penguins']}}
{'generate_joke': {'jokes': ["Why don't lions like fast food? Because they can't catch it!"]}}
{'generate_joke': {'jokes': ["Why don't elephants use computers? They're afraid of the mouse!"]}}
{'generate_joke': {'jokes': ['Why don't penguins like talking to strangers at parties? Because they find it hard to break the ice.']}}
{'best_joke': {'best_selected_joke': 'penguins'}}
```

## 创建和控制循环

当创建带有循环的图时，我们需要一种机制来终止执行。这通常通过添加一个[条件边](/oss/python/langgraph/graph-api#conditional-edges)来完成，该边一旦达到某个终止条件就会路由到[END](/oss/python/langgraph/graph-api#end-node)节点。

您还可以在调用或流式传输图时设置图的递归限制。递归限制设置图被允许执行的最大[超级步](/oss/python/langgraph/graph-api#graphs)数，之后它会引发错误。有关递归限制概念的更多信息，请阅读[此处](/oss/python/langgraph/graph-api#recursion-limit)。

让我们考虑一个带有循环的简单图，以便更好地理解这些机制的工作原理。

<Tip>
  要返回状态的最后一个值而不是接收递归限制错误，请参阅[下一节](#impose-a-recursion-limit)。
</Tip>

创建循环时，您可以包含一个条件边，该边指定终止条件：

```python  theme={null}
builder = StateGraph(State)
builder.add_node(a)
builder.add_node(b)

def route(state: State) -> Literal["b", END]:
    if termination_condition(state):
        return END
    else:
        return "b"

builder.add_edge(START, "a")
builder.add_conditional_edges("a", route)
builder.add_edge("b", "a")
graph = builder.compile()
```

要控制递归限制，请在配置中指定`"recursionLimit"`。这将引发一个`GraphRecursionError`，您可以捕获并处理它：

```python  theme={null}
from langgraph.errors import GraphRecursionError

try:
    graph.invoke(inputs, {"recursion_limit": 3})
except GraphRecursionError:
    print("递归错误")
```

让我们定义一个带有简单循环的图。注意，我们使用条件边来实现终止条件。

```python  theme={null}
import operator
from typing import Annotated, Literal
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END

class State(TypedDict):
    # operator.add 约简器函数使此键成为仅附加
    aggregate: Annotated[list, operator.add]

def a(state: State):
    print(f'节点A看到 {state["aggregate"]}')
    return {"aggregate": ["A"]}

def b(state: State):
    print(f'节点B看到 {state["aggregate"]}')
    return {"aggregate": ["B"]}

# 定义节点
builder = StateGraph(State)
builder.add_node(a)
builder.add_node(b)

# 定义边
def route(state: State) -> Literal["b", END]:
    if len(state["aggregate"]) < 7:
        return "b"
    else:
        return END

builder.add_edge(START, "a")
builder.add_conditional_edges("a", route)
builder.add_edge("b", "a")
graph = builder.compile()
```

```python  theme={null}
from IPython.display import Image, display

display(Image(graph.get_graph().draw_mermaid_png()))
```

<img src="https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/graph_api_image_7.png?fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=e1b99e7efe45b1fdc5836d590d5fbbc3" alt="简单循环图" data-og-width="188" width="188" data-og-height="249" height="249" data-path="oss/images/graph_api_image_7.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/graph_api_image_7.png?w=280&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=a443c1ddc2f6a4e7c73f4482c7d63912 280w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/graph_api_image_7.png?w=560&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=f65d82d8aaeb024beb5da1aa2948bcdb 560w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/graph_api_image_7.png?w=840&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=b95f4df2fb69f28779a1d8dd113409d0 840w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/graph_api_image_7.png?w=1100&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=bdb4011d05756c10a1c7b5dea683fdb7 1100w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/graph_api_image_7.png?w=1650&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=dde791caa4279a6248b59b70df99dd2c 1650w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/graph_api_image_7.png?w=2500&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=e4d568719f1761ff3a3d2ea9175241d8 2500w" />

这种架构类似于[ReAct代理](/oss/python/langgraph/workflows-agents)，其中节点`"a"`是工具调用模型，节点`"b"`表示工具。

在我们的`route`条件边中，我们指定在状态中的`"aggregate"`列表超过阈值长度后应该结束。

调用图时，我们看到我们在节点`"a"`和`"b"`之间交替，直到达到终止条件。

```python  theme={null}
graph.invoke({"aggregate": []})
```

```
节点A看到 []
节点B看到 ['A']
节点A看到 ['A', 'B']
节点B看到 ['A', 'B', 'A']
节点A看到 ['A', 'B', 'A', 'B']
节点B看到 ['A', 'B', 'A', 'B', 'A']
节点A看到 ['A', 'B', 'A', 'B', 'A', 'B']
```

### 强制递归限制

在某些应用程序中，我们可能无法保证会达到给定的终止条件。在这些情况下，我们可以设置图的[递归限制](/oss/python/langgraph/graph-api#recursion-limit)。这将在给定的[超级步](/oss/python/langgraph/graph-api#graphs)数量后引发`GraphRecursionError`。然后我们可以捕获并处理此异常：

```python  theme={null}
from langgraph.errors import GraphRecursionError

try:
    graph.invoke({"aggregate": []}, {"recursion_limit": 4})
except GraphRecursionError:
    print("递归错误")
```

```
节点A看到 []
节点B看到 ['A']
节点C看到 ['A', 'B']
节点D看到 ['A', 'B']
节点A看到 ['A', 'B', 'C', 'D']
递归错误
```

<Accordion title="扩展示例：在达到递归限制时返回状态">
  而不是引发`GraphRecursionError`，我们可以引入一个新键到状态中，该键跟踪达到递归限制前剩余的步骤数。然后我们可以使用此键来确定是否应该结束运行。

  LangGraph实现了特殊的`RemainingSteps`注释。在幕后，它创建了一个`ManagedValue`通道——一个将在我们的图运行期间存在但不再存在的状态通道。

  ```python  theme={null}
  import operator
  from typing import Annotated, Literal
  from typing_extensions import TypedDict
  from langgraph.graph import StateGraph, START, END
  from langgraph.managed.is_last_step import RemainingSteps

  class State(TypedDict):
      aggregate: Annotated[list, operator.add]
      remaining_steps: RemainingSteps

  def a(state: State):
      print(f'节点A看到 {state["aggregate"]}')
      return {"aggregate": ["A"]}

  def b(state: State):
      print(f'节点B看到 {state["aggregate"]}')
      return {"aggregate": ["B"]}

  # 定义节点
  builder = StateGraph(State)
  builder.add_node(a)
  builder.add_node(b)

  # 定义边
  def route(state: State) -> Literal["b", END]:
      if state["remaining_steps"] <= 2:
          return END
      else:
          return "b"

  builder.add_edge(START, "a")
  builder.add_conditional_edges("a", route)
  builder.add_edge("b", "a")
  graph = builder.compile()

  # 测试它
  result = graph.invoke({"aggregate": []}, {"recursion_limit": 4})
  print(result)
  ```

  ```
  节点A看到 []
  节点B看到 ['A']
  节点A看到 ['A', 'B']
  {'aggregate': ['A', 'B', 'A']}
  ```
</Accordion>

<Accordion title="扩展示例：带分支的循环">
  为了更好地理解递归限制的工作原理，让我们考虑一个更复杂的示例。下面我们实现了一个循环，但一个步骤扇出到两个节点：

  ```python  theme={null}
  import operator
  from typing import Annotated, Literal
  from typing_extensions import TypedDict
  from langgraph.graph import StateGraph, START, END

  class State(TypedDict):
      aggregate: Annotated[list, operator.add]

  def a(state: State):
      print(f'节点A看到 {state["aggregate"]}')
      return {"aggregate": ["A"]}

  def b(state: State):
      print(f'节点B看到 {state["aggregate"]}')
      return {"aggregate": ["B"]}

  def c(state: State):
      print(f'节点C看到 {state["aggregate"]}')
      return {"aggregate": ["C"]}

  def d(state: State):
      print(f'节点D看到 {state["aggregate"]}')
      return {"aggregate": ["D"]}

  # 定义节点
  builder = StateGraph(State)
  builder.add_node(a)
  builder.add_node(b)
  builder.add_node(c)
  builder.add_node(d)

  # 定义边
  def route(state: State) -> Literal["b", END]:
      if len(state["aggregate"]) < 7:
          return "b"
      else:
          return END

  builder.add_edge(START, "a")
  builder.add_conditional_edges("a", route)
  builder.add_edge("b", "c")
  builder.add_edge("b", "d")
  builder.add_edge(["c", "d"], "a")
  graph = builder.compile()
  ```

  ```python  theme={null}
  from IPython.display import Image, display

  display(Image(graph.get_graph().draw_mermaid_png()))
  ```

    <img src="https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/graph_api_image_8.png?fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=20e2a9e8c15760eb9ecb07fc411aa70e" alt="具有分支的复杂循环图" data-og-width="297" width="297" data-og-height="348" height="348" data-path="oss/images/graph_api_image_8.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/graph_api_image_8.png?w=280&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=65ee62a3adb7bedaf7571d9ecdacb908 280w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/graph_api_image_8.png?w=560&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=e7c4c3341baeed9c747082f69d2b3ded 560w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/graph_api_image_8.png?w=840&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=b64849cfc877d1b32422f6666d5f93a0 840w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/graph_api_image_8.png?w=1100&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=3d384eba95e1082504c7ef1d5309dfae 1100w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/graph_api_image_8.png?w=1650&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=2fef71e345a90e5c2321c0dfda15d91b 1650w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/graph_api_image_8.png?w=2500&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=09cf8e8ac3215e359e6e4304c09b3a9f 2500w" />

  这个图看起来很复杂，但可以概念化为[超级步](/oss/python/langgraph/graph-api#graphs)的循环：

  1. 节点A
  2. 节点B
  3. 节点C和D
  4. 节点A
  5. ...

  我们有一个四个超级步的循环，其中节点C和D同时执行。

  像之前一样调用图，我们看到我们在终止条件下完成两个完整的"圈"：

  ```python  theme={null}
  result = graph.invoke({"aggregate": []})
  ```

  ```
  节点A看到 []
  节点B看到 ['A']
  节点D看到 ['A', 'B']
  节点C看到 ['A', 'B']
  节点A看到 ['A', 'B', 'C', 'D']
  节点B看到 ['A', 'B', 'C', 'D', 'A']
  节点D看到 ['A', 'B', 'C', 'D', 'A', 'B']
  节点C看到 ['A', 'B', 'C', 'D', 'A', 'B']
  节点A看到 ['A', 'B', 'C', 'D', 'A', 'B', 'C', 'D']
  ```

  但是，如果我们设置递归限制为四，我们只完成一圈，因为每圈是四个超级步：

  ```python  theme={null}
  from langgraph.errors import GraphRecursionError

  try:
      result = graph.invoke({"aggregate": []}, {"recursion_limit": 4})
  except GraphRecursionError:
      print("递归错误")
  ```

  ```
  节点A看到 []
  节点B看到 ['A']
  节点C看到 ['A', 'B']
  节点D看到 ['A', 'B']
  节点A看到 ['A', 'B', 'C', 'D']
  递归错误
  ```
</Accordion>

## 异步

使用异步编程范式在运行[IO绑定](https://en.wikipedia.org/wiki/I/O_bound)代码并发时可以产生显著的性能改进（例如，向聊天模型提供商发出并发API请求）。

要将图的`sync`实现转换为`async`实现，您需要：

1. 更新`nodes`使用`async def`而不是`def`。
2. 适当更新内部代码以使用`await`。
3. 使用`.ainvoke`或`.astream`调用图。

因为许多LangChain对象实现了[Runnable协议](https://python.langchain.com/docs/expression_language/interface/)，它具有所有`sync`方法的`async`变体，所以将`sync`图升级为`async`图通常相当快。

参见下面的示例。为了演示底层LLM的异步调用，我们将包含一个聊天模型：

<Tabs>
  <Tab title="OpenAI">
    👉 阅读[OpenAI聊天模型集成文档](/oss/python/integrations/chat/openai/)

    ```shell  theme={null}
    pip install -U "langchain[openai]"
    ```

    <CodeGroup>
      ```python init_chat_model theme={null}
      import os
      from langchain.chat_models import init_chat_model

      os.environ["OPENAI_API_KEY"] = "sk-..."

      model = init_chat_model("openai:gpt-4.1")
      ```

      ```python Model Class theme={null}
      import os
      from langchain_openai import ChatOpenAI

      os.environ["OPENAI_API_KEY"] = "sk-..."

      model = ChatOpenAI(model="gpt-4.1")
      ```
    </CodeGroup>
  </Tab>

  <Tab title="Anthropic">
    👉 阅读[Anthropic聊天模型集成文档](/oss/python/integrations/chat/anthropic/)

    ```shell  theme={null}
    pip install -U "langchain[anthropic]"
    ```

    <CodeGroup>
      ```python init_chat_model theme={null}
      import os
      from langchain.chat_models import init_chat_model

      os.environ["ANTHROPIC_API_KEY"] = "sk-..."

      model = init_chat_model("anthropic:claude-sonnet-4-5")
      ```

      ```python Model Class theme={null}
      import os
      from langchain_anthropic import ChatAnthropic

      os.environ["ANTHROPIC_API_KEY"] = "sk-..."

      model = ChatAnthropic(model="claude-sonnet-4-5")
      ```
    </CodeGroup>
  </Tab>

  <Tab title="Azure">
    👉 阅读[Azure聊天模型集成文档](/oss/python/integrations/chat/azure_chat_openai/)

    ```shell  theme={null}
    pip install -U "langchain[openai]"
    ```

    <CodeGroup>
      ```python init_chat_model theme={null}
      import os
      from langchain.chat_models import init_chat_model

      os.environ["AZURE_OPENAI_API_KEY"] = "..."
      os.environ["AZURE_OPENAI_ENDPOINT"] = "..."
      os.environ["OPENAI_API_VERSION"] = "2025-03-01-preview"

      model = init_chat_model(
          "azure_openai:gpt-4.1",
          azure_deployment=os.environ["AZURE_OPENAI_DEPLOYMENT_NAME"],
      )
      ```

      ```python Model Class theme={null}
      import os
      from langchain_openai import AzureChatOpenAI

      os.environ["AZURE_OPENAI_API_KEY"] = "..."
      os.environ["AZURE_OPENAI_ENDPOINT"] = "..."
      os.environ["OPENAI_API_VERSION"] = "2025-03-01-preview"

      model = AzureChatOpenAI(
          model="gpt-4.1",
          azure_deployment=os.environ["AZURE_OPENAI_DEPLOYMENT_NAME"]
      )
      ```
    </CodeGroup>
  </Tab>

  <Tab title="Google Gemini">
    👉 阅读[Google GenAI聊天模型集成文档](/oss/python/integrations/chat/google_generative_ai/)

    ```shell  theme={null}
    pip install -U "langchain[google-genai]"
    ```

    <CodeGroup>
      ```python init_chat_model theme={null}
      import os
      from langchain.chat_models import init_chat_model

      os.environ["GOOGLE_API_KEY"] = "..."

      model = init_chat_model("google_genai:gemini-2.5-flash-lite")
      ```

      ```python Model Class theme={null}
      import os
      from langchain_google_genai import ChatGoogleGenerativeAI

      os.environ["GOOGLE_API_KEY"] = "..."

      model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite")
      ```
    </CodeGroup>
  </Tab>

  <Tab title="AWS Bedrock">
    👉 阅读[AWS Bedrock聊天模型集成文档](/oss/python/integrations/chat/bedrock/)

    ```shell  theme={null}
    pip install -U "langchain[aws]"
    ```

    <CodeGroup>
      ```python init_chat_model theme={null}
      from langchain.chat_models import init_chat_model

      # 按照以下步骤配置您的凭据：
      # https://docs.aws.amazon.com/bedrock/latest/userguide/getting-started.html

      model = init_chat_model(
          "anthropic.claude-3-5-sonnet-20240620-v1:0",
          model_provider="bedrock_converse",
      )
      ```

      ```python Model Class theme={null}
      from langchain_aws import ChatBedrock

      model = ChatBedrock(model="anthropic.claude-3-5-sonnet-20240620-v1:0")
      ```
    </CodeGroup>
  </Tab>
</Tabs>

```python  theme={null}
from langchain.chat_models import init_chat_model
from langgraph.graph import MessagesState, StateGraph

async def node(state: MessagesState):  # [!code highlight]
    new_message = await llm.ainvoke(state["messages"])  # [!code highlight]
    return {"messages": [new_message]}

builder = StateGraph(MessagesState).add_node(node).set_entry_point("node")
graph = builder.compile()

input_message = {"role": "user", "content": "Hello"}
result = await graph.ainvoke({"messages": [input_message]})  # [!code highlight]
```

<Tip>
  **异步流式传输**
  有关异步流式传输的示例，请参阅[流式传输指南](/oss/python/langgraph/streaming)。
</Tip>

## 使用`Command`结合控制流和状态更新

将控制流（边）和状态更新（节点）结合在一起可能很有用。例如，您可能希望**同时**执行状态更新**并决定**下一个要去哪个节点。LangGraph提供了一种方法，通过从节点函数返回[Command](https://langchain-ai.github.io/langgraph/reference/types/#langgraph.types.Command)对象来实现这一点：

```python  theme={null}
def my_node(state: State) -> Command[Literal["my_other_node"]]:
    return Command(
        # 状态更新
        update={"foo": "bar"},
        # 控制流
        goto="my_other_node"
    )
```

我们在下面展示一个端到端示例。让我们创建一个包含3个节点的简单图：A、B和C。我们将首先执行节点A，然后根据节点A的输出决定是去节点B还是节点C。

```python  theme={null}
import random
from typing_extensions import TypedDict, Literal
from langgraph.graph import StateGraph, START
from langgraph.types import Command

# 定义图状态
class State(TypedDict):
    foo: str

# 定义节点

def node_a(state: State) -> Command[Literal["node_b", "node_c"]]:
    print("调用A")
    value = random.choice(["b", "c"])
    # 这是条件边函数的替代
    if value == "b":
        goto = "node_b"
    else:
        goto = "node_c"

    # 注意Command如何让您**同时**更新图状态**并**路由到下一个节点
    return Command(
        # 这是状态更新
        update={"foo": value},
        # 这是边的替代
        goto=goto,
    )

def node_b(state: State):
    print("调用B")
    return {"foo": state["foo"] + "b"}

def node_c(state: State):
    print("调用C")
    return {"foo": state["foo"] + "c"}
```

我们现在可以使用上面的节点创建[`StateGraph`](https://reference.langchain.com/python/langchain/core/runnables/langchain_core.runnables.base.RunnableConfig.html)。注意，图没有用于路由的[条件边](/oss/python/langgraph/graph-api#conditional-edges)！这是因为控制流是在`node_a`中使用[`Command`](https://reference.langchain.com/python/langchain/core/runnables/langchain_core.runnables.base.RunnableConfig.html)定义的。

```python  theme={null}
builder = StateGraph(State)
builder.add_edge(START, "node_a")
builder.add_node(node_a)
builder.add_node(node_b)
builder.add_node(node_c)
# 注意：节点A、B和C之间没有边！

graph = builder.compile()
```

<Warning>
  您可能已经注意到，我们使用[`Command`](https://reference.langchain.com/python/langchain/core/runnables/langchain_core.runnables.base.RunnableConfig.html)作为返回类型注释，例如`Command[Literal["node_b", "node_c"]]`。这对于图渲染是必要的，并告诉LangGraph `node_a`可以导航到`node_b`和`node_c`。
</Warning>

```python  theme={null}
from IPython.display import display, Image

display(Image(graph.get_graph().draw_mermaid_png()))
```

<img src="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_11.png?fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=f11e5cddedbf2760d40533f294c44aea" alt="基于Command的图导航" data-og-width="232" width="232" data-og-height="333" height="333" data-path="oss/images/graph_api_image_11.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_11.png?w=280&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=c1b27d92b257a6c4ac57f34f007d0ee1 280w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_11.png?w=560&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=695d0062e5fb8ebea5525379edbba476 560w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_11.png?w=840&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=7bd3f779df628beba60a397674f85b59 840w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_11.png?w=1100&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=85a9194e8b4d9df2d01d10784dcf75d0 1100w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_11.png?w=1650&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=efd9118d4bcd6d1eb92760c573645fbd 1650w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_11.png?w=2500&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=1eb2a132386a64d18582af6978e4ac24 2500w" />

如果我们多次运行图，我们会看到它根据节点A中的随机选择采取不同路径（A -> B 或 A -> C）。

```python  theme={null}
graph.invoke({"foo": ""})
```

```
调用A
调用C
```

### 导航到父图中的节点

如果您使用[子图](/oss/python/langgraph/use-subgraphs)，您可能希望从子图内的节点导航到不同的子图（即父图中的不同节点）。为此，您可以在`Command`中指定`graph=Command.PARENT`：

```python  theme={null}
def my_node(state: State) -> Command[Literal["my_other_node"]]:
    return Command(
        update={"foo": "bar"},
        goto="other_subgraph",  # 其中`other_subgraph`是父图中的一个节点
        graph=Command.PARENT
    )
```

让我们使用上面的示例来演示这一点。我们将通过将上面示例中的`nodeA`更改为我们作为子图添加到父图中的单节点图来实现这一点。

<Warning>
  **使用`Command.PARENT`的状态更新**
  当您从子图节点向共享父图和子图[状态模式](/oss/python/langgraph/graph-api#schema)的键发送更新时，您**必须**为要在父图状态中更新的键定义[约简器](/oss/python/langgraph/graph-api#reducers)。请参见下面的示例。
</Warning>

```python  theme={null}
import operator
from typing_extensions import Annotated

class State(TypedDict):
    # 注意：我们在这里定义了一个约简器
    foo: Annotated[str, operator.add]  # [!code highlight]

def node_a(state: State):
    print("调用A")
    value = random.choice(["a", "b"])
    # 这是条件边函数的替代
    if value == "a":
        goto = "node_b"
    else:
        goto = "node_c"

    # 注意Command如何让您**同时**更新图状态**并**路由到下一个节点
    return Command(
        update={"foo": value},
        goto=goto,
        # 这告诉LangGraph导航到父图中的node_b或node_c
        # 注意：这将导航到相对于子图的最近的父图
        graph=Command.PARENT,  # [!code highlight]
    )

subgraph = StateGraph(State).add_node(node_a).add_edge(START, "node_a").compile()

def node_b(state: State):
    print("调用B")
    # 注意：既然我们已经定义了约简器，我们不需要手动将新字符附加到现有的'foo'值
    # 相反，约简器将自动附加这些（通过operator.add）
    return {"foo": "b"}  # [!code highlight]

def node_c(state: State):
    print("调用C")
    return {"foo": "c"}  # [!code highlight]

builder = StateGraph(State)
builder.add_edge(START, "subgraph")
builder.add_node("subgraph", subgraph)
builder.add_node(node_b)
builder.add_node(node_c)

graph = builder.compile()
```

```python  theme={null}
graph.invoke({"foo": ""})
```

```
调用A
调用C
```

### 在工具内部使用

一个常见的用例是从工具内部更新图状态。例如，在客户支持应用程序中，您可能希望在对话开始时基于他们的帐号或ID查找客户信息。要从工具更新图状态，您可以从工具返回`Command(update={"my_custom_key": "foo", "messages": [...]})`：

```python  theme={null}
@tool
def lookup_user_info(tool_call_id: Annotated[str, InjectedToolCallId], config: RunnableConfig):
    """使用此查找用户信息以更好地协助他们回答问题。"""
    user_info = get_user_info(config.get("configurable", {}).get("user_id"))
    return Command(
        update={
            # 更新状态键
            "user_info": user_info,
            # 更新消息历史
            "messages": [ToolMessage("Successfully looked up user information", tool_call_id=tool_call_id)]
        }
    )
```

<Warning>
  当您从工具返回[`Command`](https://reference.langchain.com/python/langchain/core/runnables/langchain_core.runnables.base.RunnableConfig.html)时，您**必须**将`messages`（或用于消息历史的任何状态键）包含在`Command.update`中，并且`messages`列表中的消息**必须**包含一个`ToolMessage`。这对于使结果消息历史记录有效是必要的（LLM提供商要求带有工具调用的AI消息后跟工具结果消息）。
</Warning>

如果您使用通过[`Command`](https://reference.langchain.com/python/langchain/core/runnables/langchain_core.runnables.base.RunnableConfig.html)更新状态的工具，我们建议使用预构建的[`ToolNode`](https://reference.langchain.com/python/langchain/agents/#langgraph.prebuilt.tool_node.ToolNode)，它会自动处理返回[`Command`](https://reference.langchain.com/python/langchain/core/runnables/langchain_core.runnables.base.RunnableConfig.html)对象并将其传播到图状态的工具。如果您正在编写调用工具的自定义节点，您需要手动将从工具返回的[`Command`](https://reference.langchain.com/python/langchain/core/runnables/langchain_core.runnables.base.RunnableConfig.html)对象作为节点的更新传播。

## 可视化您的图

这里我们演示如何可视化您创建的图。

您可以可视化任何任意的[图](https://langchain-ai.github.io/langgraph/reference/graphs/)，包括[StateGraph](https://langchain-ai.github.io/langgraph/reference/graphs/#langgraph.graph.state.StateGraph)。

让我们通过绘制分形来玩得开心:).

```python  theme={null}
import random
from typing import Annotated, Literal
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages

class State(TypedDict):
    messages: Annotated[list, add_messages]

class MyNode:
    def __init__(self, name: str):
        self.name = name
    def __call__(self, state: State):
        return {"messages": [("assistant", f"Called node {self.name}")]}

def route(state) -> Literal["entry_node", END]:
    if len(state["messages"]) > 10:
        return END
    return "entry_node"

def add_fractal_nodes(builder, current_node, level, max_level):
    if level > max_level:
        return
    # 在此级别创建的节点数
    num_nodes = random.randint(1, 3)  # 根据需要调整随机性
    for i in range(num_nodes):
        nm = ["A", "B", "C"][i]
        node_name = f"node_{current_node}_{nm}"
        builder.add_node(node_name, MyNode(node_name))
        builder.add_edge(current_node, node_name)
        # 递归添加更多节点
        r = random.random()
        if r > 0.2 and level + 1 < max_level:
            add_fractal_nodes(builder, node_name, level + 1, max_level)
        elif r > 0.05:
            builder.add_conditional_edges(node_name, route, node_name)
        else:
            # 结束
            builder.add_edge(node_name, END)

def build_fractal_graph(max_level: int):
    builder = StateGraph(State)
    entry_point = "entry_node"
    builder.add_node(entry_point, MyNode(entry_point))
    builder.add_edge(START, entry_point)
    add_fractal_nodes(builder, entry_point, 1, max_level)
    # 可选：如果需要，设置结束点
    builder.add_edge(entry_point, END)  # 或任何特定节点
    return builder.compile()

app = build_fractal_graph(3)
```

### Mermaid

我们还可以将图类转换为Mermaid语法。

```python  theme={null}
print(app.get_graph().draw_mermaid())
```

```
%%{init: {'flowchart': {'curve': 'linear'}}}%%
graph TD;
    tart__([<p>__start__</p>]):::first
    ry_node(entry_node)
    e_entry_node_A(node_entry_node_A)
    e_entry_node_B(node_entry_node_B)
    e_node_entry_node_B_A(node_node_entry_node_B_A)
    e_node_entry_node_B_B(node_node_entry_node_B_B)
    e_node_entry_node_B_C(node_node_entry_node_B_C)
    nd__([<p>__end__</p>]):::last
    tart__ --> entry_node;
    ry_node --> __end__;
    ry_node --> node_entry_node_A;
    ry_node --> node_entry_node_B;
    e_entry_node_B --> node_node_entry_node_B_A;
    e_entry_node_B --> node_node_entry_node_B_B;
    e_entry_node_B --> node_node_entry_node_B_C;
    e_entry_node_A -.-> entry_node;
    e_entry_node_A -.-> __end__;
    e_node_entry_node_B_A -.-> entry_node;
    e_node_entry_node_B_A -.-> __end__;
    e_node_entry_node_B_B -.-> entry_node;
    e_node_entry_node_B_B -.-> __end__;
    e_node_entry_node_B_C -.-> entry_node;
    e_node_entry_node_B_C -.-> __end__;
    ssDef default fill:#f2f0ff,line-height:1.2
    ssDef first fill-opacity:0
    ssDef last fill:#bfb6fc
```

### PNG

如果需要，我们可以将图渲染为`.png`。这里我们可以使用三个选项：

* 使用Mermaid.ink API（不需要额外包）
* 使用Mermaid + Pyppeteer（需要`pip install pyppeteer`）
* 使用graphviz（需要`pip install graphviz`）

**使用Mermaid.Ink**

默认情况下，`draw_mermaid_png()`使用Mermaid.Ink的API生成图表。

```python  theme={null}
from IPython.display import Image, display
from langchain_core.runnables.graph import CurveStyle, MermaidDrawMethod, NodeStyles

display(Image(app.get_graph().draw_mermaid_png()))
```

<img src="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_10.png?fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=6cb916b7c627e81c2816cc74ebf3f913" alt="分形图可视化" data-og-width="2382" width="2382" data-og-height="1131" height="1131" data-path="oss/images/graph_api_image_10.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_10.png?w=280&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=01b02e6994b97c652851bf1a5be524b5 280w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_10.png?w=560&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=9ac63a57750ff509e5bcf0662a141092 560w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_10.png?w=840&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=5458c09f31e42d0fd8f58ba85626d89c 840w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_10.png?w=1100&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=feb0a463b249cd838ad31105ef695214 1100w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_10.png?w=1650&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=1a83b92a2d3b428d9b788720a7e54184 1650w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/graph_api_image_10.png?w=2500&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=8bf42c6ee15584253dc036ff9b60191a 2500w" />

**使用Mermaid + Pyppeteer**

```python  theme={null}
import nest_asyncio

nest_asyncio.apply()  # Jupyter Notebook需要运行异步函数

display(
    Image(
        app.get_graph().draw_mermaid_png(
            curve_style=CurveStyle.LINEAR,
            node_colors=NodeStyles(first="#ffdfba", last="#baffc9", default="#fad7de"),
            wrap_label_n_words=9,
            output_file_path=None,
            draw_method=MermaidDrawMethod.PYPPETEER,
            background_color="white",
            padding=10,
        )
    )
)
```

**使用Graphviz**

```python  theme={null}
try:
    display(Image(app.get_graph().draw_png()))
except ImportError:
    print(
        "您可能需要安装pygraphviz的依赖项，更多信息请参见 https://github.com/pygraphviz/pygraphviz/blob/main/INSTALL.txt"
    )
```

***

<Callout icon="pen-to-square" iconType="regular">
  [在GitHub上编辑此页面的源代码。](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/use-graph-api.mdx)
</Callout>

<Tip icon="terminal" iconType="regular">
  [通过MCP以编程方式连接这些文档](/use-these-docs)，以通过Claude、VSCode等获得实时答案。
</Tip>