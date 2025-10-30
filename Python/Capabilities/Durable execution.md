# 持久执行

**持久执行**是一种技术，其中进程或工作流在关键点保存其进度，允许它暂停并在稍后精确地从离开的地方恢复。这在需要[人工参与循环](/oss/python/langgraph/interrupts)的场景中特别有用，用户可以在继续前检查、验证或修改进程，以及在可能遇到中断或错误的长时间运行任务中（例如，对LLM的调用超时）。通过保留已完成的工作，持久执行使进程能够在显著延迟后（例如一周后）恢复而无需重新处理之前的步骤。

LangGraph内置的[持久化](/oss/python/langgraph/persistence)层为工作流提供持久执行功能，确保每个执行步骤的状态都被保存到持久存储中。此功能保证如果工作流被中断——无论是系统故障还是[人工参与循环](/oss/python/langgraph/interrupts)交互——都可以从最后记录的状态恢复。

<Tip>
  如果你使用LangGraph与检查点(checkpointer)一起，你已经启用了持久执行。你可以在任何点暂停和恢复工作流，即使在中断或故障之后。
  要充分利用持久执行，请确保你的工作流设计为[确定性](#determinism-and-consistent-replay)和[幂等性](#determinism-and-consistent-replay)，并将任何副作用或非确定性操作包装在[任务](/oss/python/langgraph/functional-api#task)中。你可以从[状态图（图形API）](/oss/python/langgraph/graph-api)和[函数式API](/oss/python/langgraph/functional-api)中使用[任务](/oss/python/langgraph/functional-api#task)。
</Tip>

## 要求

要在LangGraph中利用持久执行，你需要：

1. 通过指定一个保存工作流进度的[检查点(checkpointer)](/oss/python/langgraph/persistence#checkpointer-libraries)来在工作流中启用[持久化](/oss/python/langgraph/persistence)。

2. 在执行工作流时指定一个[线程标识符](/oss/python/langgraph/persistence#threads)。这将跟踪工作流特定实例的执行历史记录。

3. 将任何非确定性操作（例如随机数生成）或有副作用的操作（例如文件写入、API调用）包装在@\[`task`]中，以确保当工作流恢复时，这些操作不会为特定运行重复执行，而是从持久化层检索其结果。有关更多信息，请参阅[确定性与一致重放](#determinism-and-consistent-replay)。

## 确定性与一致重放

当你恢复工作流运行时，代码**不会**从停止执行的**同一行代码**处恢复；相反，它将识别一个适当的[起点](#starting-points-for-resuming-workflows)来从中断处继续。这意味着工作流将从[起点](#starting-points-for-resuming-workflows)重放所有步骤，直到达到停止的点。

因此，在为持久执行编写工作流时，你必须将任何非确定性操作（例如随机数生成）和任何有副作用的操作（例如文件写入、API调用）包装在[任务](/oss/python/langgraph/functional-api#task)或[节点](/oss/python/langgraph/graph-api#nodes)中。

为确保你的工作流是确定性的并且可以一致地重放，请遵循以下准则：

* **避免重复工作**：如果一个[节点](/oss/python/langgraph/graph-api#nodes)包含多个有副作用的操作（例如日志记录、文件写入或网络调用），将每个操作包装在一个单独的**任务**中。这确保当工作流恢复时，操作不会重复执行，其结果将从持久化层检索。
* **封装非确定性操作**：将可能产生非确定性结果的任何代码（例如随机数生成）包装在**任务**或**节点**中。这确保在恢复时，工作流遵循确切的记录步骤序列并产生相同的结果。
* **使用幂等操作**：尽可能确保副作用（例如API调用、文件写入）是幂等的。这意味着如果工作流失败后重试操作，它将具有与首次执行时相同的效果。这对于导致数据写入的操作尤为重要。如果一个**任务**启动但未能成功完成，工作流的恢复将重新运行该**任务**，依赖记录的结果来保持一致性。使用幂等键或验证现有结果以避免意外重复，确保工作流执行流畅且可预测。

有关要避免的陷阱示例，请参阅函数式API中的[常见陷阱](/oss/python/langgraph/functional-api#common-pitfalls)部分，该部分展示了如何使用**任务**构建代码以避免这些问题。相同的原则适用于[状态图（图形API）](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.StateGraph)。

## 持久模式

LangGraph支持三种持久模式，允许您根据应用程序需求平衡性能和数据一致性。持久模式从最不持久到最持久如下：

* [`"exit"`](#exit)
* [`"async"`](#async)
* [`"sync"`](#sync)

更高的持久模式会增加工作流执行的开销。

<Tip>
  **v0.6.0版本添加**
  使用`durability`参数而不是`checkpoint_during`（v0.6.0中已弃用）来进行持久性策略管理：

  * `durability="async"` 替换 `checkpoint_during=True`
  * `durability="exit"` 替换 `checkpoint_during=False`

  用于持久性策略管理，映射如下：

  * `checkpoint_during=True` -> `durability="async"`
  * `checkpoint_during=False` -> `durability="exit"`
</Tip>

### `"exit"`

仅在图形执行完成时（成功或出错）保存更改。这为长时间运行的图形提供了最佳性能，但意味着中间状态未保存，因此您无法从执行中故障或中断图形执行中恢复。

### `"async"`

在下一步执行时异步保存更改。这提供了良好的性能和持久性，但存在一个小的风险，即如果在执行过程中进程崩溃，检查点可能未被写入。

### `"sync"`

在下一步开始前同步保存更改。这确保在继续执行前每个检查点都被写入，提供了高持久性，但会带来一些性能开销。

你可以在调用任何图形执行方法时指定持久模式：

```python  theme={null}
graph.stream(
    {"input": "test"},
    durability="sync"
)
```

## 在节点中使用任务

如果一个[节点](/oss/python/langgraph/graph-api#nodes)包含多个操作，您可能会发现将每个操作转换为**任务**比将操作重构为单独节点更容易。

<Tabs>
  <Tab title="原始版本">
    ```python  theme={null}
    from typing import NotRequired
    from typing_extensions import TypedDict
    import uuid

    from langgraph.checkpoint.memory import InMemorySaver
    from langgraph.graph import StateGraph, START, END
    import requests

    # 定义一个TypedDict来表示状态
    class State(TypedDict):
        url: str
        result: NotRequired[str]

    def call_api(state: State):
        """示例节点，执行API请求。"""
        result = requests.get(state['url']).text[:100]  # 副作用  # [!code highlight]
        return {
            "result": result
        }

    # 创建一个StateGraph构建器并添加call_api函数的节点
    builder = StateGraph(State)
    builder.add_node("call_api", call_api)

    # 将开始和结束节点连接到call_api节点
    builder.add_edge(START, "call_api")
    builder.add_edge("call_api", END)

    # 指定一个检查点
    checkpointer = InMemorySaver()

    # 使用检查点编译图形
    graph = builder.compile(checkpointer=checkpointer)

    # 定义一个带有线程ID的配置。
    thread_id = uuid.uuid4()
    config = {"configurable": {"thread_id": thread_id}}

    # 调用图形
    graph.invoke({"url": "https://www.example.com"}, config)
    ```
  </Tab>

  <Tab title="使用任务">
    ```python  theme={null}
    from typing import NotRequired
    from typing_extensions import TypedDict
    import uuid

    from langgraph.checkpoint.memory import InMemorySaver
    from langgraph.func import task
    from langgraph.graph import StateGraph, START, END
    import requests

    # 定义一个TypedDict来表示状态
    class State(TypedDict):
        urls: list[str]
        result: NotRequired[list[str]]


    @task
    def _make_request(url: str):
        """执行请求。"""
        return requests.get(url).text[:100]  # [!code highlight]

    def call_api(state: State):
        """示例节点，执行API请求。"""
        requests = [_make_request(url) for url in state['urls']]  # [!code highlight]
        results = [request.result() for request in requests]
        return {
            "results": results
        }

    # 创建一个StateGraph构建器并添加call_api函数的节点
    builder = StateGraph(State)
    builder.add_node("call_api", call_api)

    # 将开始和结束节点连接到call_api节点
    builder.add_edge(START, "call_api")
    builder.add_edge("call_api", END)

    # 指定一个检查点
    checkpointer = InMemorySaver()

    # 使用检查点编译图形
    graph = builder.compile(checkpointer=checkpointer)

    # 定义一个带有线程ID的配置。
    thread_id = uuid.uuid4()
    config = {"configurable": {"thread_id": thread_id}}

    # 调用图形
    graph.invoke({"urls": ["https://www.example.com"]}, config)
    ```
  </Tab>
</Tabs>

## 恢复工作流

一旦你在工作流中启用了持久执行，你可以在以下场景恢复执行：

* **暂停和恢复工作流**：使用[interrupt](https://reference.langchain.com/python/langgraph/types/#langgraph.types.interrupt)函数在特定点暂停工作流，并使用[`Command`](https://reference.langchain.com/python/langgraph/types/#langgraph.types.Command)原语使用更新的状态恢复它。有关更多详情，请参阅[**中断**](/oss/python/langgraph/interrupts)。
* **从故障中恢复**：在异常后（例如LLM提供商服务中断）自动从最后一个成功检查点恢复工作流。这涉及通过提供`None`作为输入值来执行具有相同线程标识符的工作流（请参阅函数式API的[此示例](/oss/python/langgraph/use-functional-api#resuming-after-an-error)）。

## 恢复工作流的起点

* 如果你使用[状态图（图形API）](/https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.StateGraph)，起点是执行停止的[**节点**](/oss/python/langgraph/graph-api#nodes)的开始。
* 如果你在节点内进行子图调用，起点将是调用被中止子图的**父**节点。
  在子图内，起点是执行停止的特定[**节点**](/oss/python/langgraph/graph-api#nodes)的开始。
* 如果你使用函数式API，起点是执行停止的[**入口点**](/oss/python/langgraph/functional-api#entrypoint)的开始。

***

<Callout icon="pen-to-square" iconType="regular">
  [在GitHub上编辑此页面的源代码。](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/durable-execution.mdx)
</Callout>

<Tip icon="terminal" iconType="regular">
  [通过MCP将这些文档与Claude、VSCode等连接](/use-these-docs)，以获取实时答案。
</Tip>