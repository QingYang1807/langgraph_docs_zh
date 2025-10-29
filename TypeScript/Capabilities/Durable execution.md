# 持久化执行

**持久化执行**是一种技术，其中流程或工作流在关键点保存其进度，使其能够暂停并在稍后从 exactly where it left off 的位置恢复。这在需要 [人工参与循环](/oss/javascript/langgraph/interrupts) 的场景中特别有用，用户可以在继续之前检查、验证或修改流程，以及在可能遇到中断或错误的长时间运行任务中（例如调用 LLM 超时）。通过保留已完成的工作，持久化执行使流程能够在显著延迟后（例如一周后）恢复执行，而无需重新处理之前的步骤。

LangGraph 内置的 [持久化](/oss/javascript/langgraph/persistence) 层为工作流提供持久化执行，确保每个执行步骤的状态都保存到持久化存储中。此功能保证如果工作流被中断——无论是由于系统故障还是 [人工参与循环](/oss/javascript/langgraph/interrupts) 交互——它可以从最后记录的状态恢复执行。

<Tip>
  如果您正在使用 LangGraph 并配合检查点(checkpointer)，您已经启用了持久化执行。您可以在任何点暂停和恢复工作流，即使在中断或故障之后。
  要充分利用持久化执行，请确保您的工作流设计为 [确定性](#determinism-and-consistent-replay) 和 [幂等性](#determinism-and-consistent-replay)，并将任何副作用或非确定性操作包装在 [tasks](/oss/javascript/langgraph/functional-api#task) 中。您可以从 [StateGraph (Graph API)](/oss/javascript/langgraph/graph-api) 和 [Functional API](/oss/javascript/langgraph/functional-api) 中使用 [tasks](/oss/javascript/langgraph/functional-api#task)。
</Tip>

## 要求

要在 LangGraph 中利用持久化执行，您需要：

1. 通过指定一个将保存工作流进度的 [checkpointer](/oss/javascript/langgraph/persistence#checkpointer-libraries) 在您的工作流中启用 [持久化](/oss/javascript/langgraph/persistence)。

2. 执行工作流时指定 [线程标识符](/oss/javascript/langgraph/persistence#threads)。这将跟踪工作流特定实例的执行历史记录。

3. 将任何非确定性操作（例如随机数生成）或有副作用操作（例如文件写入、API 调用）包装在 [tasks](https://langchain-ai.github.io/langgraphjs/reference/functions/langgraph.task.html) 中，以确保当工作流恢复时，这些操作不会针对特定运行重复执行，而是从持久化层检索其结果。有关更多信息，请参见 [确定性与一致性重放](#determinism-and-consistent-replay)。

## 确定性与一致性重放

当您恢复工作流运行时，代码**不会**从执行停止的**相同代码行**恢复；相反，它将确定适当的 [起点](#starting-points-for-resuming-workflows) 从中继续执行。这意味着工作流将从 [起点](#starting-points-for-resuming-workflows) 重放所有步骤，直到达到停止的点。

因此，当您为持久化执行编写工作流时，您必须将任何非确定性操作（例如随机数生成）和任何有副作用操作（例如文件写入、API 调用）包装在 [tasks](/oss/javascript/langgraph/functional-api#task) 或 [nodes](/oss/javascript/langgraph/graph-api#nodes) 中。

为确保您的工作流是确定性的并且可以一致地重放，请遵循以下指南：

* **避免重复工作**：如果 [node](/oss/javascript/langgraph/graph-api#nodes) 包含多个有副作用操作（例如日志记录、文件写入或网络调用），请将每个操作包装在单独的 **task** 中。这确保当工作流恢复时，操作不会重复执行，而是从持久化层检索其结果。
* **封装非确定性操作**：将可能产生非确定性结果的任何代码（例如随机数生成）包装在 **tasks** 或 **nodes** 中。这确保在恢复时，工作流遵循确切的记录步骤序列，具有相同的结果。
* **使用幂等操作**：在可能的情况下，确保副作用（例如 API 调用、文件写入）是幂等的。这意味着如果在工作流失败后重试操作，它将具有与第一次执行时相同的效果。这对于导致数据写入的操作尤为重要。如果 **task** 启动但未能成功完成，工作流的恢复将重新运行 **task**，依靠记录的结果来保持一致性。使用幂等键或验证现有结果以避免意外重复，确保工作流执行平稳且可预测。

有关需要避免的陷阱示例，请参见功能性 API 中的 [常见陷阱](/oss/javascript/langgraph/functional-api#common-pitfalls) 部分，它展示了如何使用 **tasks** 构建代码以避免这些问题。相同的原则适用于 [StateGraph (Graph API)](https://langchain-ai.github.io/langgraphjs/reference/classes/langgraph.StateGraph.html)。

## 持久化模式

LangGraph 支持三种持久化模式，允许您根据应用程序的需求在性能和数据一致性之间取得平衡。持久化模式按持久化程度从低到高如下：

* [`"exit"`](#exit)
* [`"async"`](#async)
* [`"sync"`](#sync)

更高的持久化模式会给工作流执行带来更多开销。

<Tip>
  **v0.6.0 中添加**
  使用 `durability` 参数而不是 `checkpoint_during`（v0.6.0 中已弃用）进行持久性策略管理：

  * `durability="async"` 替换 `checkpoint_during=True`
  * `durability="exit"` 替换 `checkpoint_during=False`

  进行持久性策略管理，映射关系如下：

  * `checkpoint_during=True` -> `durability="async"`
  * `checkpoint_during=False` -> `durability="exit"`
</Tip>

### `"exit"`

仅在图执行完成时（成功或出错）才会持久化更改。这为长时间运行的图提供了最佳性能，但意味着中间状态不会被保存，因此您无法从执行中途故障中恢复或中断图执行。

### `"async"`

在下一步执行时异步持久化更改。这提供了良好的性能和持久性，但如果执行过程中进程崩溃，检查点可能无法写入。

### `"sync"`

在下一步开始之前同步持久化更改。这确保在继续执行之前每个检查点都被写入，以一些性能开销提供高持久性。

您可以在调用任何图执行方法时指定持久化模式：

## 在节点中使用任务

如果 [node](/oss/javascript/langgraph/graph-api#nodes) 包含多个操作，您可能会发现将每个操作转换为 **task** 比将操作重构为单独节点更容易。

<Tabs>
  <Tab title="原始版本">
    ```typescript  theme={null}
    import { StateGraph, START, END } from "@langchain/langgraph";
    import { MemorySaver } from "@langchain/langgraph";
    import { v4 as uuidv4 } from "uuid";
    import * as z from "zod";

    // 定义 Zod 架构以表示状态
    const State = z.object({
      url: z.string(),
      result: z.string().optional(),
    });

    const callApi = async (state: z.infer<typeof State>) => {
      const response = await fetch(state.url);  // [!code highlight]
      const text = await response.text();
      const result = text.slice(0, 100); // 副作用
      return {
        result,
      };
    };

    // 创建 StateGraph 构建器并为 callApi 函数添加一个节点
    const builder = new StateGraph(State)
      .addNode("callApi", callApi)
      .addEdge(START, "callApi")
      .addEdge("callApi", END);

    // 指定检查点
    const checkpointer = new MemorySaver();

    // 使用检查点编译图
    const graph = builder.compile({ checkpointer });

    // 定义带有线程 ID 的配置。
    const threadId = uuidv4();
    const config = { configurable: { thread_id: threadId } };

    // 调用图
    await graph.invoke({ url: "https://www.example.com" }, config);
    ```
  </Tab>

  <Tab title="使用任务">
    ```typescript  theme={null}
    import { StateGraph, START, END } from "@langchain/langgraph";
    import { MemorySaver } from "@langchain/langgraph";
    import { task } from "@langchain/langgraph";
    import { v4 as uuidv4 } from "uuid";
    import * as z from "zod";

    // 定义 Zod 架构以表示状态
    const State = z.object({
      urls: z.array(z.string()),
      results: z.array(z.string()).optional(),
    });

    const makeRequest = task("makeRequest", async (url: string) => {
      const response = await fetch(url);  // [!code highlight]
      const text = await response.text();
      return text.slice(0, 100);
    });

    const callApi = async (state: z.infer<typeof State>) => {
      const requests = state.urls.map((url) => makeRequest(url));  // [!code highlight]
      const results = await Promise.all(requests);
      return {
        results,
      };
    };

    // 创建 StateGraph 构建器并为 callApi 函数添加一个节点
    const builder = new StateGraph(State)
      .addNode("callApi", callApi)
      .addEdge(START, "callApi")
      .addEdge("callApi", END);

    // 指定检查点
    const checkpointer = new MemorySaver();

    // 使用检查点编译图
    const graph = builder.compile({ checkpointer });

    // 定义带有线程 ID 的配置。
    const threadId = uuidv4();
    const config = { configurable: { thread_id: threadId } };

    // 调用图
    await graph.invoke({ urls: ["https://www.example.com"] }, config);
    ```
  </Tab>
</Tabs>

## 恢复工作流

一旦您在工作流中启用了持久化执行，您可以在以下场景中恢复执行：

* **暂停和恢复工作流**：使用 [interrupt](https://langchain-ai.github.io/langgraphjs/reference/functions/langgraph.interrupt-2.html) 函数在特定点暂停工作流，并使用 [`Command`](https://langchain-ai.github.io/langgraphjs/reference/classes/langgraph.Command.html) 原语使用更新后的状态恢复它。有关更多详细信息，请参见 [**中断**](/oss/javascript/langgraph/interrupts)。
* **从故障中恢复**：在异常后从最后一个成功检查点自动恢复工作流（例如 LLM 提供商中断）。这涉及通过提供 `null` 作为输入值来执行具有相同线程标识符的工作流（请参见功能性 API 的此[示例](/oss/javascript/langgraph/use-functional-api#resuming-after-an-error)）。

## 恢复工作流的起点

* 如果您使用的是 [StateGraph (Graph API)](/oss/javascript/langgraph/graph-api)，起点是执行停止的 [**node**](/oss/javascript/langgraph/graph-api#nodes) 的开始位置。
* 如果您在节点中进行子图调用，起点将是调用被中止的子图的 **父** 节点。
  在子图中，起点将是执行停止的特定 [**node**](/oss/javascript/langgraph/graph-api#nodes)。
* 如果您使用的是 Functional API，起点是执行停止的 [**entrypoint**](/oss/javascript/langgraph/functional-api#entrypoint) 的开始位置。

***

<Callout icon="pen-to-square" iconType="regular">
  [在 GitHub 上编辑此页面的源代码。](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/durable-execution.mdx)
</Callout>

<Tip icon="terminal" iconType="regular">
  [通过 MCP 以编程方式连接这些文档](/use-these-docs) 到 Claude、VSCode 等，获取实时答案。
</Tip>