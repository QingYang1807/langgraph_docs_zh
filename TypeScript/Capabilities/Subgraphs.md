# 子图（Subgraphs）

子图是一个可以嵌套在另一个图（graph）节点中的 **LangGraph 工作流**。
它能让我们把复杂系统拆解为模块化子单元，从而实现复用与团队协作。

子图常见的应用场景包括：

* 🧠 **多智能体系统（multi-agent systems）**：每个智能体是独立子图；
* 🧩 **可复用流程组件**：将一组通用节点封装为可重用子图；
* 👥 **分布式开发协作**：不同团队可独立开发各自的子图，只需遵守输入输出接口即可。

---

## 子图与主图的通信方式

在 LangGraph 中，父图与子图之间有两种主要的交互方式：

1. [在节点中调用子图](#在节点中调用子图) — 子图通过函数形式在节点中被调用；
2. [将子图作为节点加入](#将子图作为节点加入) — 子图直接作为节点存在，**与父图共享状态键（state keys）**。

---

## 初始化（Setup）

```bash
npm install @langchain/langgraph
```

> 💡 **推荐连接 LangSmith 进行调试与性能监控**
> LangSmith 可以可视化追踪你的 LangGraph 执行链路。
> 👉 [注册 LangSmith](https://smith.langchain.com)

---

## 在节点中调用子图（Invoke a Graph from a Node）

这种方式最灵活，父图和子图 **可以拥有完全不同的状态结构（schemas）**。

> 典型场景：在多智能体系统中，每个智能体都拥有独立的私有记忆和对话历史。

你需要编写一个节点函数，在其中：

1. 将父图状态转换为子图的输入；
2. 调用子图；
3. 将子图输出再转换回父图的状态。

```typescript
import { StateGraph, START } from "@langchain/langgraph";
import * as z from "zod";

const SubgraphState = z.object({ bar: z.string() });

// 子图定义
const subgraph = new StateGraph(SubgraphState)
  .addNode("subgraphNode1", (state) => ({ bar: "hi! " + state.bar }))
  .addEdge(START, "subgraphNode1")
  .compile();

// 父图定义
const State = z.object({ foo: z.string() });

const graph = new StateGraph(State)
  .addNode("node1", async (state) => {
    const result = await subgraph.invoke({ bar: state.foo });
    return { foo: result.bar };
  })
  .addEdge(START, "node1")
  .compile();
```

---

### 多层子图（Parent → Child → Grandchild）

可以嵌套多层子图，每一层都有独立的状态与作用域：

```typescript
import { StateGraph, START, END } from "@langchain/langgraph";
import * as z from "zod";

// 孙图
const GrandChildState = z.object({ text: z.string() });
const grandchild = new StateGraph(GrandChildState)
  .addNode("greet", (s) => ({ text: s.text + ", how are you" }))
  .addEdge(START, "greet")
  .addEdge("greet", END)
  .compile();

// 子图
const ChildState = z.object({ msg: z.string() });
const child = new StateGraph(ChildState)
  .addNode("process", async (s) => {
    const res = await grandchild.invoke({ text: s.msg });
    return { msg: res.text + " today?" };
  })
  .addEdge(START, "process")
  .addEdge("process", END)
  .compile();

// 父图
const ParentState = z.object({ name: z.string() });
const parent = new StateGraph(ParentState)
  .addNode("init", (s) => ({ name: "hi " + s.name }))
  .addNode("child", async (s) => {
    const res = await child.invoke({ msg: s.name });
    return { name: res.msg };
  })
  .addNode("end", (s) => ({ name: s.name + " bye!" }))
  .addEdge(START, "init")
  .addEdge("init", "child")
  .addEdge("child", "end")
  .addEdge("end", END)
  .compile();

for await (const chunk of await parent.stream({ name: "Bob" }, { subgraphs: true })) {
  console.log(chunk);
}
```

---

## 将子图作为节点加入（Add a Graph as a Node）

当子图与父图之间**共享状态键（如 `messages`）**时，可以直接将子图作为节点加入父图。

> 场景示例：多智能体系统中，多个智能体通过共享的 `messages` 通道进行通信。

```typescript
import { StateGraph, START } from "@langchain/langgraph";
import * as z from "zod";

const State = z.object({ foo: z.string() });

// 定义子图
const subgraph = new StateGraph(State)
  .addNode("subgraphNode1", (s) => ({ foo: "hi! " + s.foo }))
  .addEdge(START, "subgraphNode1")
  .compile();

// 父图直接使用子图作为节点
const graph = new StateGraph(State)
  .addNode("node1", subgraph)
  .addEdge(START, "node1")
  .compile();
```

输出：

```
{ node1: { foo: 'hi! foo' } }
```

---

## 添加持久化（Add Persistence）

你只需要在 **编译父图时提供 checkpointer**，LangGraph 会自动将其传递到子图。

```typescript
import { StateGraph, START, MemorySaver } from "@langchain/langgraph";
import * as z from "zod";

const State = z.object({ foo: z.string() });

const subgraph = new StateGraph(State)
  .addNode("subgraphNode1", (s) => ({ foo: s.foo + "bar" }))
  .addEdge(START, "subgraphNode1")
  .compile();

const checkpointer = new MemorySaver();
const graph = new StateGraph(State)
  .addNode("node1", subgraph)
  .addEdge(START, "node1")
  .compile({ checkpointer });
```

如需让子图拥有独立记忆，可单独配置：

```typescript
const subgraph = subgraphBuilder.compile({ checkpointer: true });
```

---

## 查看子图状态（View Subgraph State）

你可以通过 `graph.getState(config, { subgraphs: true })` 查看被中断的子图状态。

> ⚠️ 仅在子图被 **中断（interrupted）** 时可查看。
> 一旦执行恢复，子图状态将不可再直接访问。

```typescript
import { StateGraph, START, MemorySaver, interrupt, Command } from "@langchain/langgraph";
import * as z from "zod";

const State = z.object({ foo: z.string() });

// 定义子图
const subgraph = new StateGraph(State)
  .addNode("subgraphNode1", (s) => {
    const val = interrupt("Provide value:");
    return { foo: s.foo + val };
  })
  .addEdge(START, "subgraphNode1")
  .compile();

// 父图
const graph = new StateGraph(State)
  .addNode("node1", subgraph)
  .addEdge(START, "node1")
  .compile({ checkpointer: new MemorySaver() });

const config = { configurable: { thread_id: "1" } };
await graph.invoke({ foo: "" }, config);

const parentState = await graph.getState(config);
const subState = (await graph.getState(config, { subgraphs: true })).tasks[0].state;

await graph.invoke(new Command({ resume: "bar" }), config);
```

---

## 流式输出子图结果（Stream Subgraph Outputs）

设置 `subgraphs: true` 即可在流式输出中包含子图数据。

```typescript
for await (const chunk of await graph.stream(
  { foo: "foo" },
  { streamMode: "updates", subgraphs: true }
)) {
  console.log(chunk);
}
```

示例输出：

```
[[], { node1: { foo: 'hi! foo' } }]
[['node2:xxx'], { subgraphNode1: { bar: 'bar' } }]
[['node2:xxx'], { subgraphNode2: { foo: 'hi! foobar' } }]
[[], { node2: { foo: 'hi! foobar' } }]
```

---

## 总结

| 功能            | 说明                   | 示例                                      |
| ------------- | -------------------- | --------------------------------------- |
| **在节点中调用子图**  | 父子图状态结构不同，适合多智能体     | `subgraph.invoke()`                     |
| **将子图作为节点加入** | 父子图共享状态键             | `.addNode("x", subgraph)`               |
| **持久化共享**     | 父图提供 checkpointer 即可 | `MemorySaver()`                         |
| **查看子图状态**    | 仅在中断时可见              | `getState(config, { subgraphs: true })` |
| **流式输出子图**    | 同时获取父图与子图事件          | `subgraphs: true`                       |

---

> ✏️ [编辑此页面](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/use-subgraphs.mdx)
> ⚙️ [接入 Claude、VSCode 等工具](/use-these-docs)，实现实时智能调试。
