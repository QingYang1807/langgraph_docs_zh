# Graph API 概览

## 图（Graph）

从根本上说，LangGraph 将代理工作流建模为图。您可以使用三个关键组件定义代理的行为：

1. [`State`](#state)：表示应用程序当前快照的共享数据结构。它可以是任何数据类型，但通常使用共享状态模式定义。

2. [`Nodes`](#nodes)：编码代理逻辑的函数。它们接收当前状态作为输入，执行一些计算或副作用，并返回更新后的状态。

3. [`Edges`](#edges)：根据当前状态确定执行下一个`Node`的函数。它们可以是条件分支或固定转换。

通过组合`Nodes`和`Edges`，您可以创建复杂的、循环的工作流，随着时间的推移逐步发展状态。然而，真正的力量来自于LangGraph管理状态的方式。需要强调的是：`Nodes`和`Edges`不过是函数而已 - 它们可以包含LLM，也可以只是普通的旧代码。

简而言之：*节点执行工作，边缘决定下一步该做什么*。

LangGraph的底层图算法使用[消息传递](https://en.wikipedia.org/wiki/Message_passing)来定义一个通用程序。当一个节点完成其操作时，它会沿一个或多个边缘向其他节点发送消息。这些接收节点然后执行它们的功能，将结果消息传递给下一组节点，这个过程继续进行。受Google的[Pregel](https://research.google/pubs/pregel-a-system-for-large-scale-graph-processing/)系统启发，程序以离散的"超级步骤"（super-steps）进行。

一个超级步骤可以被视为对图节点的一次迭代。并行运行的节点属于同一个超级步骤，而顺序运行的节点属于不同的超级步骤。在图执行开始时，所有节点都处于`inactive`状态。当节点在其任何入边（或"通道"）上收到新消息（状态）时，该节点变为`active`状态。然后活动节点运行其函数并响应更新。在每个超级步骤结束时，没有入边消息的节点通过将自己标记为`inactive`来投票`halt`。当所有节点都处于`inactive`状态且没有消息在传输中时，图执行终止。

### StateGraph

[`StateGraph`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.StateGraph)类是要使用的主要图类。它由用户定义的`State`对象参数化。

### 编译您的图

要构建您的图，首先定义[state](#state)，然后添加[nodes](#nodes)和[edges](#edges)，然后编译它。编译您的图究竟是什么，为什么需要它？

编译是一个非常简单的步骤。它对您的图结构提供一些基本检查（没有孤立节点等）。它也是您可以指定运行时参数的地方，如[checkpointers](/oss/python/langgraph/persistence)和断点。您只需调用`.compile`方法来编译图：

```python  theme={null}
graph = graph_builder.compile(...)
```

您**必须**在使用图之前编译它。

## 状态（State）

您在定义图时要做第一件事是定义图的`State`。`State`包括图的[模式](#schema)以及指定如何应用更新的[`reducer`函数](#reducers)。`State`的模式将是图中所有`Nodes`和`Edges`的输入模式，可以是`TypedDict`或`Pydantic`模型。所有`Nodes`都将发出对`State`的更新，然后使用指定的`reducer`函数应用这些更新。

### 模式（Schema）

指定图模式的主要记录方式是使用[`TypedDict`](https://docs.python.org/3/library/typing.html#typing.TypedDict)。如果您想在状态中提供默认值，请使用[`dataclass`](https://docs.python.org/3/library/dataclasses.html)。如果您想要递归数据验证，我们也支持使用Pydantic的[BaseModel](/oss/python/langgraph/graph-api.md#use-pydantic-models-for-graph-state)作为图状态（尽管请注意，pydantic的性能不如`TypedDict`或`dataclass`）。

默认情况下，图将具有相同的输入和输出模式。如果要更改这一点，您也可以直接指定显式的输入和输出模式。当您有很多键，有些明确用于输入而其他用于输出时，这很有用。有关如何使用的指南，请参见[这里](/oss/python/langgraph/graph-api.md#define-input-and-output-schemas)。

#### 多个模式

通常，所有图节点都使用单个模式进行通信。这意味着它们将读写相同的状态通道。但是，在某些情况下，我们想要更多控制：

* 内部节点可以传递在图的输入/输出中不需要的信息。
* 我们可能还希望为图使用不同的输入/输出模式。例如，输出可能只包含一个相关的输出键。

可以让节点在图内部写入私有状态通道以进行内部节点通信。我们可以简单地定义一个私有模式`PrivateState`。

还可以为图定义显式的输入和输出模式。在这些情况下，我们定义一个"内部"模式，其中包含与图操作相关的所有键。但我们还定义`input`和`output`模式，它们是"内部"模式的子集，用于约束图的输入和输出。有关更多详细信息，请参见[本指南](/oss/python/langgraph/graph-api#define-input-and-output-schemas)。

让我们看一个例子：

```python  theme={null}
class InputState(TypedDict):
    user_input: str

class OutputState(TypedDict):
    graph_output: str

class OverallState(TypedDict):
    foo: str
    user_input: str
    graph_output: str

class PrivateState(TypedDict):
    bar: str

def node_1(state: InputState) -> OverallState:
    # 写入OverallState
    return {"foo": state["user_input"] + " name"}

def node_2(state: OverallState) -> PrivateState:
    # 从OverallState读取，写入PrivateState
    return {"bar": state["foo"] + " is"}

def node_3(state: PrivateState) -> OutputState:
    # 从PrivateState读取，写入OutputState
    return {"graph_output": state["bar"] + " Lance"}

builder = StateGraph(OverallState,input_schema=InputState,output_schema=OutputState)
builder.add_node("node_1", node_1)
builder.add_node("node_2", node_2)
builder.add_node("node_3", node_3)
builder.add_edge(START, "node_1")
builder.add_edge("node_1", "node_2")
builder.add_edge("node_2", "node_3")
builder.add_edge("node_3", END)

graph = builder.compile()
graph.invoke({"user_input":"My"})
# {'graph_output': 'My name is Lance'}
```

这里有两个细微但重要的点需要注意：

1. 我们将`state: InputState`作为输入模式传递给`node_1`。但是，我们写入`foo`，这是`OverallState`中的一个通道。如何写入到未包含在输入模式中的状态通道？这是因为节点*可以写入图状态中的任何状态通道*。图状态是初始化时定义的状态通道的并集，包括`OverallState`以及筛选器`InputState`和`OutputState`。

2. 我们使用`StateGraph(OverallState,input_schema=InputState,output_schema=OutputState)`初始化图。那么，如何在`node_2`中写入`PrivateState`？如果它没有在`StateGraph`初始化中传递，图如何访问这个模式？我们可以做到这一点，因为*节点也可以声明额外的状态通道*，只要状态模式定义存在。在这种情况下，`PrivateState`模式已定义，因此我们可以将`bar`添加为图中的新状态通道并写入它。

### Reducer函数

Reducer函数是理解节点的更新如何应用于`State`的关键。`State`中的每个键都有其独立的reducer函数。如果没有明确指定reducer函数，则假设对该键的所有更新都应该覆盖它。有几种不同类型的reducer函数，从默认类型开始：

#### 默认Reducer

这两个示例展示了如何使用默认reducer：

**示例A:**

```python  theme={null}
from typing_extensions import TypedDict

class State(TypedDict):
    foo: int
    bar: list[str]
```

在这个例子中，没有为任何键指定reducer函数。让我们假设图的输入是：

`{"foo": 1, "bar": ["hi"]}`。然后假设第一个`Node`返回`{"foo": 2}`。这被视为对状态的更新。注意，节点不需要返回整个`State`模式 - 只需返回更新。应用此更新后，`State`将是`{"foo": 2, "bar": ["hi"]}`。如果第二个节点返回`{"bar": ["bye"]}`，那么`State`将是`{"foo": 2, "bar": ["bye"]}`

**示例B:**

```python  theme={null}
from typing import Annotated
from typing_extensions import TypedDict
from operator import add

class State(TypedDict):
    foo: int
    bar: Annotated[list[str], add]
```

在这个例子中，我们使用`Annotated`类型为第二个键（`bar`）指定了reducer函数（`operator.add`）。注意第一个键保持不变。让我们假设图的输入是`{"foo": 1, "bar": ["hi"]}`。然后假设第一个`Node`返回`{"foo": 2}`。这被视为对状态的更新。注意，节点不需要返回整个`State`模式 - 只需返回更新。应用此更新后，`State`将是`{"foo": 2, "bar": ["hi"]}`。如果第二个节点返回`{"bar": ["bye"]}`，那么`State`将是`{"foo": 2, "bar": ["hi", "bye"]}`。注意这里，`bar`键是通过将两个列表相加来更新的。

### 在图状态中使用消息

#### 为什么使用消息？

大多数现代LLM提供商都有聊天模型界面，接受消息列表作为输入。特别是LangChain的[`ChatModel`](https://python.langchain.com/docs/concepts/#chat-models)接受`Message`对象的列表作为输入。这些消息有多种形式，如[`HumanMessage`](https://reference.langchain.com/python/langchain/messages/#langchain.messages.HumanMessage)（用户输入）或[`AIMessage`](https://reference.langchain.com/python/langchain/messages/#langchain.messages.AIMessage)（LLM响应）。要阅读有关消息对象的更多信息，请参考[此](https://python.langchain.com/docs/concepts/#messages)概念指南。

#### 在您的图中使用消息

在许多情况下，在图状态中将先前的对话历史记录存储为消息列表很有帮助。为此，我们可以向图状态添加一个键（通道），用于存储`Message`对象的列表，并使用reducer函数对其进行注释（参见下面示例中的`messages`键）。reducer函数对于告诉图如何使用每个状态更新更新状态中的消息列表非常重要（例如，当节点发送更新时）。如果您没有指定reducer，每个状态更新都会用最近提供的值覆盖消息列表。如果您只想将消息附加到现有列表中，可以使用`operator.add`作为reducer。

但是，您可能还想手动更新图状态中的消息（例如，人在回路中）。如果您使用`operator.add`，您发送到图的手动状态更新将被附加到现有消息列表中，而不是更新现有消息。为了避免这种情况，您需要一个能够跟踪消息ID并在更新时覆盖现有消息的reducer。为此，您可以使用预构建的[`add_messages`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.message.add_messages)函数。对于全新消息，它将简单附加到现有列表，但它也会正确处理现有消息的更新。

#### 序列化

除了跟踪消息ID外，[`add_messages`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.message.add_messages)函数还尝试在`messages`通道上接收到状态更新时将消息反序列化为LangChain `Message`对象。有关LangChain序列化/反序列化的更多信息，请参见[这里](https://python.langchain.com/docs/how_to/serialization/)。这允许以以下格式发送图输入/状态更新：

```python  theme={null}
# 这是支持的
{"messages": [HumanMessage(content="message")]}

# 这也是支持的
{"messages": [{"type": "human", "content": "message"}]}
```

由于在使用[`add_messages`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.message.add_messages)时状态更新总是被反序列化为LangChain `Messages`，您应该使用点符号来访问消息属性，如`state["messages"][-1].content`。下面是一个使用[`add_messages`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.message.add_messages)作为其reducer函数的图的示例。

```python  theme={null}
from langchain.messages import AnyMessage
from langgraph.graph.message import add_messages
from typing import Annotated
from typing_extensions import TypedDict

class GraphState(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]
```

#### MessagesState

由于在状态中有消息列表非常常见，因此存在一个名为`MessagesState`的预构建状态，它使使用消息变得容易。`MessagesState`被定义为具有单个`messages`键，它是`AnyMessage`对象的列表，并使用[`add_messages`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.message.add_messages) reducer。通常，除了消息外还有更多状态需要跟踪，因此我们看到人们子类化此状态并添加更多字段，如：

```python  theme={null}
from langgraph.graph import MessagesState

class State(MessagesState):
    documents: list[str]
```

## 节点（Nodes）

在LangGraph中，节点是Python函数（可以是同步或异步），接受以下参数：

1. `state`：图的[state](#state)
2. `config`：一个[`RunnableConfig`](https://reference.langchain.com/python/langchain_core/runnables/#langchain_core.runnables.RunnableConfig)对象，包含配置信息，如`thread_id`和跟踪信息，如`tags`
3. `runtime`：一个`Runtime`对象，包含[运行时`context`](#runtime-context)和其他信息，如`store`和`stream_writer`

类似于`NetworkX`，您可以使用[`add_node`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.StateGraph.add_node)方法将这些节点添加到图中：

```python  theme={null}
from dataclasses import dataclass
from typing_extensions import TypedDict

from langchain_core.runnables import RunnableConfig
from langgraph.graph import StateGraph
from langgraph.runtime import Runtime

class State(TypedDict):
    input: str
    results: str

@dataclass
class Context:
    user_id: str

builder = StateGraph(State)

def plain_node(state: State):
    return state

def node_with_runtime(state: State, runtime: Runtime[Context]):
    print("In node: ", runtime.context.user_id)
    return {"results": f"Hello, {state['input']}!"}

def node_with_config(state: State, config: RunnableConfig):
    print("In node with thread_id: ", config["configurable"]["thread_id"])
    return {"results": f"Hello, {state['input']}!"}


builder.add_node("plain_node", plain_node)
builder.add_node("node_with_runtime", node_with_runtime)
builder.add_node("node_with_config", node_with_config)
...
```

在幕后，函数被转换为[RunnableLambda](https://python.langchain.com/api_reference/core/runnables/langchain_core.runnables.base.RunnableLambda.html)s，这为您的函数添加批处理和异步支持，以及原生跟踪和调试。

如果您向图中添加节点而没有指定名称，它将被赋予一个默认名称，等同于函数名。

```python  theme={null}
builder.add_node(my_node)
# 您可以通过引用为`"my_node"`来创建到此节点的边
```

### `START` 节点

[`START`](https://reference.langchain.com/python/langgraph/constants/#langgraph.constants.START)节点是一个特殊节点，代表将用户输入发送到图的节点。引用此节点的主要目的是确定应首先调用哪些节点。

```python  theme={null}
from langgraph.graph import START

graph.add_edge(START, "node_a")
```

### `END` 节点

`END`节点是一个表示终端节点的特殊节点。当您想要表示哪些边在完成后没有操作时，会引用此节点。

```python  theme={null}
from langgraph.graph import END

graph.add_edge("node_a", END)
```

### 节点缓存

LangGraph支持基于节点输入的任务/节点缓存。要使用缓存：

* 在编译图（或指定入口点）时指定缓存
* 为节点指定缓存策略。每个缓存策略支持：
  * `key_func`用于基于节点输入生成缓存键，默认为使用pickle对输入进行`hash`。
  * `ttl`，缓存的生命周期（秒）。如果未指定，缓存将永不过期。

例如：

```python  theme={null}
import time
from typing_extensions import TypedDict
from langgraph.graph import StateGraph
from langgraph.cache.memory import InMemoryCache
from langgraph.types import CachePolicy


class State(TypedDict):
    x: int
    result: int


builder = StateGraph(State)


def expensive_node(state: State) -> dict[str, int]:
    # 昂贵计算
    time.sleep(2)
    return {"result": state["x"] * 2}


builder.add_node("expensive_node", expensive_node, cache_policy=CachePolicy(ttl=3))
builder.set_entry_point("expensive_node")
builder.set_finish_point("expensive_node")

graph = builder.compile(cache=InMemoryCache())

print(graph.invoke({"x": 5}, stream_mode='updates'))    # [!code highlight]
# [{'expensive_node': {'result': 10}}]
print(graph.invoke({"x": 5}, stream_mode='updates'))    # [!code highlight]
# [{'expensive_node': {'result': 10}, '__metadata__': {'cached': True}}]
```

1. 第一次运行需要两秒运行（由于模拟的昂贵计算）。
2. 第二次运行利用缓存并快速返回。

## 边（Edges）

边定义了逻辑如何路由以及图如何决定停止。这是您的代理工作方式以及不同节点如何相互通信的重要部分。有几种关键的边类型：

* 普通边（Normal Edges）：直接从一个节点到下一个节点。
* 条件边（Conditional Edges）：调用函数以确定下一个要去哪个（些）节点。
* 入口点（Entry Point）：当用户输入到达时首先调用的节点。
* 条件入口点（Conditional Entry Point）：调用函数以确定当用户输入到达时首先调用的哪个（些）节点。

一个节点可以有多个出边。如果一个节点有多个出边，**所有**这些目标节点将在下一个超级步骤中并行执行。

### 普通边

如果您**总是**想从节点A到节点B，可以直接使用[`add_edge`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.StateGraph.add_edge)方法。

```python  theme={null}
graph.add_edge("node_a", "node_b")
```

### 条件边

如果您想要**可选地**路由到一个或多个边（或可选地终止），可以使用[`add_conditional_edges`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.StateGraph.add_conditional_edges)方法。此方法接受一个节点名称和一个"路由函数"在该节点执行后调用：

```python  theme={null}
graph.add_conditional_edges("node_a", routing_function)
```

类似于节点，`routing_function`接受图的当前`state`并返回一个值。

默认情况下，返回值`routing_function`用作下一个要发送状态的节点（或节点列表）的名称。所有这些节点将在下一个超级步骤中并行运行。

您可以选择提供一个将`routing_function`的输出映射到下一个节点名称的字典。

```python  theme={null}
graph.add_conditional_edges("node_a", routing_function, {True: "node_b", False: "node_c"})
```

<Tip>
  如果您想在单个函数中组合状态更新和路由，请使用[`Command`](#command)而不是条件边。
</Tip>

### 入口点

入口点是图启动时运行的第一个节点。您可以使用[`add_edge`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.StateGraph.add_edge)方法从虚拟的[`START`](https://reference.langchain.com/python/langgraph/constants/#langgraph.constants.START)节点到第一个要执行的节点来指定进入图的位置。

```python  theme={null}
from langgraph.graph import START

graph.add_edge(START, "node_a")
```

### 条件入口点

条件入口点允许您根据自定义逻辑从不同节点开始。您可以使用[`add_conditional_edges`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.StateGraph.add_conditional_edges)从虚拟的[`START`](https://reference.langchain.com/python/langgraph/constants/#langgraph.constants.START)节点来实现这一点。

```python  theme={null}
from langgraph.graph import START

graph.add_conditional_edges(START, routing_function)
```

您可以选择提供一个将`routing_function`的输出映射到下一个节点名称的字典。

```python  theme={null}
graph.add_conditional_edges(START, routing_function, {True: "node_b", False: "node_c"})
```

## `Send`

默认情况下，`Nodes`和`Edges`是预先定义的，并在相同的共享状态上运行。然而，在某些情况下，确切的边是预先不知道的，或者您可能希望同时存在不同版本的`State`。一个常见的例子是[map-reduce](/oss/python/langgraph/graph-api#map-reduce-and-the-send-api)设计模式。在这种设计模式中，第一个节点可能生成一个对象列表，您可能希望将某个其他节点应用于所有这些对象。对象的数量可能预先不知道（意味着边的数量可能不知道），并且下游`Node`的输入`State`应该是不同的（每个生成的对象一个）。

为了支持这种设计模式，LangGraph支持从条件边返回[`Send`](https://reference.langchain.com/python/langgraph/types/#langgraph.types.Send)对象。`Send`接受两个参数：第一个是节点名称，第二个是要传递给该节点的状态。

```python  theme={null}
def continue_to_jokes(state: OverallState):
    return [Send("generate_joke", {"subject": s}) for s in state['subjects']]

graph.add_conditional_edges("node_a", continue_to_jokes)
```

## `Command`

结合控制流（边）和状态更新（节点）可能很有用。例如，您可能想在**同一个**节点中**既**执行状态更新**又**决定下一个要去哪个节点。LangGraph通过从节点函数返回[`Command`](https://reference.langchain.com/python/langgraph/types/#langgraph.types.Command)对象提供了一种方法来实现：

```python  theme={null}
def my_node(state: State) -> Command[Literal["my_other_node"]]:
    return Command(
        # 状态更新
        update={"foo": "bar"},
        # 控制流
        goto="my_other_node"
    )
```

使用[`Command`](https://reference.langchain.com/python/langgraph/types/#langgraph.types.Command)您还可以实现动态控制流行为（与[条件边](#conditional-edges)相同）：

```python  theme={null}
def my_node(state: State) -> Command[Literal["my_other_node"]]:
    if state["foo"] == "bar":
        return Command(update={"foo": "baz"}, goto="my_other_node")
```

<Note>
  在节点函数中返回[`Command`](https://reference.langchain.com/python/langgraph/types/#langgraph.types.Command)时，您必须添加返回类型注释，注明节点正在路由到的节点名称列表，例如`Command[Literal["my_other_node"]]`。这对于图渲染是必要的，并告诉LangGraph`my_node`可以导航到`my_other_node`。
</Note>

查看此[使用指南](/oss/python/langgraph/graph-api.md#combine-control-flow-and-state-updates-with-command)，了解如何使用[`Command`](https://reference.langchain.com/python/langgraph/types/#langgraph.types.Command)的端到端示例。

### 何时应使用Command而不是条件边？

* 当您需要**同时**更新图状态**和**路由到不同节点时，使用[`Command`](https://reference.langchain.com/python/langgraph/types/#langgraph.types.Command)。例如，在实现[多代理交接](/oss/python/langchain/multi-agent#handoffs)时，路由到不同的代理并将一些信息传递给该代理很重要。
* 使用[条件边](#conditional-edges)在节点之间条件性地路由而不更新状态。

### 导航到父图中的节点

如果您使用[子图](/oss/python/langgraph/use-subgraphs)，您可能想要从子图内的节点导航到不同的子图（即父图中的不同节点）。为此，您可以在[`Command`](https://reference.langchain.com/python/langgraph/types/#langgraph.types.Command)中指定`graph=Command.PARENT`：

```python  theme={null}
def my_node(state: State) -> Command[Literal["other_subgraph"]]:
    return Command(
        update={"foo": "bar"},
        goto="other_subgraph",  # 其中`other_subgraph`是父图中的一个节点
        graph=Command.PARENT
    )
```

<Note>
  将`graph`设置为`Command.PARENT`将导航到最近的父图。

  当您从子图节点向父图节点发送更新时，对于父图和子图[state schemas](#schema)共享的键，您**必须**为父图状态中正在更新的键定义一个[reducer](#reducers)。参见此[示例](/oss/python/langgraph/graph-api.md#navigate-to-a-node-in-a-parent-graph)。
</Note>

这在实现[多代理交接](/oss/python/langchain/multi-agent#handoffs)时特别有用。

有关详细信息，请查看[本指南](/oss/python/langgraph/graph-api.md#navigate-to-a-node-in-a-parent-graph)。

### 在工具内部使用

一个常见用例是从工具内部更新图状态。例如，在客户支持应用程序中，您可能想在对话开始时根据客户的账户号码或ID查找客户信息。

有关详细信息，请参考[本指南](/oss/python/langgraph/graph-api.md#use-inside-tools)。

### 人在回路中

[`Command`](https://reference.langchain.com/python/langgraph/types/#langgraph.types.Command)是人在回路工作流程的重要组成部分：使用`interrupt()`收集用户输入时，[`Command`](https://reference.langchain.com/python/langgraph/types/#langgraph.types.Command)随后用于提供输入并通过`Command(resume="User input")`恢复执行。有关更多信息，请查看此[概念指南](/oss/python/langgraph/interrupts)。

## 图迁移

LangGraph可以轻松处理图定义（节点、边和状态）的迁移，即使使用checkpointer来跟踪状态也是如此。

* 对于图末尾的线程（即未中断的），您可以更改图的整个拓扑结构（即所有节点和边，添加、删除、重命名等）
* 对于当前中断的线程，我们支持除重命名/删除节点外的所有拓扑更改（因为该线程现在可能即将进入一个不再存在的节点）--如果这是障碍，请联系我们，我们可以优先解决。
* 对于修改状态，我们完全支持向后和向前兼容，用于添加和删除键
* 在现有线程中被重命名的状态键会失去其保存的状态
* 对于以不兼容方式更改其类型的键，在具有来自更改之前状态的线程中 currently 可能会导致问题--如果这是障碍，请联系我们，我们可以优先解决。

## 运行时上下文

创建图时，您可以为传递给节点的运行时上下文指定`context_schema`。这对于传递不属于图状态的信息给节点很有用。例如，您可能想要传递依赖项，如模型名称或数据库连接。

```python  theme={null}
@dataclass
class ContextSchema:
    llm_provider: str = "openai"

graph = StateGraph(State, context_schema=ContextSchema)
```

然后您可以使用`invoke`方法的`context`参数将此上下文传递给图。

```python  theme={null}
graph.invoke(inputs, context={"llm_provider": "anthropic"})
```

然后您可以在节点或条件边缘内部访问和使用此上下文：

```python  theme={null}
from langgraph.runtime import Runtime

def node_a(state: State, runtime: Runtime[ContextSchema]):
    llm = get_llm(runtime.context.llm_provider)
    # ...
```

有关配置的完整分解，请参阅[本指南](/oss/python/langgraph/use-graph-api#add-runtime-configuration)。

### 递归限制

递归限制设置图在单个执行期间可以执行的最大[超级步骤](#graphs)数。一旦达到限制，LangGraph将引发`GraphRecursionError`。默认情况下，此值设置为25步。递归限制可以在任何图上在运行时设置，并通过配置字典传递给`invoke`/`stream`。重要的是，`recursion_limit`是一个独立的`config`键，不应像所有其他用户定义的配置一样传递在`configurable`键内。参见下面的示例：

```python  theme={null}
graph.invoke(inputs, config={"recursion_limit": 5}, context={"llm": "anthropic"})
```

阅读[本使用指南](/oss/python/langgraph/graph-api#impose-a-recursion-limit)以了解有关递归限制如何工作的更多信息。

## 可视化

能够可视化图通常很有帮助，特别是当图变得更复杂时。LangGraph提供了几种内置的图可视化方法。有关更多信息，请参见[本使用指南](/oss/python/langchain/graph-api.md#visualize-your-graph)。

***

<Callout icon="pen-to-square" iconType="regular">
  [在GitHub上编辑此页面的源代码。](https://github.com/langchain-ai/docs/edit/main/src/oss/langchain/graph-api.mdx)
</Callout>

<Tip icon="terminal" iconType="regular">
  [通过MCP以编程方式连接这些文档](/use-these-docs)到Claude、VSCode等，以获得实时答案。
</Tip>