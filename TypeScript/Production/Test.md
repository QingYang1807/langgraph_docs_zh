# 测试

当你完成了 **LangGraph** 智能体的原型设计后，下一步自然是为它编写测试。本指南将介绍一些在编写单元测试时非常有用的模式与技巧。

请注意，本指南是 **LangGraph 特定** 的，主要讲解如何测试带有自定义结构的图。如果你刚开始接触，可以先查看 [这个章节](/oss/javascript/langchain/test/)，它介绍了使用 LangChain 内置的 [`create_agent`] 方法进行测试的方式。

---

## 前置条件

首先，请确保你已安装 [`vitest`](https://vitest.dev/)：

```bash
npm install -D vitest
```

---

## 入门示例

由于许多 LangGraph 智能体依赖状态（state），一个常见的好做法是：在每个测试开始前创建图实例，并在测试中使用新的 **checkpointer**（检查点保存器）进行编译。

下面的示例演示了一个简单的线性图：流程依次经过 `node1` 和 `node2`，每个节点都会更新同一个状态键 `my_key`。

```ts
import { test, expect } from 'vitest';
import {
  StateGraph,
  START,
  END,
  MemorySaver,
} from '@langchain/langgraph';
import { z } from "zod/v4";

const State = z.object({
  my_key: z.string(),
});

const createGraph = () => {
  return new StateGraph(State)
    .addNode('node1', (state) => ({ my_key: 'hello from node1' }))
    .addNode('node2', (state) => ({ my_key: 'hello from node2' }))
    .addEdge(START, 'node1')
    .addEdge('node1', 'node2')
    .addEdge('node2', END);
};

test('基本的智能体执行测试', async () => {
  const uncompiledGraph = createGraph();
  const checkpointer = new MemorySaver();
  const compiledGraph = uncompiledGraph.compile({ checkpointer });
  const result = await compiledGraph.invoke(
    { my_key: 'initial_value' },
    { configurable: { thread_id: '1' } }
  );
  expect(result.my_key).toBe('hello from node2');
});
```

---

## 测试单个节点或边

编译后的 LangGraph 智能体会在 `graph.nodes` 中暴露每个节点的引用。
这意味着你可以单独测试某个节点，而无需运行整个图。
需要注意的是，这种方式不会使用你在编译图时传入的 checkpointer。

```ts
import { test, expect } from 'vitest';
import {
  StateGraph,
  START,
  END,
  MemorySaver,
} from '@langchain/langgraph';
import { z } from "zod/v4";

const State = z.object({
  my_key: z.string(),
});

const createGraph = () => {
  return new StateGraph(State)
    .addNode('node1', (state) => ({ my_key: 'hello from node1' }))
    .addNode('node2', (state) => ({ my_key: 'hello from node2' }))
    .addEdge(START, 'node1')
    .addEdge('node1', 'node2')
    .addEdge('node2', END);
};

test('单个节点执行测试', async () => {
  const uncompiledGraph = createGraph();
  const checkpointer = new MemorySaver();
  const compiledGraph = uncompiledGraph.compile({ checkpointer });
  // 只执行 node1
  const result = await compiledGraph.nodes['node1'].invoke(
    { my_key: 'initial_value' },
  );
  expect(result.my_key).toBe('hello from node1');
});
```

---

## 部分执行（Partial Execution）

当你的智能体由更大的图组成时，你可能希望只测试某一部分执行路径，而不是整个流程的端到端测试。
有时，你可以将这些部分重构为 [子图（subgraph）](/oss/javascript/langgraph/use-subgraphs)，从而在测试中单独调用它们。

但如果你不想改变图的整体结构，也可以利用 **LangGraph 的持久化机制（persistence）**，来模拟智能体在某个阶段暂停的状态，使其从中间节点开始执行，并在指定节点结束。
实现步骤如下：

1. 使用带有 **checkpointer** 的方式编译智能体（例如用于测试的内存型 `MemorySaver`）。
2. 调用智能体的 [`update_state`](/oss/javascript/langgraph/use-time-travel) 方法，并将参数 `asNode` 设置为**你希望开始执行的节点之前的那个节点**。
3. 通过相同的 `thread_id` 调用智能体，并设置 `interruptBefore` 或 `interruptAfter` 参数来控制停止位置。

下面是一个示例：只执行线性图中的第二个和第三个节点。

```ts
import { test, expect } from 'vitest';
import {
  StateGraph,
  START,
  END,
  MemorySaver,
} from '@langchain/langgraph';
import { z } from "zod/v4";

const State = z.object({
  my_key: z.string(),
});

const createGraph = () => {
  return new StateGraph(State)
    .addNode('node1', (state) => ({ my_key: 'hello from node1' }))
    .addNode('node2', (state) => ({ my_key: 'hello from node2' }))
    .addNode('node3', (state) => ({ my_key: 'hello from node3' }))
    .addNode('node4', (state) => ({ my_key: 'hello from node4' }))
    .addEdge(START, 'node1')
    .addEdge('node1', 'node2')
    .addEdge('node2', 'node3')
    .addEdge('node3', 'node4')
    .addEdge('node4', END);
};

test('部分执行测试（node2 → node3）', async () => {
  const uncompiledGraph = createGraph();
  const checkpointer = new MemorySaver();
  const compiledGraph = uncompiledGraph.compile({ checkpointer });
  await compiledGraph.updateState(
    { configurable: { thread_id: '1' } },
    // 模拟 node1 执行结束时的状态
    { my_key: 'initial_value' },
    // 将保存的状态标记为来自 node1
    'node1',
  );
  const result = await compiledGraph.invoke(
    // 传入 null 表示从保存状态继续执行
    null,
    {
      configurable: { thread_id: '1' },
      // 执行完 node3 后停止
      interruptAfter: ['node3']
    },
  );
  expect(result.my_key).toBe('hello from node3');
});
```

---

💡 **编辑提示**
👉 [在 GitHub 上编辑本文档](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/test.mdx)

🧠 **开发者提示**
👉 [将这些文档连接到 Claude、VSCode 等工具](/use-these-docs)，通过 MCP 获取实时答案。

---
