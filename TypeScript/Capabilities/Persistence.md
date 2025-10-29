# 持久性

LangGraph 内置了一个持久化层，通过检查点(checkpointers)实现。当你使用检查点编译图时，检查点会在每个超步骤(super-step)保存图状态的`checkpoint`。这些检查点被保存到一个`线程(thread)`中，可以在图执行后访问。由于`线程`允许在执行后访问图的状态，因此包括人在回路中、记忆、时间旅行和容错在内的几种强大功能成为可能。下面，我们将详细讨论这些概念中的每一个。

<img src="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/checkpoints.jpg?fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=966566aaae853ed4d240c2d0d067467c" alt="Checkpoints" data-og-width="2316" width="2316" data-og-height="748" height="748" data-path="oss/images/checkpoints.jpg" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/checkpoints.jpg?w=280&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=7bb8525bfcd22b3903b3209aa7497f47 280w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/checkpoints.jpg?w=560&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=e8d07fc2899b9a13c7b00eb9b259c3c9 560w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/checkpoints.jpg?w=840&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=46a2f9ed3b131a7c78700711e8c314d6 840w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/checkpoints.jpg?w=1100&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=c339bd49757810dad226e1846f066c94 1100w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/checkpoints.jpg?w=1650&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=8333dfdb9d766363f251132f2dfa08a1 1650w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/checkpoints.jpg?w=2500&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=33ba13937eed043ba4a7a87b36d3046f 2500w" />

<Info>
  **LangGraph API 自动处理检查点**
  使用 LangGraph API 时，无需手动实现或配置检查点。API 在后台为您处理所有持久化基础设施。
</Info>

## 线程(Thread)

