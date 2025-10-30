# 持久化

LangGraph 具有内置的持久化层，通过检查点器(checkpointer)实现。当你使用检查点器编译一个图时，检查点器会在每个超级步骤(super-step)保存图状态的`checkpoint`。这些检查点被保存到一个`线程(thread)`中，可以在图执行后访问。由于`线程`允许在执行后访问图的状态，因此包括人机交互、记忆、时间旅行和容错在内的几种强大功能都成为可能。下面，我们将更详细地讨论这些概念中的每一个。

<img src="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/checkpoints.jpg?fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=966566aaae853ed4d240c2d0d067467c" alt="Checkpoints" data-og-width="2316" width="2316" data-og-height="748" height="748" data-path="oss/images/checkpoints.jpg" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/checkpoints.jpg?w=280&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=7bb8525bfcd22b3903b3209aa7497f47 280w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/checkpoints.jpg?w=560&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=e8d07fc2899b9a13c7b00eb9b259c3c9 560w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/checkpoints.jpg?w=840&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=46a2f9ed3b131a7c78700711e8c314d6 840w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/checkpoints.jpg?w=1100&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=c339bd49757810dad226e1846f066c94 1100w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/checkpoints.jpg?w=1650&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=8333dfdb9d766363f251132f2dfa08a1 1650w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/checkpoints.jpg?w=2500&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=33ba13937eed043ba4a7a87b36d3046f 2500w" />

<Info>
  **LangGraph API 自动处理检查点**
  使用 LangGraph API 时，你无需手动实现或配置检查点器。API 在后台为你处理所有持久化基础设施。
</Info>

## 线程(Thread)

