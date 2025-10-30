# 测试

在完成 LangGraph 代理的原型设计后，一个自然的下一步是添加测试。本指南介绍了一些在编写单元测试时可以使用的有用模式。

请注意，本指南是特定于 LangGraph 的，涵盖了具有自定义结构的图形场景 - 如果您是初学者，请查看[此部分](/oss/python/langchain/test/)，该部分使用 LangChain 内置的[`create_agent`](https://reference.langchain.com/python/langchain/agents/#langchain.agents.create_agent)。

## 先决条件

首先，确保您已安装[`pytest`](https://docs.pytest.org/)：

```bash  theme={null}
$ pip install -U pytest
```

## 入门指南

由于许多 LangGraph 代理依赖于状态，一个有用的模式是在每个使用图形的测试之前创建图形，然后在测试中使用新的检查点实例编译它。

下面的示例展示了如何使用一个简单的线性图形来实现这一点，该图形通过`node1`和`node2`逐步执行。每个节点都更新单个状态键`my_key`：

```python  theme={null}
import pytest

from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import MemorySaver

def create_graph() -> StateGraph:
    class MyState(TypedDict):
        my_key: str

    graph = StateGraph(MyState)
    graph.add_node("node1", lambda state: {"my_key": "hello from node1"})
    graph.add_node("node2", lambda state: {"my_key": "hello from node2"})
    graph.add_edge(START, "node1")
    graph.add_edge("node1", "node2")
    graph.add_edge("node2", END)
    return graph

def test_basic_agent_execution() -> None:
    checkpointer = MemorySaver()
    graph = create_graph()
    compiled_graph = graph.compile(checkpointer=checkpointer)
    result = compiled_graph.invoke(
        {"my_key": "initial_value"},
        config={"configurable": {"thread_id": "1"}}
    )
    assert result["my_key"] == "hello from node2"
```

## 测试单个节点和边

编译后的 LangGraph 代理将每个单独节点的引用公开为`graph.nodes`。您可以利用这一点来测试代理中的单个节点。请注意，这将绕过编译图形时传递的任何检查点：

```python  theme={null}
import pytest

from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import MemorySaver

def create_graph() -> StateGraph:
    class MyState(TypedDict):
        my_key: str

    graph = StateGraph(MyState)
    graph.add_node("node1", lambda state: {"my_key": "hello from node1"})
    graph.add_node("node2", lambda state: {"my_key": "hello from node2"})
    graph.add_edge(START, "node1")
    graph.add_edge("node1", "node2")
    graph.add_edge("node2", END)
    return graph

def test_individual_node_execution() -> None:
    # 在此示例中将被忽略
    checkpointer = MemorySaver()
    graph = create_graph()
    compiled_graph = graph.compile(checkpointer=checkpointer)
    # 仅调用节点 1
    result = compiled_graph.nodes["node1"].invoke(
        {"my_key": "initial_value"},
    )
    assert result["my_key"] == "hello from node1"
```

## 部分执行

对于由较大图形组成的代理，您可能希望测试代理内的部分执行路径，而不是整个端到端流程。在某些情况下，将这些部分[重组为子图](/oss/python/langgraph/use-subgraphs)在语义上可能更有意义，您可以像平常一样单独调用它们。

但是，如果您不希望更改代理图形的整体结构，可以使用 LangGraph 的持久性机制来模拟一种状态，即代理在所需部分开始之前暂停，并在所需部分结束后再次暂停。步骤如下：

1. 使用检查点编译代理（内存检查点[`InMemorySaver`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.memory.InMemorySaver)足以用于测试）。
2. 使用[`as_node`](/oss/python/langgraph/persistence#as-node)参数调用代理的[`update_state`](/oss/python/langgraph/use-time-travel)方法，该参数设置为您想要开始测试的节点*之前*的节点名称。
3. 使用您更新状态时使用的相同`thread_id`和设置为您想要停止的节点名称的`interrupt_after`参数调用您的代理。

下面是一个仅执行线性图形中第二个和第三个节点的示例：

```python  theme={null}
import pytest

from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import MemorySaver

def create_graph() -> StateGraph:
    class MyState(TypedDict):
        my_key: str

    graph = StateGraph(MyState)
    graph.add_node("node1", lambda state: {"my_key": "hello from node1"})
    graph.add_node("node2", lambda state: {"my_key": "hello from node2"})
    graph.add_node("node3", lambda state: {"my_key": "hello from node3"})
    graph.add_node("node4", lambda state: {"my_key": "hello from node4"})
    graph.add_edge(START, "node1")
    graph.add_edge("node1", "node2")
    graph.add_edge("node2", "node3")
    graph.add_edge("node3", "node4")
    graph.add_edge("node4", END)
    return graph

def test_partial_execution_from_node2_to_node3() -> None:
    checkpointer = MemorySaver()
    graph = create_graph()
    compiled_graph = graph.compile(checkpointer=checkpointer)
    compiled_graph.update_state(
        config={
          "configurable": {
            "thread_id": "1"
          }
        },
        # 传递给节点 2 的状态 - 模拟节点 1 结束时的状态
        values={"my_key": "initial_value"},
        # 更新保存的状态，就像它来自节点 1 一样
        # 执行将在节点 2 处恢复
        as_node="node1",
    )
    result = compiled_graph.invoke(
        # 通过传递 None 恢复执行
        None,
        config={"configurable": {"thread_id": "1"}},
        # 在节点 3 之后停止，以便节点 4 不运行
        interrupt_after="node3",
    )
    assert result["my_key"] == "hello from node3"
```

***

<Callout icon="pen-to-square" iconType="regular">
  [在 GitHub 上编辑此页面的源代码。](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/test.mdx)
</Callout>

<Tip icon="terminal" iconType="regular">
  [通过 MCP 以编程方式连接这些文档](/use-these-docs)到 Claude、VSCode 等，以获得实时答案。
</Tip>