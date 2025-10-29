# 流式传输（Streaming）

LangGraph 实现了一套用于**实时更新**的流式系统。流式传输在提升基于大语言模型（LLM）应用的响应性方面至关重要。通过**逐步展示输出**，即便完整响应尚未生成，也能显著提升用户体验（UX），尤其是在 LLM 存在延迟的场景中。

LangGraph 流式传输可以实现以下功能：

* <Icon icon="share-nodes" size={16} /> [**流式传输图状态**](#stream-graph-state) —— 通过 `updates` 或 `values` 模式获取状态更新或完整状态值。
* <Icon icon="square-poll-horizontal" size={16} /> [**流式传输子图输出**](#stream-subgraph-outputs) —— 可同时获取主图与嵌套子图的输出。
* <Icon icon="square-binary" size={16} /> [**流式传输 LLM tokens**](#messages) —— 从任意节点、子图或工具中捕获 LLM token 流。
* <Icon icon="table" size={16} /> [**流式传输自定义数据**](#stream-custom-data) —— 从工具函数中直接发送自定义更新或进度信号。
* <Icon icon="layer-plus" size={16} /> [**使用多种流式模式**](#stream-multiple-modes) —— 可选择 `values`（全量状态）、`updates`（增量状态）、`messages`（LLM token + 元数据）、`custom`（自定义数据）或 `debug`（详细执行信息）。

---

## 支持的流式模式

可将以下一种或多种模式以列表形式传递给 [`stream`](https://langchain-ai.github.io/langgraphjs/reference/classes/langgraph.CompiledStateGraph.html#stream) 方法：

| 模式         | 描述                                                 |
| ---------- | -------------------------------------------------- |
| `values`   | 在图每个步骤执行后，流式传输整个状态的完整值。                            |
| `updates`  | 在每个步骤后流式传输状态更新。如果同一步骤中有多个更新（例如同时运行多个节点），这些更新将单独传输。 |
| `custom`   | 从图节点内部流式传输自定义数据。                                   |
| `messages` | 从任意包含 LLM 调用的节点中流式传输 `(token, metadata)` 二元组。      |
| `debug`    | 在执行过程中尽可能多地流式传输调试信息。                               |

---

## 基本用法示例

LangGraph 的图对象暴露了 [`stream`](https://langchain-ai.github.io/langgraphjs/reference/classes/langgraph.Pregel.html#stream) 方法，用于以异步迭代器的形式返回流式输出：

```typescript
for await (const chunk of await graph.stream(inputs, {
  streamMode: "updates",
})) {
  console.log(chunk);
}
```

---

### 扩展示例：流式更新

```typescript
import { StateGraph, START, END } from "@langchain/langgraph";
import * as z from "zod";

const State = z.object({
  topic: z.string(),
  joke: z.string(),
});

const graph = new StateGraph(State)
  .addNode("refineTopic", (state) => {
    return { topic: state.topic + " and cats" };
  })
  .addNode("generateJoke", (state) => {
    return { joke: `This is a joke about ${state.topic}` };
  })
  .addEdge(START, "refineTopic")
  .addEdge("refineTopic", "generateJoke")
  .addEdge("generateJoke", END)
  .compile();

for await (const chunk of await graph.stream(
  { topic: "ice cream" },
  { streamMode: "updates" }
)) {
  console.log(chunk);
}
```

**输出：**

```
{'refineTopic': {'topic': 'ice cream and cats'}}
{'generateJoke': {'joke': 'This is a joke about ice cream and cats'}}
```

---

## 多模式流式传输

你可以同时启用多个流式模式。
返回结果为 `[mode, chunk]` 元组，其中 `mode` 表示模式名称，`chunk` 表示对应的数据。

```typescript
for await (const [mode, chunk] of await graph.stream(inputs, {
  streamMode: ["updates", "custom"],
})) {
  console.log(mode, chunk);
}
```

---

## 流式传输图状态

使用 `updates` 或 `values` 模式可在执行期间流式传输图的状态：

* `updates` —— 仅传输每步更新的状态。
* `values` —— 每步传输完整状态。

```typescript
import { StateGraph, START, END } from "@langchain/langgraph";
import * as z from "zod";

const State = z.object({
  topic: z.string(),
  joke: z.string(),
});

const graph = new StateGraph(State)
  .addNode("refineTopic", (state) => ({ topic: state.topic + " and cats" }))
  .addNode("generateJoke", (state) => ({ joke: `This is a joke about ${state.topic}` }))
  .addEdge(START, "refineTopic")
  .addEdge("refineTopic", "generateJoke")
  .addEdge("generateJoke", END)
  .compile();
```

---

## 流式传输子图输出

通过在 `.stream()` 方法中设置 `subgraphs: true`，可以在流式结果中包含子图的输出。
返回结果格式为 `[namespace, data]`，其中 `namespace` 表示路径层级，例如：

```
["parent_node:<task_id>", "child_node:<task_id>"]
```

---

## LLM Tokens 实时流式传输

使用 `messages` 模式可以逐 token 获取 LLM 的输出。
返回的 `[message_chunk, metadata]` 包含：

* `message_chunk`：LLM 生成的 token。
* `metadata`：包含节点名与模型调用详情的元数据。

你还可以通过 `metadata.tags` 或 `metadata.langgraph_node` 过滤特定模型或节点的输出。

---

## 自定义数据流式传输

你可以从节点或工具中使用 `config.writer()` 主动发送自定义数据。
需启用 `streamMode: "custom"`。

示例：

```typescript
.addNode("node", async (state, config) => {
  config.writer({ progress: "50%" });
  return { answer: "done" };
});
```

---

## 兼容任意 LLM

通过 `custom` 模式，你可以整合任意第三方流式 API（如自定义 LLM 客户端）：

```typescript
for await (const chunk of yourCustomStreamingClient(state.topic)) {
  config.writer({ custom_llm_chunk: chunk });
}
```

---

## 禁用特定模型的流式传输

若某些模型不支持流式传输，可在初始化时关闭：

```typescript
const model = new ChatOpenAI({
  model: "o1-preview",
  streaming: false,
});
```

---

> 📄 [在 GitHub 上编辑此文档](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/streaming.mdx)
> 💡 [将此文档连接至 Claude、VSCode 等工具以实时调用](/use-these-docs)