线程是由检查点器为每个保存的检查点分配的唯一 ID 或线程标识符。它包含一系列[运行](/langsmith/assistants#execution)的累积状态。当运行执行时，助手的底层图的[状态](/oss/python/langgraph/graph-api#state)将被持久化到线程中。

使用检查点器调用图时，你必须将`thread_id`作为配置中`configurable`部分的一部分指定。

```python  theme={null}
{"configurable": {"thread_id": "1"}}
```

可以检索线程的当前和历史状态。为了持久化状态，必须在执行运行之前创建线程。LangSmith API 提供了多个用于创建和管理线程及线程状态的端点。更多详情请参见[API 参考](https://langchain-ai.github.io/langgraph/cloud/reference/api/)。

## 检查点(Checkpoint)

线程在特定时间点的状态称为检查点。检查点是在每个超级步骤保存的图状态的快照，由具有以下关键属性的`StateSnapshot`对象表示：

* `config`:与此检查点关联的配置。
* `metadata`:与此检查点关联的元数据。
* `values`:此时状态通道的值。
* `next`:图中下一个要执行的节点名称元组。
* `tasks`:包含关于要执行的任务信息的`PregelTask`对象元组。如果该步骤之前曾尝试执行过，将包含错误信息。如果图在节点内被[动态](/oss/python/langgraph/interrupts#pause-using-interrupt)中断，任务将包含与中断相关的附加数据。

检查点被持久化并可用于在以后恢复线程的状态。

让我们看看当调用如下简单图时会保存哪些检查点：

```python  theme={null}
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import InMemorySaver
from langchain_core.runnables import RunnableConfig
from typing import Annotated
from typing_extensions import TypedDict
from operator import add

class State(TypedDict):
    foo: str
    bar: Annotated[list[str], add]

def node_a(state: State):
    return {"foo": "a", "bar": ["a"]}

def node_b(state: State):
    return {"foo": "b", "bar": ["b"]}


workflow = StateGraph(State)
workflow.add_node(node_a)
workflow.add_node(node_b)
workflow.add_edge(START, "node_a")
workflow.add_edge("node_a", "node_b")
workflow.add_edge("node_b", END)

checkpointer = InMemorySaver()
graph = workflow.compile(checkpointer=checkpointer)

config: RunnableConfig = {"configurable": {"thread_id": "1"}}
graph.invoke({"foo": ""}, config)
```

运行图后，我们期望看到恰好4个检查点：

* 包含[`START`](https://reference.langchain.com/python/langgraph/constants/#langgraph.constants.START)作为下一个要执行的节点的空检查点
* 包含用户输入`{'foo': '', 'bar': []}`和`node_a`作为下一个要执行节点的检查点
* 包含`node_a`的输出`{'foo': 'a', 'bar': ['a']}`和`node_b`作为下一个要执行节点的检查点
* 包含`node_b`的输出`{'foo': 'b', 'bar': ['a', 'b']}`且没有下一个要执行节点的检查点

注意，由于`bar`通道有reducer，其值包含来自两个节点的输出。

### 获取状态

与保存的图状态交互时，你必须指定[线程标识符](#threads)。可以通过调用`graph.get_state(config)`查看图的*最新*状态。这将返回一个`StateSnapshot`对象，对应于配置中提供的线程ID关联的最新检查点，或者如果提供了检查点ID，则对应于该线程的检查点ID关联的检查点。

```python  theme={null}
# 获取最新的状态快照
config = {"configurable": {"thread_id": "1"}}
graph.get_state(config)

# 获取特定checkpoint_id的状态快照
config = {"configurable": {"thread_id": "1", "checkpoint_id": "1ef663ba-28fe-6528-8002-5a559208592c"}}
graph.get_state(config)
```

在我们的示例中，`get_state`的输出如下所示：

```
StateSnapshot(
    values={'foo': 'b', 'bar': ['a', 'b']},
    next=(),
    config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28fe-6528-8002-5a559208592c'}},
    metadata={'source': 'loop', 'writes': {'node_b': {'foo': 'b', 'bar': ['b']}}, 'step': 2},
    created_at='2024-08-29T19:19:38.821749+00:00',
    parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28f9-6ec4-8001-31981c2c39f8'}}, tasks=()
)
```

### 获取状态历史

可以通过调用[`graph.get_state_history(config)`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.CompiledStateGraph.get_state_history)获取给定线程的图执行完整历史。这将返回与配置中提供的线程ID关联的`StateSnapshot`对象列表。重要的是，检查点将按时间顺序排列，最近的检查点/`StateSnapshot`将是列表中的第一个。

```python  theme={null}
config = {"configurable": {"thread_id": "1"}}
list(graph.get_state_history(config))
```

在我们的示例中，[`get_state_history`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.CompiledStateGraph.get_state_history)的输出如下所示：

```
[
    StateSnapshot(
        values={'foo': 'b', 'bar': ['a', 'b']},
        next=(),
        config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28fe-6528-8002-5a559208592c'}},
        metadata={'source': 'loop', 'writes': {'node_b': {'foo': 'b', 'bar': ['b']}}, 'step': 2},
        created_at='2024-08-29T19:19:38.821749+00:00',
        parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28f9-6ec4-8001-31981c2c39f8'}},
        tasks=(),
    ),
    StateSnapshot(
        values={'foo': 'a', 'bar': ['a']},
        next=('node_b',),
        config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28f9-6ec4-8001-31981c2c39f8'}},
        metadata={'source': 'loop', 'writes': {'node_a': {'foo': 'a', 'bar': ['a']}}, 'step': 1},
        created_at='2024-08-29T19:19:38.819946+00:00',
        parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28f4-6b4a-8000-ca575a13d36a'}},
        tasks=(PregelTask(id='6fb7314f-f114-5413-a1f3-d37dfe98ff44', name='node_b', error=None, interrupts=()),),
    ),
    StateSnapshot(
        values={'foo': '', 'bar': []},
        next=('node_a',),
        config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28f4-6b4a-8000-ca575a13d36a'}},
        metadata={'source': 'loop', 'writes': None, 'step': 0},
        created_at='2024-08-29T19:19:38.817813+00:00',
        parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28f0-6c66-bfff-6723431e8481'}},
        tasks=(PregelTask(id='f1b14528-5ee5-579c-949b-23ef9bfbed58', name='node_a', error=None, interrupts=()),),
    ),
    StateSnapshot(
        values={'bar': []},
        next=('__start__',),
        config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28f0-6c66-bfff-6723431e8481'}},
        metadata={'source': 'input', 'writes': {'foo': ''}, 'step': -1},
        created_at='2024-08-29T19:19:38.816205+00:00',
        parent_config=None,
        tasks=(PregelTask(id='6d27aa2e-d72b-5504-a36f-8620e54a76dd', name='__start__', error=None, interrupts=()),),
    )
]
```

<img src="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/get_state.jpg?fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=38ffff52be4d8806b287836295a3c058" alt="State" data-og-width="2692" width="2692" data-og-height="1056" height="1056" data-path="oss/images/get_state.jpg" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/get_state.jpg?w=280&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=e932acac5021614d0eb99b90e54be004 280w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/get_state.jpg?w=560&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=2eaf153fd49ba728e1d679c12bb44b6f 560w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/get_state.jpg?w=840&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=0ac091c7dbe8b1f0acff97615a3683ee 840w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/get_state.jpg?w=1100&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=9921a482f1c4f86316fca23a5150b153 1100w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/get_state.jpg?w=1650&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=9412cd906f6d67a9fe1f50a5d4f4c674 1650w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/get_state.jpg?w=2500&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=ccc5118ed85926bda3715c81ce728fcc 2500w" />

### 重放(Replay)

也可以重放先前的图执行。如果我们使用`thread_id`和`checkpoint_id`来调用图，那么我们将*重放*与`checkpoint_id`对应的检查点之前执行的步骤，并且只执行检查点之后的步骤。

* `thread_id`是线程的ID。
* `checkpoint_id`是引用线程内特定检查点的标识符。

调用图时，必须将它们作为配置`configurable`部分的一部分传递：

```python  theme={null}
config = {"configurable": {"thread_id": "1", "checkpoint_id": "0c62ca34-ac19-445d-bbb0-5b4984975b2a"}}
graph.invoke(None, config=config)
```

重要的是，LangGraph知道特定步骤是否已经执行过。如果已经执行过，LangGraph将简单地*重放*图中的特定步骤，并且不会重新执行该步骤，但仅限于提供的`checkpoint_id`之前的步骤。所有`checkpoint_id`之后的步骤都将被执行（即，一个新的分支），即使它们之前已经执行过。有关重放的更多信息，请参阅此[时间旅行使用指南](/oss/python/langgraph/use-time-travel)。

<img src="https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/re_play.png?fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=d7b34b85c106e55d181ae1f4afb50251" alt="Replay" data-og-width="2276" width="2276" data-og-height="986" height="986" data-path="oss/images/re_play.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/re_play.png?w=280&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=627d1fb4cb0ce3e5734784cc4a841cca 280w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/re_play.png?w=560&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=ab462e9559619778d1bdfced578ee0ba 560w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/re_play.png?w=840&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=7cc304a2a0996e22f783e9a5f7a69f89 840w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/re_play.png?w=1100&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=b322f66ef96d6734dcac38213104f080 1100w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/re_play.png?w=1650&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=922f1b014b33fae4fda1e576d57a9983 1650w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/re_play.png?w=2500&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=efae9196c69a2908846c9d23ad117a90 2500w" />

### 更新状态

除了从特定的`checkpoints`重放图，我们还可以*编辑*图状态。我们使用[`update_state`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.CompiledStateGraph.update_state)来实现。此方法接受三个不同的参数：

#### `config`

配置应包含要更新的线程的`thread_id`。当只传递`thread_id`时，我们更新（或分支）当前状态。或者，如果包含`checkpoint_id`字段，则分支选定的检查点。

#### `values`

这些将用于更新状态的值。注意，此更新被视为与来自节点的任何更新完全相同。这意味着这些值将传递给[reducer](/oss/python/langgraph/graph-api#reducers)函数（如果为图状态中的某些通道定义了）。这意味着[`update_state`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.CompiledStateGraph.update_state)不会自动覆盖每个通道的通道值，而只覆盖没有reducer的通道。让我们通过一个例子来说明。

假设你已使用以下模式定义了图的状态（见上面的完整示例）：

```python  theme={null}
from typing import Annotated
from typing_extensions import TypedDict
from operator import add

class State(TypedDict):
    foo: int
    bar: Annotated[list[str], add]
```

假设图的当前状态是

```
{"foo": 1, "bar": ["a"]}
```

如果你如下更新状态：

```python  theme={null}
graph.update_state(config, {"foo": 2, "bar": ["b"]})
```

那么图的新状态将是：

```
{"foo": 2, "bar": ["a", "b"]}
```

`foo`键（通道）被完全更改（因为该通道没有指定reducer，所以[`update_state`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.CompiledStateGraph.update_state)会覆盖它）。但是，为`bar`键指定了reducer，因此它将`"b"`附加到`bar`的状态。

#### `as_node`

调用[`update_state`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.CompiledStateGraph.update_state)时最后可以选择指定的是`as_node`。如果你提供了它，更新将像来自节点`as_node`一样应用。如果没有提供`as_node`，它将被设置为最后一个更新状态的节点（如果不明确）。重要的是，下一步要执行的节点取决于最后一个给出更新的节点，因此这可用于控制哪个节点执行下一步。有关分支状态的更多信息，请参阅此[时间旅行使用指南](/oss/python/langgraph/use-time-travel)。

<img src="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/checkpoints_full_story.jpg?fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=a52016b2c44b57bd395d6e1eac47aa36" alt="Update" data-og-width="3705" width="3705" data-og-height="2598" height="2598" data-path="oss/images/checkpoints_full_story.jpg" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/checkpoints_full_story.jpg?w=280&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=06de1669d4d62f0e8013c4ffef021437 280w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/checkpoints_full_story.jpg?w=560&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=b149bed4f842c4f179e55247a426befe 560w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/checkpoints_full_story.jpg?w=840&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=58cfc0341a77e179ce443a89d667784c 840w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/checkpoints_full_story.jpg?w=1100&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=29776799d5a22c3aec7d4a45f675ba14 1100w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/checkpoints_full_story.jpg?w=1650&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=5600d9dd7c52dda79e4eb240c344f84a 1650w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/checkpoints_full_story.jpg?w=2500&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=e428c9c4fc060579c0b7fead1d4a54cb 2500w" />

## 内存存储

<img src="https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/shared_state.png?fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=354526fb48c5eb11b4b2684a2df40d6c" alt="Model of shared state" data-og-width="1482" width="1482" data-og-height="777" height="777" data-path="oss/images/shared_state.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/shared_state.png?w=280&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=1965b83f077aea6301b95b59a9a1e318 280w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/shared_state.png?w=560&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=02898a7498e355e04919ac4121678179 560w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/shared_state.png?w=840&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=4ef92e64d1151922511c78afde7abdca 840w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/shared_state.png?w=1100&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=abddd799a170aa9af9145574e46cff6f 1100w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/shared_state.png?w=1650&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=14025324ecb0c462ee1919033d2ae9c5 1650w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/shared_state.png?w=2500&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=a4f7989c4392a7ba8160f559d6fd8942 2500w" />

[状态模式](/oss/python/langgraph/graph-api#schema)指定一组在图执行时填充的键。如上所述，状态可以由检查点器在每个图步骤写入线程，从而实现状态持久化。

但是，如果我们想在*跨线程*中保留一些信息怎么办？考虑聊天机器人的情况，我们希望在与该用户的所有*聊天对话*（即线程）中保留关于该用户的具体信息！

仅使用检查点器，我们无法在线程之间共享信息。这就需要[`Store`](https://python.langchain.com/api_reference/langgraph/index.html#module-langgraph.store)接口。作为说明，我们可以定义一个`InMemoryStore`来跨线程存储有关用户的信息。我们像以前一样使用检查点器编译图，并使用新的`in_memory_store`变量。

<Info>
  **LangGraph API 自动处理存储**
  使用 LangGraph API 时，你无需手动实现或配置存储。API 在后台为你处理所有存储基础设施。
</Info>

### 基本用法

首先，让我们在没有使用 LangGraph 的情况下单独展示这一点。

```python  theme={null}
from langgraph.store.memory import InMemoryStore
in_memory_store = InMemoryStore()
```

记忆通过`tuple`命名空间化，在此特定示例中将是`(<user_id>, "memories")`。命名空间可以是任何长度并表示任何内容，不必特定于用户。

```python  theme={null}
user_id = "1"
namespace_for_memory = (user_id, "memories")
```

我们使用`store.put`方法将记忆保存到我们在存储中定义的命名空间中。这样做时，我们指定如上定义的命名空间，以及记忆的键值对：键只是记忆的唯一标识符(`memory_id`)，值（字典）是记忆本身。

```python  theme={null}
memory_id = str(uuid.uuid4())
memory = {"food_preference" : "I like pizza"}
in_memory_store.put(namespace_for_memory, memory_id, memory)
```

我们可以使用`store.search`方法读取我们命名空间中的记忆，这将返回给定用户的所有记忆列表。最新的记忆是列表中的最后一个。

```python  theme={null}
memories = in_memory_store.search(namespace_for_memory)
memories[-1].dict()
{'value': {'food_preference': 'I like pizza'},
 'key': '07e0caf4-1631-47b7-b15f-65515d4c1843',
 'namespace': ['1', 'memories'],
 'created_at': '2024-10-02T17:22:31.590602+00:00',
 'updated_at': '2024-10-02T17:22:31.590605+00:00'}
```

每种记忆类型都是一个具有某些属性的Python类([`Item`](https://langchain-ai.github.io/langgraph/reference/store/#langgraph.store.base.Item))。我们可以通过像上面那样通过`.dict`转换来将其作为字典访问。

它具有的属性是：

* `value`: 此记忆的值（本身是字典）
* `key`: 此记忆在此命名空间中的唯一键
* `namespace`: 字符串列表，此记忆类型的命名空间
* `created_at`: 创建此记忆的时间戳
* `updated_at`: 更新此记忆的时间戳

### 语义搜索

除了简单的检索，存储还支持语义搜索，允许你根据含义而非精确匹配来找到记忆。为此，使用嵌入模型配置存储：

```python  theme={null}
from langchain.embeddings import init_embeddings

store = InMemoryStore(
    index={
        "embed": init_embeddings("openai:text-embedding-3-small"),  # 嵌入提供者
        "dims": 1536,                              # 嵌入维度
        "fields": ["food_preference", "$"]              # 要嵌入的字段
    }
)
```

现在搜索时，你可以使用自然语言查询来查找相关记忆：

```python  theme={null}
# 查找关于食物偏好的记忆
# (这可以在将记忆放入存储后完成)
memories = store.search(
    namespace_for_memory,
    query="What does the user like to eat?",
    limit=3  # 返回前3个匹配项
)
```

你可以通过配置`fields`参数或在存储记忆时指定`index`参数来控制记忆的哪些部分被嵌入：

```python  theme={null}
# 存储要嵌入特定字段
store.put(
    namespace_for_memory,
    str(uuid.uuid4()),
    {
        "food_preference": "I love Italian cuisine",
        "context": "Discussing dinner plans"
    },
    index=["food_preference"]  # 仅嵌入"food_preferences"字段
)

# 存储时不嵌入（仍然可检索，但不可搜索）
store.put(
    namespace_for_memory,
    str(uuid.uuid4()),
    {"system_info": "Last updated: 2024-01-01"},
    index=False
)
```

### 在 LangGraph 中使用

有了这一切，我们在 LangGraph 中使用`in_memory_store`。`in_memory_store`与检查点器协同工作：检查点器如上所述将状态保存到线程中，而`in_memory_store`允许我们存储任意信息以供*跨*线程访问。我们按如下方式使用检查点和`in_memory_store`编译图：

```python  theme={null}
from langgraph.checkpoint.memory import InMemorySaver

# 我们需要这个，因为我们希望启用线程（对话）
checkpointer = InMemorySaver()

# ... 定义图 ...

# 使用检查点和存储编译图
graph = graph.compile(checkpointer=checkpointer, store=in_memory_store)
```

我们如前所述使用`thread_id`调用图，还使用`user_id`，我们将用它来命名空间化我们展示给这个特定用户的记忆。

```python  theme={null}
# 调用图
user_id = "1"
config = {"configurable": {"thread_id": "1", "user_id": user_id}}

# 首先只是对AI说声你好
for update in graph.stream(
    {"messages": [{"role": "user", "content": "hi"}]}, config, stream_mode="updates"
):
    print(update)
```

我们可以在*任何节点*中通过传递`store: BaseStore`和`config: RunnableConfig`作为节点参数来访问`in_memory_store`和`user_id`。以下是我们在节点中使用语义搜索来查找相关记忆的方法：

```python  theme={null}
def update_memory(state: MessagesState, config: RunnableConfig, *, store: BaseStore):

    # 从配置中获取用户ID
    user_id = config["configurable"]["user_id"]

    # 命名空间化记忆
    namespace = (user_id, "memories")

    # ... 分析对话并创建新记忆

    # 创建新的记忆ID
    memory_id = str(uuid.uuid4())

    # 我们创建一个新记忆
    store.put(namespace, memory_id, {"memory": memory})

```

如上所述，我们还可以在任何节点中访问存储并使用`store.search`方法获取记忆。回想一下，记忆作为对象列表返回，可以转换为字典。

```python  theme={null}
memories[-1].dict()
{'value': {'food_preference': 'I like pizza'},
 'key': '07e0caf4-1631-47b7-b15f-65515d4c1843',
 'namespace': ['1', 'memories'],
 'created_at': '2024-10-02T17:22:31.590602+00:00',
 'updated_at': '2024-10-02T17:22:31.590605+00:00'}
```

我们可以访问记忆并在模型调用中使用它们。

```python  theme={null}
def call_model(state: MessagesState, config: RunnableConfig, *, store: BaseStore):
    # 从配置中获取用户ID
    user_id = config["configurable"]["user_id"]

    # 命名空间化记忆
    namespace = (user_id, "memories")

    # 根据最新消息进行搜索
    memories = store.search(
        namespace,
        query=state["messages"][-1].content,
        limit=3
    )
    info = "\n".join([d.value["memory"] for d in memories])

    # ... 在模型调用中使用记忆
```

如果我们创建一个新线程，只要`user_id`相同，我们仍然可以访问相同的记忆。

```python  theme={null}
# 调用图
config = {"configurable": {"thread_id": "2", "user_id": "1"}}

# 再次说你好
for update in graph.stream(
    {"messages": [{"role": "user", "content": "hi, tell me about my memories"}]}, config, stream_mode="updates"
):
    print(update)
```

当我们使用 LangSmith 时，无论是本地（例如，在[Studio](/langsmith/studio)中）还是[通过 LangSmith 托管](/langsmith/hosting)，基础存储默认可用，并且在图编译期间无需指定。但是，为了启用语义搜索，你确实需要在`langgraph.json`文件中配置索引设置。例如：

```json  theme={null}
{
    ...
    "store": {
        "index": {
            "embed": "openai:text-embeddings-3-small",
            "dims": 1536,
            "fields": ["$"]
        }
    }
}
```

更多详情和配置选项，请参见[部署指南](/langsmith/semantic-search)。

## 检查点器库

在幕后，检查点是由符合[`BaseCheckpointSaver`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.base.BaseCheckpointSaver)接口的检查点对象驱动的。LangGraph 提供了几个检查点实现，都是通过独立的可安装库实现的：

* `langgraph-checkpoint`: 检查点保存器的基本接口（[`BaseCheckpointSaver`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.base.BaseCheckpointSaver)）和序列化/反序列化接口（[`SerializerProtocol`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.serde.base.SerializerProtocol)）。包括用于实验的内存检查点实现（[`InMemorySaver`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.memory.InMemorySaver)）。LangGraph 附带了`langgraph-checkpoint`。
* `langgraph-checkpoint-sqlite`: 使用 SQLite 数据库的 LangGraph 检查点实现（[`SqliteSaver`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.sqlite.SqliteSaver) / [`AsyncSqliteSaver`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.sqlite.aio.AsyncSqliteSaver)）。适用于实验和本地工作流。需要单独安装。
* `langgraph-checkpoint-postgres`: 使用 Postgres 数据库的高级检查点（[`PostgresSaver`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.postgres.PostgresSaver) / [`AsyncPostgresSaver`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.postgres.aio.AsyncPostgresSaver)），在 LangSmith 中使用。适用于生产环境使用。需要单独安装。

### 检查点器接口

每个检查点都符合[`BaseCheckpointSaver`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.base.BaseCheckpointSaver)接口并实现以下方法：

* `.put` - 存储带有其配置和元数据的检查点。
* `.put_writes` - 存储与检查点关联的中间写入（即[待处理写入](#pending-writes)）。
* `.get_tuple` - 使用给定配置（`thread_id`和`checkpoint_id`）获取检查点元组。这用于在`graph.get_state()`中填充`StateSnapshot`。
* `.list` - 列出与给定配置和过滤条件匹配的检查点。这用于在`graph.get_state_history()`中填充状态历史。

如果检查点器与异步图执行一起使用（即通过`.ainvoke`、`.astream`、`.abatch`执行图），将使用上述方法的异步版本（`.aput`、`.aput_writes`、`.aget_tuple`、`.alist`）。

<Note>
  要异步运行你的图，你可以使用[`InMemorySaver`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.memory.InMemorySaver)，或 Sqlite/Postgres 检查器的异步版本 -- [`AsyncSqliteSaver`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.sqlite.aio.AsyncSqliteSaver) / [`AsyncPostgresSaver`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.postgres.aio.AsyncPostgresSaver)检查器。
</Note>

### 序列化器

当检查点器保存图状态时，它们需要序列化状态中的通道值。这是使用序列化器对象完成的。

`langgraph_checkpoint`定义了实现序列化器的[协议](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.serde.base.SerializerProtocol)，并提供了默认实现（[`JsonPlusSerializer`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.serde.jsonplus.JsonPlusSerializer)），它可以处理各种类型，包括 LangChain 和 LangGraph 原语、日期时间、枚举等。

#### 使用`pickle`进行序列化

默认序列化器[`JsonPlusSerializer`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.serde.jsonplus.JsonPlusSerializer)在后台使用 ormsgpack 和 JSON，这不适用于所有类型的对象。

如果你想回退到 pickle 以支持我们 msgpack 编码器当前不支持的类型（如 Pandas 数据框），
你可以使用[`JsonPlusSerializer`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.serde.jsonplus.JsonPlusSerializer)的`pickle_fallback`参数：

```python  theme={null}
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.checkpoint.serde.jsonplus import JsonPlusSerializer

# ... 定义图 ...
graph.compile(
    checkpointer=InMemorySaver(serde=JsonPlusSerializer(pickle_fallback=True))
)
```

#### 加密

检查点器可以选择加密所有持久化状态。为此，将[`EncryptedSerializer`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.serde.encrypted.EncryptedSerializer)的实例传递给任何[`BaseCheckpointSaver`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.base.BaseCheckpointSaver)实现的`serde`参数。创建加密序列化器最简单的方法是通过[`from_pycryptodome_aes`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.serde.encrypted.EncryptedSerializer.from_pycryptodome_aes)，它从`LANGGRAPH_AES_KEY`环境变量读取 AES 密钥（或接受`key`参数）：

```python  theme={null}
import sqlite3

from langgraph.checkpoint.serde.encrypted import EncryptedSerializer
from langgraph.checkpoint.sqlite import SqliteSaver

serde = EncryptedSerializer.from_pycryptodome_aes()  # 读取 LANGGRAPH_AES_KEY
checkpointer = SqliteSaver(sqlite3.connect("checkpoint.db"), serde=serde)
```

```python  theme={null}
from langgraph.checkpoint.serde.encrypted import EncryptedSerializer
from langgraph.checkpoint.postgres import PostgresSaver

serde = EncryptedSerializer.from_pycryptodome_aes()
checkpointer = PostgresSaver.from_conn_string("postgresql://...", serde=serde)
checkpointer.setup()
```

在 LangSmith 上运行时，只要存在`LANGGRAPH_AES_KEY`，就会自动启用加密，所以你只需要提供环境变量。可以通过实现[`CipherProtocol`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.serde.base.CipherProtocol)并将其提供给[`EncryptedSerializer`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.serde.encrypted.EncryptedSerializer)来使用其他加密方案。

## 能力

### 人机交互

首先，检查点器通过允许人类检查、中断和批准图步骤来促进[人机工作流](/oss/python/langgraph/interrupts)。这些工作流需要检查点器，因为人类必须能够随时查看图的任何时间点的状态，并且图必须能够在人类对状态进行任何更新后恢复执行。有关示例，请参见[使用指南](/oss/python/langgraph/interrupts)。

### 记忆

其次，检查点器允许在交互之间实现["记忆"](/oss/python/concepts/memory)。在重复的人类交互（如对话）中，任何后续消息都可以发送到该线程，该线程将保留对之前消息的记忆。有关如何使用检查点器添加和管理对话记忆的信息，请参见[添加记忆](/oss/python/langgraph/add-memory)。

### 时间旅行

第三，检查点器允许实现["时间旅行"](/oss/python/langgraph/use-time-travel)，允许用户重放先前的图执行以审查和/或调试特定图步骤。此外，检查点器使得能够在任意检查点分支图状态以探索替代轨迹成为可能。

### 容错

最后，检查点还提供容错和错误恢复：如果一个或多个节点在给定的超级步骤中失败，你可以从最后一个成功的步骤重新启动图。此外，当图节点在给定的超级步骤执行过程中失败时，LangGraph 存储在该超级步骤成功完成的任何其他节点的待处理检查点写入，这样无论我们何时从该超级步骤恢复图执行，我们都不会重新运行成功的节点。

#### 待处理写入

此外，当图节点在给定的超级步骤执行过程中失败时，LangGraph 存储在该超级步骤成功完成的任何其他节点的待处理检查点写入，这样无论我们何时从该超级步骤恢复图执行，我们都不会重新运行成功的节点。

***

<Callout icon="pen-to-square" iconType="regular">
  [在GitHub上编辑此页面的源代码。](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/persistence.mdx)
</Callout>

<Tip icon="terminal" iconType="regular">
  [通过MCP以编程方式连接这些文档](/use-these-docs)到Claude、VSCode等，以获取实时答案。
</Tip>