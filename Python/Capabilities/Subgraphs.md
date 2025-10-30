# 子图(Subgraphs)

本指南解释了使用子图的机制。子图是作为另一个图中的[节点](/oss/python/langgraph/graph-api#nodes)使用的[图](/oss/python/langgraph/graph-api#graphs)。

子图的用途包括：

* 构建[多智能体系统](/oss/python/langchain/multi-agent)
* 在多个图中重用一组节点
* 分发开发：当您希望不同的团队独立地处理图的不同部分时，您可以将每个部分定义为一个子图，只要子图接口（输入和输出模式）得到尊重，父图就可以在不了解子图任何细节的情况下构建

在添加子图时，您需要定义父图和子图如何通信：

* [从节点调用图](#invoke-a-graph-from-a-node) — 子图在父图的节点内部被调用
* [将图作为节点添加](#add-a-graph-as-a-node) — 子图直接作为父图中的节点添加，并与父图**共享[状态键](/oss/python/langgraph/graph-api#state)**

## 设置

<CodeGroup>
  ```bash pip theme={null}
  pip install -U langgraph
  ```

  ```bash uv theme={null}
  uv add langgraph
  ```
</CodeGroup>

<Tip>
  **为LangGraph开发设置LangSmith**
  注册[LangSmith](https://smith.langchain.com)以快速发现并改进您的LangGraph项目性能。LangSmith允许您使用跟踪数据来调试、测试和监控使用LangGraph构建的LLM应用 — 更多关于如何入门的信息请阅读[这里](https://docs.smith.langchain.com)。
</Tip>

## 从节点调用图

实现子图的一种简单方法是从另一个图的节点内调用图。在这种情况下，子图可以与父图具有**完全不同的模式**（无共享键）。例如，您可能希望在[多智能体](/oss/python/langchain/multi-agent)系统中为每个智能体保留私有消息历史记录。

如果您的应用属于这种情况，您需要定义一个**调用子图的节点函数**。此函数需要在调用子图之前将输入（父级）状态转换子图状态，并在从节点返回状态更新之前将结果转换回父级状态。

```python  theme={null}
from typing_extensions import TypedDict
from langgraph.graph.state import StateGraph, START

class SubgraphState(TypedDict):
    bar: str

# 子图

def subgraph_node_1(state: SubgraphState):
    return {"bar": "hi! " + state["bar"]}

subgraph_builder = StateGraph(SubgraphState)
subgraph_builder.add_node(subgraph_node_1)
subgraph_builder.add_edge(START, "subgraph_node_1")
subgraph = subgraph_builder.compile()

# 父图

class State(TypedDict):
    foo: str

def call_subgraph(state: State):
    # 将状态转换到子图状态
    subgraph_output = subgraph.invoke({"bar": state["foo"]})  # [!code highlight]
    # 将响应转换回父级状态
    return {"foo": subgraph_output["bar"]}

builder = StateGraph(State)
builder.add_node("node_1", call_subgraph)
builder.add_edge(START, "node_1")
graph = builder.compile()
```

<Accordion title="完整示例：不同的状态模式">
  ```python  theme={null}
  from typing_extensions import TypedDict
  from langgraph.graph.state import StateGraph, START

  # 定义子图
  class SubgraphState(TypedDict):
      # 注意这些键都不与父图状态共享
      bar: str
      baz: str

  def subgraph_node_1(state: SubgraphState):
      return {"baz": "baz"}

  def subgraph_node_2(state: SubgraphState):
      return {"bar": state["bar"] + state["baz"]}

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

  def node_2(state: ParentState):
      # 将状态转换到子图状态
      response = subgraph.invoke({"bar": state["foo"]})
      # 将响应转换回父级状态
      return {"foo": response["bar"]}


  builder = StateGraph(ParentState)
  builder.add_node("node_1", node_1)
  builder.add_node("node_2", node_2)
  builder.add_edge(START, "node_1")
  builder.add_edge("node_1", "node_2")
  graph = builder.compile()

  for chunk in graph.stream({"foo": "foo"}, subgraphs=True):
      print(chunk)
  ```

  ```
  ((), {'node_1': {'foo': 'hi! foo'}})
  (('node_2:9c36dd0f-151a-cb42-cbad-fa2f851f9ab7',), {'grandchild_1': {'my_grandchild_key': 'hi Bob, how are you'}})
  (('node_2:9c36dd0f-151a-cb42-cbad-fa2f851f9ab7',), {'grandchild_2': {'bar': 'hi! foobaz'}})
  ((), {'node_2': {'foo': 'hi! foobaz'}})
  ```
</Accordion>

<Accordion title="完整示例：不同的状态模式（两层子图）">
  这是一个具有两层子图的示例：父图 -> 子图 -> 孙图。

  ```python  theme={null}
  # 孙图
  from typing_extensions import TypedDict
  from langgraph.graph.state import StateGraph, START, END

  class GrandChildState(TypedDict):
      my_grandchild_key: str

  def grandchild_1(state: GrandChildState) -> GrandChildState:
      # 注意：此处无法访问子图或父图的键
      return {"my_grandchild_key": state["my_grandchild_key"] + ", how are you"}


  grandchild = StateGraph(GrandChildState)
  grandchild.add_node("grandchild_1", grandchild_1)

  grandchild.add_edge(START, "grandchild_1")
  grandchild.add_edge("grandchild_1", END)

  grandchild_graph = grandchild.compile()

  # 子图
  class ChildState(TypedDict):
      my_child_key: str

  def call_grandchild_graph(state: ChildState) -> ChildState:
      # 注意：此处无法访问父图或孙图的键
      grandchild_graph_input = {"my_grandchild_key": state["my_child_key"]}
      grandchild_graph_output = grandchild_graph.invoke(grandchild_graph_input)
      return {"my_child_key": grandchild_graph_output["my_grandchild_key"] + " today?"}

  child = StateGraph(ChildState)
  # 我们在这里传递一个函数，而不是仅仅编译的图（`grandchild_graph`）
  child.add_node("child_1", call_grandchild_graph)
  child.add_edge(START, "child_1")
  child.add_edge("child_1", END)
  child_graph = child.compile()

  # 父图
  class ParentState(TypedDict):
      my_key: str

  def parent_1(state: ParentState) -> ParentState:
      # 注意：此处无法访问子图或孙图的键
      return {"my_key": "hi " + state["my_key"]}

  def parent_2(state: ParentState) -> ParentState:
      return {"my_key": state["my_key"] + " bye!"}

  def call_child_graph(state: ParentState) -> ParentState:
      child_graph_input = {"my_child_key": state["my_key"]}
      child_graph_output = child_graph.invoke(child_graph_input)
      return {"my_key": child_graph_output["my_child_key"]}

  parent = StateGraph(ParentState)
  parent.add_node("parent_1", parent_1)
  # 我们在这里传递一个函数，而不是仅仅编译的图（`child_graph`）
  parent.add_node("child", call_child_graph)
  parent.add_node("parent_2", parent_2)

  parent.add_edge(START, "parent_1")
  parent.add_edge("parent_1", "child")
  parent.add_edge("child", "parent_2")
  parent.add_edge("parent_2", END)

  parent_graph = parent.compile()

  for chunk in parent_graph.stream({"my_key": "Bob"}, subgraphs=True):
      print(chunk)
  ```

  ```
  ((), {'parent_1': {'my_key': 'hi Bob'}})
  (('child:2e26e9ce-602f-862c-aa66-1ea5a4655e3b', 'child_1:781bb3b1-3971-84ce-810b-acf819a03f9c'), {'grandchild_1': {'my_grandchild_key': 'hi Bob, how are you'}})
  (('child:2e26e9ce-602f-862c-aa66-1ea5a4655e3b',), {'child_1': {'my_child_key': 'hi Bob, how are you today?'}})
  ((), {'child': {'my_key': 'hi Bob, how are you today?'}})
  ((), {'parent_2': {'my_key': 'hi Bob, how are you today? bye!'}})
  ```
</Accordion>

## 将图作为节点添加

当父图和子图可以在[模式](/oss/python/langgraph/graph-api#state)中通过共享状态键（通道）通信时，您可以将图作为另一个图中的[节点](/oss/python/langgraph/graph-api#nodes)添加。例如，在[多智能体](/oss/python/langchain/multi-agent)系统中，智能体通常通过共享的[messages](/oss/python/langgraph/graph-api#why-use-messages)键进行通信。

<img src="https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/subgraph.png?fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=c280df5c968cd4237b0b5d03823d8946" alt="SQL智能体图" style={{ height: "450px" }} data-og-width="1177" width="1177" data-og-height="818" height="818" data-path="oss/images/subgraph.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/subgraph.png?w=280&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=e3d08dae8fb81e15b4d8069a48999eac 280w, https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/subgraph.png?w=560&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=8d8942031ba051119e0cb772ef697e0b 560w, https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/subgraph.png?w=840&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=0d5285bd104c542fe660bc09fed53e5e 840w, https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/subgraph.png?w=1100&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=32bc8ffa0eda13a0f3bb163631774a60 1100w, https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/subgraph.png?w=1650&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=6a511f3b9dc44383614803d32390875a 1650w, https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/subgraph.png?w=2500&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=169d55e154e5ea0146a57373235f768e 2500w" />

如果您的子图与父图共享状态键，您可以按照以下步骤将其添加到您的图中：

1. 定义子图工作流（下面的示例中的`subgraph_builder`）并编译它
2. 在定义父图工作流时，将编译的子图传递给[`add_node`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.StateGraph.add_node)方法

```python  theme={null}
from typing_extensions import TypedDict
from langgraph.graph.state import StateGraph, START

class State(TypedDict):
    foo: str

# 子图

def subgraph_node_1(state: State):
    return {"foo": "hi! " + state["foo"]}

subgraph_builder = StateGraph(State)
subgraph_builder.add_node(subgraph_node_1)
subgraph_builder.add_edge(START, "subgraph_node_1")
subgraph = subgraph_builder.compile()

# 父图

builder = StateGraph(State)
builder.add_node("node_1", subgraph)  # [!code highlight]
builder.add_edge(START, "node_1")
graph = builder.compile()
```

<Accordion title="完整示例：共享状态模式">
  ```python  theme={null}
  from typing_extensions import TypedDict
  from langgraph.graph.state import StateGraph, START

  # 定义子图
  class SubgraphState(TypedDict):
      foo: str  # 与父图状态共享
      bar: str  # 仅SubgraphState私有

  def subgraph_node_1(state: SubgraphState):
      return {"bar": "bar"}

  def subgraph_node_2(state: SubgraphState):
      # 注意这个节点使用了只在子图中可用的状态键（'bar'）
      # 并且在共享状态键（'foo'）上发送更新
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

  for chunk in graph.stream({"foo": "foo"}):
      print(chunk)
  ```

  ```
  {'node_1': {'foo': 'hi! foo'}}
  {'node_2': {'foo': 'hi! foobar'}}
  ```
</Accordion>

## 添加持久性

您只需要**在编译父图时提供检查点(checkpointer)**。LangGraph会自动将检查点传播到子图。

```python  theme={null}
from langgraph.graph import START, StateGraph
from langgraph.checkpoint.memory import MemorySaver
from typing_extensions import TypedDict

class State(TypedDict):
    foo: str

# 子图

def subgraph_node_1(state: State):
    return {"foo": state["foo"] + "bar"}

subgraph_builder = StateGraph(State)
subgraph_builder.add_node(subgraph_node_1)
subgraph_builder.add_edge(START, "subgraph_node_1")
subgraph = subgraph_builder.compile()

# 父图

builder = StateGraph(State)
builder.add_node("node_1", subgraph)
builder.add_edge(START, "node_1")

checkpointer = MemorySaver()
graph = builder.compile(checkpointer=checkpointer)
```

如果您希望子图**拥有自己的内存**，可以使用适当的检查点选项对其进行编译。这在[多智能体](/oss/python/langchain/multi-agent)系统中很有用，如果您希望智能体跟踪其内部消息历史：

```python  theme={null}
subgraph_builder = StateGraph(...)
subgraph = subgraph_builder.compile(checkpointer=True)
```

## 查看子图状态

当您启用[持久性](/oss/python/langgraph/persistence)时，您可以通过适当的方法[检查图状态](/oss/python/langgraph/persistence#checkpoints)（检查点）。要查看子图状态，可以使用subgraphs选项。

您可以通过`graph.get_state(config)`检查图状态。要查看子图状态，可以使用`graph.get_state(config, subgraphs=True)`。

<Warning>
  **仅在中断时可用**
  子图状态**只能在子图被中断时查看**。一旦您恢复图，将无法访问子图状态。
</Warning>

<Accordion title="查看中断的子图状态">
  ```python  theme={null}
  from langgraph.graph import START, StateGraph
  from langgraph.checkpoint.memory import MemorySaver
  from langgraph.types import interrupt, Command
  from typing_extensions import TypedDict

  class State(TypedDict):
      foo: str

  # 子图

  def subgraph_node_1(state: State):
      value = interrupt("Provide value:")
      return {"foo": state["foo"] + value}

  subgraph_builder = StateGraph(State)
  subgraph_builder.add_node(subgraph_node_1)
  subgraph_builder.add_edge(START, "subgraph_node_1")

  subgraph = subgraph_builder.compile()

  # 父图

  builder = StateGraph(State)
  builder.add_node("node_1", subgraph)
  builder.add_edge(START, "node_1")

  checkpointer = MemorySaver()
  graph = builder.compile(checkpointer=checkpointer)

  config = {"configurable": {"thread_id": "1"}}

  graph.invoke({"foo": ""}, config)
  parent_state = graph.get_state(config)

  # 这将在子图被中断时可用。
  # 一旦您恢复图，将无法访问子图状态。
  subgraph_state = graph.get_state(config, subgraphs=True).tasks[0].state

  # 恢复子图
  graph.invoke(Command(resume="bar"), config)
  ```

  1. 这将在子图被中断时可用。一旦您恢复图，将无法访问子图状态。
</Accordion>

## 流式传输子图输出

要在流式输出中包含子图的输出，可以在父图的stream方法中设置subgraphs选项。这将同时流式传输父图和任何子图的输出。

```python  theme={null}
for chunk in graph.stream(
    {"foo": "foo"},
    subgraphs=True, # [!code highlight]
    stream_mode="updates",
):
    print(chunk)
```

<Accordion title="从子图流式传输">
  ```python  theme={null}
  from typing_extensions import TypedDict
  from langgraph.graph.state import StateGraph, START

  # 定义子图
  class SubgraphState(TypedDict):
      foo: str
      bar: str

  def subgraph_node_1(state: SubgraphState):
      return {"bar": "bar"}

  def subgraph_node_2(state: SubgraphState):
      # 注意这个节点使用了只在子图中可用的状态键（'bar'）
      # 并且在共享状态键（'foo'）上发送更新
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
      subgraphs=True, # [!code highlight]
  ):
      print(chunk)
  ```

  ```
  ((), {'node_1': {'foo': 'hi! foo'}})
  (('node_2:e58e5673-a661-ebb0-70d4-e298a7fc28b7',), {'subgraph_node_1': {'bar': 'bar'}})
  (('node_2:e58e5673-a661-ebb0-70d4-e298a7fc28b7',), {'subgraph_node_2': {'foo': 'hi! foobar'}})
  ((), {'node_2': {'foo': 'hi! foobar'}})
  ```
</Accordion>

***

<Callout icon="pen-to-square" iconType="regular">
  [在GitHub上编辑此页面源代码。](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/use-subgraphs.mdx)
</Callout>

<Tip icon="terminal" iconType="regular">
  [通过MCP以编程方式连接这些文档](/use-these-docs)，实现与Claude、VSCode等的实时问答。
</Tip>