线程是由检查点为每个保存的检查点分配的唯一 ID 或线程标识符。它包含一系列[运行](/langsmith/assistants#execution)的累积状态。当运行执行时，助手的底层图的[状态](/oss/javascript/langgraph/graph-api#state)将被持久化到线程中。

当使用检查点调用图时，您**必须**在配置的`configurable`部分中指定一个`thread_id`。

```typescript  theme={null}
{
  configurable: {
    thread_id: "1";
  }
}
```

可以检索线程的当前和历史状态。为了持久化状态，必须在执行运行之前创建线程。LangSmith API 提供了多个用于创建和管理线程及线程状态的端点。更多详情请参见[API 参考](https://langchain-ai.github.io/langgraph/cloud/reference/api/)。

## 检查点(Checkpoint)

线程在特定时间点的状态称为检查点。检查点是在每个超步骤保存的图状态的快照，并由具有以下关键属性的`StateSnapshot`对象表示：

* `config`: 与此检查点关联的配置。
* `metadata`: 与此检查点关联的元数据。
* `values`: 此时间点上状态通道的值。
* `next`: 图中下一个要执行的节点名称元组。
* `tasks`: `PregelTask`对象的元组，包含要执行的下一步任务的信息。如果步骤之前曾尝试过，将包含错误信息。如果图在节点内被[动态](/oss/javascript/langgraph/interrupts#pause-using-interrupt)中断，任务将包含与中断相关的附加数据。

检查点会被持久化，并可用于稍后恢复线程的状态。

让我们看看调用简单图时会保存哪些检查点：

```typescript  theme={null}
import { StateGraph, START, END, MemoryServer } from "@langchain/langgraph";
import { registry } from "@langchain/langgraph/zod";
import * as z from "zod";

const State = z.object({
  foo: z.string(),
  bar: z.array(z.string()).register(registry, {
    reducer: {
      fn: (x, y) => x.concat(y),
    },
    default: () => [] as string[],
  }),
});

const workflow = new StateGraph(State)
  .addNode("nodeA", (state) => {
    return { foo: "a", bar: ["a"] };
  })
  .addNode("nodeB", (state) => {
    return { foo: "b", bar: ["b"] };
  })
  .addEdge(START, "nodeA")
  .addEdge("nodeA", "nodeB")
  .addEdge("nodeB", END);

const checkpointer = new MemorySaver();
const graph = workflow.compile({ checkpointer });

const config = { configurable: { thread_id: "1" } };
await graph.invoke({ foo: "" }, config);
```

运行图后，我们期望看到恰好 4 个检查点：

* 具有用户输入`{'foo': '', 'bar': []}`和`nodeA`作为下一个要执行的节点的检查点
* 具有用户输入`{'foo': '', 'bar': []}`和`nodeA`作为下一个要执行的节点的检查点
* 具有`nodeA`的输出`{'foo': 'a', bar: ['a']}`和`nodeB`作为下一个要执行的节点的检查点
* 具有`nodeB`的输出`{'foo': 'b', bar: ['a', 'b']}`和没有下一个要执行的节点的检查点

注意，由于我们对`bar`通道有 reducer，`bar`通道值包含来自两个节点的输出。

### 获取状态

在处理保存的图状态时，您**必须**指定[线程标识符](#threads)。您可以通过调用`graph.getState(config)`查看图的*最新*状态。这将返回一个`StateSnapshot`对象，对应于配置中提供的线程 ID 关联的最新检查点，或者如果提供了检查点 ID，则对应于该线程的检查点。

```typescript  theme={null}
// 获取最新的状态快照
const config = { configurable: { thread_id: "1" } };
await graph.getState(config);

// 获取特定checkpoint_id的状态快照
const config = {
  configurable: {
    thread_id: "1",
    checkpoint_id: "1ef663ba-28fe-6528-8002-5a559208592c",
  },
};
await graph.getState(config);
```

在我们的示例中，`getState`的输出将如下所示：

```
StateSnapshot {
  values: { foo: 'b', bar: ['a', 'b'] },
  next: [],
  config: {
    configurable: {
      thread_id: '1',
      checkpoint_ns: '',
      checkpoint_id: '1ef663ba-28fe-6528-8002-5a559208592c'
    }
  },
  metadata: {
    source: 'loop',
    writes: { nodeB: { foo: 'b', bar: ['b'] } },
    step: 2
  },
  createdAt: '2024-08-29T19:19:38.821749+00:00',
  parentConfig: {
    configurable: {
      thread_id: '1',
      checkpoint_ns: '',
      checkpoint_id: '1ef663ba-28f9-6ec4-8001-31981c2c39f8'
    }
  },
  tasks: []
}
```

### 获取状态历史

您可以通过调用`graph.getStateHistory(config)`获取给定线程的图执行完整历史。这将返回与配置中提供的线程 ID 关联的`StateSnapshot`对象列表。重要的是，检查点将按时间顺序排列，最近的检查点/`StateSnapshot`是列表中的第一个。

```typescript  theme={null}
const config = { configurable: { thread_id: "1" } };
for await (const state of graph.getStateHistory(config)) {
  console.log(state);
}
```

在我们的示例中，`getStateHistory`的输出将如下所示：

```
[
  StateSnapshot {
    values: { foo: 'b', bar: ['a', 'b'] },
    next: [],
    config: {
      configurable: {
        thread_id: '1',
        checkpoint_ns: '',
        checkpoint_id: '1ef663ba-28fe-6528-8002-5a559208592c'
      }
    },
    metadata: {
      source: 'loop',
      writes: { nodeB: { foo: 'b', bar: ['b'] } },
      step: 2
    },
    createdAt: '2024-08-29T19:19:38.821749+00:00',
    parentConfig: {
      configurable: {
        thread_id: '1',
        checkpoint_ns: '',
        checkpoint_id: '1ef663ba-28f9-6ec4-8001-31981c2c39f8'
      }
    },
    tasks: []
  },
  StateSnapshot {
    values: { foo: 'a', bar: ['a'] },
    next: ['nodeB'],
    config: {
      configurable: {
        thread_id: '1',
        checkpoint_ns: '',
        checkpoint_id: '1ef663ba-28f9-6ec4-8001-31981c2c39f8'
      }
    },
    metadata: {
      source: 'loop',
      writes: { nodeA: { foo: 'a', bar: ['a'] } },
      step: 1
    },
    createdAt: '2024-08-29T19:19:38.819946+00:00',
    parentConfig: {
      configurable: {
        thread_id: '1',
        checkpoint_ns: '',
        checkpoint_id: '1ef663ba-28f4-6b4a-8000-ca575a13d36a'
      }
    },
    tasks: [
      PregelTask {
        id: '6fb7314f-f114-5413-a1f3-d37dfe98ff44',
        name: 'nodeB',
        error: null,
        interrupts: []
      }
    ]
  },
  StateSnapshot {
    values: { foo: '', bar: [] },
    next: ['node_a'],
    config: {
      configurable: {
        thread_id: '1',
        checkpoint_ns: '',
        checkpoint_id: '1ef663ba-28f4-6b4a-8000-ca575a13d36a'
      }
    },
    metadata: {
      source: 'loop',
      writes: null,
      step: 0
    },
    createdAt: '2024-08-29T19:19:38.817813+00:00',
    parentConfig: {
      configurable: {
        thread_id: '1',
        checkpoint_ns: '',
        checkpoint_id: '1ef663ba-28f0-6c66-bfff-6723431e8481'
      }
    },
    tasks: [
      PregelTask {
        id: 'f1b14528-5ee5-579c-949b-23ef9bfbed58',
        name: 'node_a',
        error: null,
        interrupts: []
      }
    ]
  },
  StateSnapshot {
    values: { bar: [] },
    next: ['__start__'],
    config: {
      configurable: {
        thread_id: '1',
        checkpoint_ns: '',
        checkpoint_id: '1ef663ba-28f0-6c66-bfff-6723431e8481'
      }
    },
    metadata: {
      source: 'input',
      writes: { foo: '' },
      step: -1
    },
    createdAt: '2024-08-29T19:19:38.816205+00:00',
    parentConfig: null,
    tasks: [
      PregelTask {
        id: '6d27aa2e-d72b-5504-a36f-8620e54a76dd',
        name: '__start__',
        error: null,
        interrupts: []
      }
    ]
  }
]
```

<img src="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/get_state.jpg?fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=38ffff52be4d8806b287836295a3c058" alt="State" data-og-width="2692" width="2692" data-og-height="1056" height="1056" data-path="oss/images/get_state.jpg" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/get_state.jpg?w=280&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=e932acac5021614d0eb99b90e54be004 280w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/get_state.jpg?w=560&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=2eaf153fd49ba728e1d679c12bb44b6f 560w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/get_state.jpg?w=840&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=0ac091c7dbe8b1f0acff97615a3683ee 840w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/get_state.jpg?w=1100&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=9921a482f1c4f86316fca23a5150b153 1100w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/get_state.jpg?w=1650&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=9412cd906f6d67a9fe1f50a5d4f4c674 1650w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/get_state.jpg?w=2500&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=ccc5118ed85926bda3715c81ce728fcc 2500w" />

### 重放(Replay)

也可以回放先前的图执行。如果我们使用`thread_id`和`checkpoint_id`调用图，那么我们将*重放*对应于`checkpoint_id`的检查点之前已执行的步骤，并且只执行该检查点之后的步骤。

* `thread_id`是线程的ID。
* `checkpoint_id`是指向线程中特定检查点的标识符。

调用图时，您必须将这些作为配置的`configurable`部分传递：

```typescript  theme={null}
const config = {
  configurable: {
    thread_id: "1",
    checkpoint_id: "0c62ca34-ac19-445d-bbb0-5b4984975b2a",
  },
};
await graph.invoke(null, config);
```

重要的是，LangGraph 知道特定步骤是否已先前执行。如果已执行，LangGraph 只是在图中*重放*该特定步骤，而不重新执行该步骤，但仅针对提供的`checkpoint_id`之前的步骤。所有`checkpoint_id`之后的步骤都将被执行（即，一个新的分支），即使它们先前已被执行。有关重放的更多信息，请参阅此[时间旅行指南](/oss/javascript/langgraph/use-time-travel)。

<img src="https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/re_play.png?fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=d7b34b85c106e55d181ae1f4afb50251" alt="Replay" data-og-width="2276" width="2276" data-og-height="986" height="986" data-path="oss/images/re_play.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/re_play.png?w=280&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=627d1fb4cb0ce3e5734784cc4a841cca 280w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/re_play.png?w=560&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=ab462e9559619778d1bdfced578ee0ba 560w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/re_play.png?w=840&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=7cc304a2a0996e22f783e9a5f7a69f89 840w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/re_play.png?w=1100&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=b322f66ef96d6734dcac38213104f080 1100w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/re_play.png?w=1650&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=922f1b014b33fae4fda1e576d57a9983 1650w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/re_play.png?w=2500&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=efae9196c69a2908846c9d23ad117a90 2500w" />

### 更新状态

除了从特定`checkpoints`重放图，我们还可以*编辑*图状态。我们使用`graph.updateState()`来实现。此方法接受三个不同的参数：

#### `config`

配置应包含要更新的线程的`thread_id`。当只传递`thread_id`时，我们更新（或分支）当前状态。可选地，如果我们包含`checkpoint_id`字段，则我们分支选定的检查点。

#### `values`

这些将用于更新状态的值。注意，此更新被视为来自节点的任何更新一样处理。这意味着这些值将被传递给[reducer](/oss/javascript/langgraph/graph-api#reducers)函数（如果为图状态中的某些通道定义了）。这意味着[`update_state`](https://langchain-ai.github.io/langgraphjs/reference/classes/langgraph.CompiledStateGraph.html#updateState)不会自动覆盖每个通道的通道值，而仅覆盖没有 reducer 的通道。让我们通过一个例子来说明。

假设您已使用以下模式定义了图的状态（见完整示例）：

```typescript  theme={null}
import { registry } from "@langchain/langgraph/zod";
import * as z from "zod";

const State = z.object({
  foo: z.number(),
  bar: z.array(z.string()).register(registry, {
    reducer: {
      fn: (x, y) => x.concat(y),
    },
    default: () => [] as string[],
  }),
});
```

现在假设图的当前状态是

```typescript  theme={null}
{ foo: 1, bar: ["a"] }
```

如果您如下更新状态：

```typescript  theme={null}
await graph.updateState(config, { foo: 2, bar: ["b"] });
```

那么图的新状态将是：

```typescript  theme={null}
{ foo: 2, bar: ["a", "b"] }
```

`foo`键（通道）完全更改（因为没有为该通道指定 reducer，所以`updateState`会覆盖它）。但是，为`bar`键指定了 reducer，因此它将`"b"`追加到`bar`的状态。

#### `as_node`

调用`updateState`时最后可以选择指定的`as_node`。如果您提供它，更新将来自节点`as_node`应用。如果未提供`as_node`，它将被设置为最后更新状态的节点（如果不模糊）。之所以重要，是因为下一步要执行的步骤取决于最后一个给出更新的节点，因此这可用于控制哪个节点执行下一步。有关分支状态的更多信息，请参阅此[时间旅行指南](/oss/javascript/langgraph/use-time-travel)。

<img src="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/checkpoints_full_story.jpg?fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=a52016b2c44b57bd395d6e1eac47aa36" alt="Update" data-og-width="3705" width="3705" data-og-height="2598" height="2598" data-path="oss/images/checkpoints_full_story.jpg" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/checkpoints_full_story.jpg?w=280&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=06de1669d4d62f0e8013c4ffef021437 280w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/checkpoints_full_story.jpg?w=560&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=b149bed4f842c4f179e55247a426befe 560w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/checkpoints_full_story.jpg?w=840&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=58cfc0341a77e179ce443a89d667784c 840w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/checkpoints_full_story.jpg?w=1100&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=29776799d5a22c3aec7d4a45f675ba14 1100w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/checkpoints_full_story.jpg?w=1650&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=5600d9dd7c52dda79e4eb240c344f84a 1650w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/checkpoints_full_story.jpg?w=2500&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=e428c9c4fc060579c0b7fead1d4a54cb 2500w" />

## 内存存储(Memory Store)

<img src="https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/shared_state.png?fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=354526fb48c5eb11b4b2684a2df40d6c" alt="Model of shared state" data-og-width="1482" width="1482" data-og-height="777" height="777" data-path="oss/images/shared_state.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/shared_state.png?w=280&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=1965b83f077aea6301b95b59a9a1e318 280w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/shared_state.png?w=560&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=02898a7498e355e04919ac4121678179 560w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/shared_state.png?w=840&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=4ef92e64d1151922511c78afde7abdca 840w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/shared_state.png?w=1100&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=abddd799a170aa9af9145574e46cff6f 1100w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/shared_state.png?w=1650&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=14025324ecb0c462ee1919033d2ae9c5 1650w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/shared_state.png?w=2500&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=a4f7989c4392a7ba8160f559d6fd8942 2500w" />

[状态模式](/oss/javascript/langgraph/graph-api#schema)指定一组键，这些键在图执行时被填充。如上所述，状态可以由检查点在图的每个步骤写入线程，实现状态持久化。

但是，如果我们想保留一些*跨线程*的信息怎么办？考虑聊天机器人的情况，我们希望在与该用户的*所有*聊天对话（例如，线程）中保留有关该用户的具体信息！

仅使用检查点，我们无法跨线程共享信息。这促使需要[`Store`](https://python.langchain.com/api_reference/langgraph/index.html#module-langgraph.store)接口。作为说明，我们可以定义一个`InMemoryStore`来存储有关用户的信息跨线程。我们像以前一样用检查点编译我们的图，并使用我们的新`in_memory_store`变量。

<Info>
  **LangGraph API 自动处理存储**
  使用 LangGraph API 时，无需手动实现或配置存储。API 在后台为您处理所有存储基础设施。
</Info>

### 基本用法

首先，让我们在不使用 LangGraph 的情况下单独展示这一点。

```typescript  theme={null}
import { MemoryStore } from "@langchain/langgraph";

const memoryStore = new MemoryStore();
```

记忆由`tuple`命名空间，在这个特定例子中将是`(<user_id>, "memories")`。命名空间可以是任何长度并代表任何内容，不一定必须是用户特定的。

```typescript  theme={null}
const userId = "1";
const namespaceForMemory = [userId, "memories"];
```

我们使用`store.put`方法将记忆保存到存储中的命名空间。当我们这样做时，我们指定上述定义的命名空间，以及记忆的键值对：键只是记忆的唯一标识符（`memory_id`），值（字典）是记忆本身。

```typescript  theme={null}
import { v4 as uuidv4 } from "uuid";

const memoryId = uuidv4();
const memory = { food_preference: "I like pizza" };
await memoryStore.put(namespaceForMemory, memoryId, memory);
```

我们可以使用`store.search`方法读取我们命名空间中的记忆，这将返回给定用户的所有记忆作为列表。最新的记忆是列表中的最后一个。

```typescript  theme={null}
const memories = await memoryStore.search(namespaceForMemory);
memories[memories.length - 1];

// {
//   value: { food_preference: 'I like pizza' },
//   key: '07e0caf4-1631-47b7-b15f-65515d4c1843',
//   namespace: ['1', 'memories'],
//   createdAt: '2024-10-02T17:22:31.590602+00:00',
//   updatedAt: '2024-10-02T17:22:31.590605+00:00'
// }
```

它的属性是：

* `value`: 此记忆的值
* `key`: 此记忆在此命名空间中的唯一键
* `namespace`: 字符串列表，此记忆类型的命名空间
* `createdAt`: 创建此记忆的时间戳
* `updatedAt`: 更新此记忆的时间戳

### 语义搜索

除了简单的检索，存储还支持语义搜索，允许您根据含义而非精确匹配来查找记忆。为此，使用嵌入模型配置存储：

```typescript  theme={null}
import { OpenAIEmbeddings } from "@langchain/openai";

const store = new InMemoryStore({
  index: {
    embeddings: new OpenAIEmbeddings({ model: "text-embedding-3-small" }),
    dims: 1536,
    fields: ["food_preference", "$"], // 要嵌入的字段
  },
});
```

现在搜索时，您可以使用自然语言查询查找相关记忆：

```typescript  theme={null}
// 查找关于食物偏好的记忆
// （这可以在将记忆放入存储后完成）
const memories = await store.search(namespaceForMemory, {
  query: "What does the user like to eat?",
  limit: 3, // 返回前3个匹配项
});
```

您可以通过配置`fields`参数或在存储记忆时指定`index`参数来控制记忆的哪些部分被嵌入：

```typescript  theme={null}
// 存储具有特定要嵌入的字段
await store.put(
  namespaceForMemory,
  uuidv4(),
  {
    food_preference: "I love Italian cuisine",
    context: "Discussing dinner plans",
  },
  { index: ["food_preference"] } // 仅嵌入"food_preferences"字段
);

// 存储而不嵌入（仍然可检索，但不可搜索）
await store.put(
  namespaceForMemory,
  uuidv4(),
  { system_info: "Last updated: 2024-01-01" },
  { index: false }
);
```

### 在 LangGraph 中使用

完成这一切后，我们在 LangGraph 中使用`memoryStore`。`memoryStore`与检查点协同工作：检查点将状态保存到线程中，如上所述，而`memoryStore`允许我们存储任意信息以供*跨*线程访问。我们如下编译图，同时使用检查点和`memoryStore`。

```typescript  theme={null}
import { MemorySaver } from "@langchain/langgraph";

// 我们需要这个，因为我们想要启用线程（对话）
const checkpointer = new MemorySaver();

// ... 定义图 ...

// 使用检查器和存储编译图
const graph = workflow.compile({ checkpointer, store: memoryStore });
```

我们像以前一样使用`thread_id`调用图，还使用`user_id`，我们将使用它来将我们的记忆命名空间到这个特定用户，如上所示。

```typescript  theme={null}
// 调用图
const userId = "1";
const config = { configurable: { thread_id: "1", user_id: userId } };

// 首先让我们对 AI 说声你好
for await (const update of await graph.stream(
  { messages: [{ role: "user", content: "hi" }] },
  { ...config, streamMode: "updates" }
)) {
  console.log(update);
}
```

我们可以通过访问`config`和`store`作为节点参数在*任何节点*中访问`memoryStore`和`user_id`。以下是我们在节点中使用语义搜索查找相关记忆的方式：

```typescript  theme={null}
import { MessagesZodMeta, Runtime } from "@langchain/langgraph";
import { BaseMessage } from "@langchain/core/messages";
import { registry } from "@langchain/langgraph/zod";
import * as z from "zod";

const MessagesZodState = z.object({
  messages: z
    .array(z.custom<BaseMessage>())
    .register(registry, MessagesZodMeta),
});

const updateMemory = async (
  state: z.infer<typeof MessagesZodState>,
  runtime: Runtime<{ user_id: string }>,
) => {
  // 从配置中获取用户ID
  const userId = runtime.context?.user_id;
  if (!userId) throw new Error("User ID is required");

  // 命名空间记忆
  const namespace = [userId, "memories"];

  // ... 分析对话并创建新记忆

  // 创建新的记忆ID
  const memoryId = uuidv4();

  // 我们创建一个新记忆
  await runtime.store?.put(namespace, memoryId, { memory });
};
```

如上所述，我们还可以在任何节点中访问存储并使用`store.search`方法获取记忆。回想一下，记忆作为对象列表返回，可以转换为字典。

```typescript  theme={null}
memories[memories.length - 1];
// {
//   value: { food_preference: 'I like pizza' },
//   key: '07e0caf4-1631-47b7-b15f-65515d4c1843',
//   namespace: ['1', 'memories'],
//   createdAt: '2024-10-02T17:22:31.590602+00:00',
//   updatedAt: '2024-10-02T17:22:31.590605+00:00'
// }
```

我们可以访问记忆并在模型调用中使用它们。

```typescript  theme={null}
const callModel = async (
  state: z.infer<typeof MessagesZodState>,
  config: LangGraphRunnableConfig,
  store: BaseStore
) => {
  // 从配置中获取用户ID
  const userId = config.configurable?.user_id;

  // 命名空间记忆
  const namespace = [userId, "memories"];

  // 基于最新消息进行搜索
  const memories = await store.search(namespace, {
    query: state.messages[state.messages.length - 1].content,
    limit: 3,
  });
  const info = memories.map((d) => d.value.memory).join("\n");

  // ... 在模型调用中使用记忆
};
```

如果我们创建一个新线程，只要`user_id`相同，我们仍然可以访问相同的记忆。

```typescript  theme={null}
// 调用图
const config = { configurable: { thread_id: "2", user_id: "1" } };

// 让我们再次说你好
for await (const update of await graph.stream(
  { messages: [{ role: "user", content: "hi, tell me about my memories" }] },
  { ...config, streamMode: "updates" }
)) {
  console.log(update);
}
```
当我们使用 LangSmith，无论是本地（例如，在[Studio](/langsmith/studio)中）还是[通过 LangSmith 托管](/langsmith/hosting)，基础存储默认可用，并且在图编译时无需指定。然而，要启用语义搜索，您**确实**需要在`langgraph.json`文件中配置索引设置。例如：

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

有关更多详细信息和配置选项，请参阅[部署指南](/langsmith/semantic-search)。

## 检查点库

在底层，检查点由符合[`BaseCheckpointSaver`](https://langchain-ai.github.io/langgraphjs/reference/classes/checkpoint.BaseCheckpointSaver.html)接口的检查点对象提供支持。LangGraph 提供了几个检查点实现，都是通过独立的、可安装的库实现的：

* `@langchain/langgraph-checkpoint`: 检查点保存器的基本接口（[`BaseCheckpointSaver`](https://langchain-ai.github.io/langgraphjs/reference/classes/checkpoint.BaseCheckpointSaver.html)）和序列化/反序列化接口（[`SerializerProtocol`](https://langchain-ai.github.io/langgraphjs/reference/interfaces/checkpoint.SerializerProtocol.html)）。包括用于实验的内存检查点实现（[`MemorySaver`](https://langchain-ai.github.io/langgraphjs/reference/classes/checkpoint.MemorySaver.html)）。LangGraph 附带了`@langchain/langgraph-checkpoint`。
* `@langchain/langgraph-checkpoint-sqlite`: 使用 SQLite 数据库（[`SqliteSaver`](https://langchain-ai.github.io/langgraphjs/reference/classes/checkpoint_sqlite.SqliteSaver.html)）的 LangGraph 检查点实现。适用于实验和本地工作流。需要单独安装。
* `@langchain/langgraph-checkpoint-postgres`: 使用 PostgreSQL 数据库（[`PostgresSaver`](https://langchain-ai.github.io/langgraphjs/reference/classes/checkpoint_postgres.PostgresSaver.html)）的高级检查点，用于 LangSmith。适用于在生产中使用。需要单独安装。

### 检查点接口

每个检查点都符合[`BaseCheckpointSaver`](https://langchain-ai.github.io/langgraphjs/reference/classes/checkpoint.BaseCheckpointSaver.html)接口并实现以下方法：

* `.put` - 存储检查点及其配置和元数据。
* `.putWrites` - 存储链接到检查点的中间写入（即[待写入](#pending-writes)）。
* `.getTuple` - 使用给定配置（`thread_id`和`checkpoint_id`）获取检查点元组。用于在`graph.getState()`中填充`StateSnapshot`。
* `.list` - 列出与给定配置和过滤条件匹配的检查点。用于在`graph.getStateHistory()`中填充状态历史。

### 序列化器

当检查点保存图状态时，它们需要序列化状态中的通道值。这是使用序列化器对象完成的。

`@langchain/langgraph-checkpoint`定义了实现序列化器的协议，并提供了一个默认实现，可以处理各种类型，包括 LangChain 和 LangGraph 原语、日期时间、枚举等。

## 功能

### 人在回路中(Human-in-the-loop)

首先，检查点通过允许人类检查、中断和批准图步骤来促进[人在回路工作流](/oss/javascript/langgraph/interrupts)。这些工作流需要检查点，因为人类必须能够随时查看图的状态，并且图必须在人类对状态进行任何更新后恢复执行。有关示例，请参阅[操作指南](/oss/javascript/langgraph/interrupts)。

### 记忆(Memory)

其次，检查器允许在交互之间保留["记忆"](/oss/javascript/concepts/memory)。在重复的人类交互（如对话）的情况下，任何后续消息都可以发送到该线程，该线程将保留之前对话的记忆。有关如何使用检查器添加和管理对话记忆的信息，请参阅[添加记忆](/oss/javascript/langgraph/add-memory)。

### 时间旅行(Time Travel)

第三，检查点允许["时间旅行"](/oss/javascript/langgraph/use-time-travel)，允许用户回放先前的图执行以查看和/或调试特定的图步骤。此外，检查点使能够在任意检查点分支图状态以探索替代轨迹成为可能。

### 容错(Fault-tolerance)

最后，检查点还提供容错和错误恢复：如果给定超步骤中的一个或多个节点失败，您可以从最后一个成功步骤重新启动图。此外，当图节点在给定超步骤执行中途失败时，LangGraph 存储在该超步骤成功完成的任何其他节点的待检查点写入，以便每当我们从该超步骤恢复图执行时，我们不会重新运行成功的节点。

#### 待写入(Pending writes)

此外，当图节点在给定超步骤执行中途失败时，LangGraph 存储在该超步骤成功完成的任何其他节点的待检查点写入，以便每当我们从该超步骤恢复图执行时，我们不会重新运行成功的节点。