# LangGraph 运行时

[`Pregel`](https://reference.langchain.com/python/langgraph/pregel/) 实现了 LangGraph 的运行时，负责管理 LangGraph 应用的执行。

编译一个 [StateGraph](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.StateGraph) 或创建一个 [`@entrypoint`](https://reference.langchain.com/python/langgraph/func/#langgraph.func.entrypoint) 会生成一个 [`Pregel`](https://reference.langchain.com/python/langgraph/pregel/) 实例，该实例可以通过输入进行调用。

本指南从高层次解释了运行时，并提供了直接使用 Pregel 实现应用的说明。

> **注意**：[`Pregel`](https://reference.langchain.com/python/langgraph/pregel/) 运行时以 [Google 的 Pregel 算法](https://research.google/pubs/pub37252/) 命名，该算法描述了一种使用图进行大规模并行计算的高效方法。

## 概述

在 LangGraph 中，Pregel 将 [**参与者**](https://en.wikipedia.org/wiki/Actor_model) 和**通道**组合成一个单一的应用程序。**参与者**从通道读取数据并向通道写入数据。Pregel 将应用的执行组织成多个步骤，遵循**Pregel 算法**/**批量同步并行**模型。

每个步骤包含三个阶段：

* **规划**：确定在当前步骤执行哪些**参与者**。例如，在第一步中，选择订阅特殊**输入**通道的**参与者**；在后续步骤中，选择订阅上一步更新通道的**参与者**。
* **执行**：并行执行所有选定的**参与者**，直到全部完成、其中一个失败或达到超时。在此阶段，通道更新对参与者在下一步之前不可见。
* **更新**：使用本步骤**参与者**写入的值更新通道。

重复此过程，直到没有**参与者**被选择执行，或达到最大步数。

## 参与者

**参与者** 是一个 `PregelNode`。它订阅通道，从通道读取数据，并向通道写入数据。可以将其视为 Pregel 算法中的一个**参与者**。`PregelNodes` 实现了 LangChain 的 Runnable 接口。

## 通道

通道用于在参与者（PregelNodes）之间通信。每个通道都有一个值类型、更新类型和更新函数——更新函数接收一系列更新并修改存储的值。通道可用于将数据从一个链发送到另一个链，或将数据从链发送到未来的自身步骤。LangGraph 提供了许多内置通道：

* [`LastValue`](https://reference.langchain.com/python/langgraph/channels/#langgraph.channels.LastValue)：默认通道，存储发送到通道的最后一个值，适用于输入和输出值，或用于将数据从前一步发送到下一步。
* [`Topic`](https://reference.langchain.com/python/langgraph/channels/#langgraph.channels.Topic)：可配置的 PubSub 主题，用于在**参与者**之间发送多个值，或用于累积输出。可以配置为去重值或在多个步骤中累积值。
* [`BinaryOperatorAggregate`](https://reference.langchain.com/python/langgraph/pregel/#langgraph.pregel.Pregel--advanced-channels-context-and-binaryoperatoraggregate)：存储持久值，通过将二元运算符应用于当前值和发送到通道的每个更新来更新，适用于在多个步骤中计算聚合；例如，`total = BinaryOperatorAggregate(int, operator.add)`

## 示例

虽然大多数用户会通过 [StateGraph](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.StateGraph) API 或 [`@entrypoint`](https://reference.langchain.com/python/langgraph/func/#langgraph.func.entrypoint) 装饰器与 Pregel 交互，但也可以直接与 Pregel 交互。

下面是几个不同的示例，让您了解 Pregel API 的使用方法。

<Tabs>
  <Tab title="单个节点">
    ```python  theme={null}
    from langgraph.channels import EphemeralValue
    from langgraph.pregel import Pregel, NodeBuilder

    node1 = (
        NodeBuilder().subscribe_only("a")
        .do(lambda x: x + x)
        .write_to("b")
    )

    app = Pregel(
        nodes={"node1": node1},
        channels={
            "a": EphemeralValue(str),
            "b": EphemeralValue(str),
        },
        input_channels=["a"],
        output_channels=["b"],
    )

    app.invoke({"a": "foo"})
    ```

    ```con  theme={null}
    {'b': 'foofoo'}
    ```
  </Tab>

  <Tab title="多个节点">
    ```python  theme={null}
    from langgraph.channels import LastValue, EphemeralValue
    from langgraph.pregel import Pregel, NodeBuilder

    node1 = (
        NodeBuilder().subscribe_only("a")
        .do(lambda x: x + x)
        .write_to("b")
    )

    node2 = (
        NodeBuilder().subscribe_only("b")
        .do(lambda x: x + x)
        .write_to("c")
    )


    app = Pregel(
        nodes={"node1": node1, "node2": node2},
        channels={
            "a": EphemeralValue(str),
            "b": LastValue(str),
            "c": EphemeralValue(str),
        },
        input_channels=["a"],
        output_channels=["b", "c"],
    )

    app.invoke({"a": "foo"})
    ```

    ```con  theme={null}
    {'b': 'foofoo', 'c': 'foofoofoofoo'}
    ```
  </Tab>

  <Tab title="主题">
    ```python  theme={null}
    from langgraph.channels import EphemeralValue, Topic
    from langgraph.pregel import Pregel, NodeBuilder

    node1 = (
        NodeBuilder().subscribe_only("a")
        .do(lambda x: x + x)
        .write_to("b", "c")
    )

    node2 = (
        NodeBuilder().subscribe_to("b")
        .do(lambda x: x["b"] + x["b"])
        .write_to("c")
    )

    app = Pregel(
        nodes={"node1": node1, "node2": node2},
        channels={
            "a": EphemeralValue(str),
            "b": EphemeralValue(str),
            "c": Topic(str, accumulate=True),
        },
        input_channels=["a"],
        output_channels=["c"],
    )

    app.invoke({"a": "foo"})
    ```

    ```pycon  theme={null}
    {'c': ['foofoo', 'foofoofoofoo']}
    ```
  </Tab>

  <Tab title="BinaryOperatorAggregate">
    此示例演示如何使用 [`BinaryOperatorAggregate`](https://reference.langchain.com/python/langgraph/pregel/#langgraph.pregel.Pregel--advanced-channels-context-and-binaryoperatoraggregate) 通道实现归约器。

    ```python  theme={null}
    from langgraph.channels import EphemeralValue, BinaryOperatorAggregate
    from langgraph.pregel import Pregel, NodeBuilder


    node1 = (
        NodeBuilder().subscribe_only("a")
        .do(lambda x: x + x)
        .write_to("b", "c")
    )

    node2 = (
        NodeBuilder().subscribe_only("b")
        .do(lambda x: x + x)
        .write_to("c")
    )

    def reducer(current, update):
        if current:
            return current + " | " + update
        else:
            return update

    app = Pregel(
        nodes={"node1": node1, "node2": node2},
        channels={
            "a": EphemeralValue(str),
            "b": EphemeralValue(str),
            "c": BinaryOperatorAggregate(str, operator=reducer),
        },
        input_channels=["a"],
        output_channels=["c"],
    )

    app.invoke({"a": "foo"})
    ```
  </Tab>

  <Tab title="循环">
    此示例演示如何通过让链写入其订阅的通道来在图中引入循环。执行将持续写入，直到向通道写入 `None` 值。

    ```python  theme={null}
    from langgraph.channels import EphemeralValue
    from langgraph.pregel import Pregel, NodeBuilder, ChannelWriteEntry

    example_node = (
        NodeBuilder().subscribe_only("value")
        .do(lambda x: x + x if len(x) < 10 else None)
        .write_to(ChannelWriteEntry("value", skip_none=True))
    )

    app = Pregel(
        nodes={"example_node": example_node},
        channels={
            "value": EphemeralValue(str),
        },
        input_channels=["value"],
        output_channels=["value"],
    )

    app.invoke({"value": "a"})
    ```

    ```pycon  theme={null}
    {'value': 'aaaaaaaaaaaaaaaa'}
    ```
  </Tab>
</Tabs>

## 高级 API

LangGraph 提供了两种创建 Pregel 应用的高级 API：[StateGraph (图 API)](/oss/python/langgraph/graph-api) 和 [函数式 API](/oss/python/langgraph/functional-api)。

<Tabs>
  <Tab title="StateGraph (图 API)">
    [StateGraph (图 API)](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.StateGraph) 是一个更高级的抽象，简化了 Pregel 应用的创建。它允许您定义一个节点和边的图。当您编译图时，StateGraph API 会自动为您创建 Pregel 应用。

    ```python  theme={null}
    from typing import TypedDict

    from langgraph.constants import START
    from langgraph.graph import StateGraph

    class Essay(TypedDict):
        topic: str
        content: str | None
        score: float | None

    def write_essay(essay: Essay):
        return {
            "content": f"Essay about {essay['topic']}",
        }

    def score_essay(essay: Essay):
        return {
            "score": 10
        }

    builder = StateGraph(Essay)
    builder.add_node(write_essay)
    builder.add_node(score_essay)
    builder.add_edge(START, "write_essay")
    builder.add_edge("write_essay", "score_essay")

    # 编译图。
    # 这将返回一个 Pregel 实例。
    graph = builder.compile()
    ```

    编译后的 Pregel 实例将与一系列节点和通道关联。您可以通过打印它们来检查节点和通道。

    ```python  theme={null}
    print(graph.nodes)
    ```

    您将看到类似这样的内容：

    ```pycon  theme={null}
    {'__start__': <langgraph.pregel.read.PregelNode at 0x7d05e3ba1810>,
     'write_essay': <langgraph.pregel.read.PregelNode at 0x7d05e3ba14d0>,
     'score_essay': <langgraph.pregel.read.PregelNode at 0x7d05e3ba1710>}
    ```

    ```python  theme={null}
    print(graph.channels)
    ```

    您应该看到类似这样的内容

    ```pycon  theme={null}
    {'topic': <langgraph.channels.last_value.LastValue at 0x7d05e3294d80>,
     'content': <langgraph.channels.last_value.LastValue at 0x7d05e3295040>,
     'score': <langgraph.channels.last_value.LastValue at 0x7d05e3295980>,
     '__start__': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e3297e00>,
     'write_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e32960c0>,
     'score_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e2d8ab80>,
     'branch:__start__:__self__:write_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e32941c0>,
     'branch:__start__:__self__:score_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e2d88800>,
     'branch:write_essay:__self__:write_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e3295ec0>,
     'branch:write_essay:__self__:score_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e2d8ac00>,
     'branch:score_essay:__self__:write_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e2d89700>,
     'branch:score_essay:__self__:score_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e2d8b400>,
     'start:write_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e2d8b280>}
    ```
  </Tab>

  <Tab title="函数式 API">
    在[函数式 API](/oss/python/langgraph/functional-api)中，您可以使用 [`@entrypoint`](https://reference.langchain.com/python/langgraph/func/#langgraph.func.entrypoint) 创建 Pregel 应用。`entrypoint` 装饰器允许您定义一个接受输入并返回输出的函数。

    ```python  theme={null}
    from typing import TypedDict

    from langgraph.checkpoint.memory import InMemorySaver
    from langgraph.func import entrypoint

    class Essay(TypedDict):
        topic: str
        content: str | None
        score: float | None


    checkpointer = InMemorySaver()

    @entrypoint(checkpointer=checkpointer)
    def write_essay(essay: Essay):
        return {
            "content": f"Essay about {essay['topic']}",
        }

    print("Nodes: ")
    print(write_essay.nodes)
    print("Channels: ")
    print(write_essay.channels)
    ```

    ```pycon  theme={null}
    Nodes:
    {'write_essay': <langgraph.pregel.read.PregelNode object at 0x7d05e2f9aad0>}
    Channels:
    {'__start__': <langgraph.channels.ephemeral_value.EphemeralValue object at 0x7d05e2c906c0>, '__end__': <langgraph.channels.last_value.LastValue object at 0x7d05e2c90c40>, '__previous__': <langgraph.channels.last_value.LastValue object at 0x7d05e1007280>}
    ```
  </Tab>
</Tabs>

***

<Callout icon="pen-to-square" iconType="regular">
  [在 GitHub 上编辑此页面的源代码。](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/pregel.mdx)
</Callout>

<Tip icon="terminal" iconType="regular">
  [通过 MCP 以编程方式连接这些文档](/use-these-docs)，与 Claude、VSCode 等连接，以获得实时答案。
</Tip>