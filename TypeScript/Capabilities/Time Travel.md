# 时间旅行（Time Travel）

在构建依赖非确定性系统（如由大语言模型 LLM 驱动的智能体 Agent）时，我们常常希望深入理解模型的决策过程。时间旅行（Time Travel）功能可以让你回到任意历史检查点，重放或修改状态，从而进行更细致的分析与探索。

它的核心用途包括：

1. 💡 **理解推理过程**：分析模型如何一步步得到正确答案。
2. 🪲 **调试错误**：定位问题出现的环节及原因。
3. 🔍 **探索备选路径**：从某个节点修改状态，观察不同输入带来的不同结果。

LangGraph 提供的 [Time Travel 功能](/oss/javascript/langgraph/use-time-travel) 支持你在任意时间点恢复执行、重放历史或分支探索。每次从旧检查点恢复，都会在历史中创建一个新的分支。

---

## 使用步骤

在 LangGraph 中使用时间旅行，通常分为四步：

1. [运行图](#1-运行图)：使用 [`invoke`](https://langchain-ai.github.io/langgraphjs/reference/classes/langgraph.CompiledStateGraph.html#invoke) 或 [`stream`](https://langchain-ai.github.io/langgraphjs/reference/classes/langgraph.CompiledStateGraph.html#stream) 执行工作流；
2. [定位检查点](#2-定位检查点)：通过 [`getStateHistory`] 方法查询指定 `thread_id` 的执行历史；
3. [更新状态（可选）](#3-更新状态)：使用 [`updateState`] 修改某个检查点的状态，以探索不同路径；
4. [从检查点恢复执行](#4-从检查点恢复执行)：使用 `invoke` 或 `stream` 并传入 `null` 输入，从指定 `checkpoint_id` 恢复执行。

> 💡 想了解时间旅行的概念性介绍，可参阅：[Time travel](/oss/javascript/langgraph/use-time-travel)

---

## 工作流示例

以下示例构建了一个简单的 LangGraph 工作流，用于生成一个笑话主题并基于该主题编写笑话。
该例展示了如何运行图、获取历史检查点、修改状态并从旧检查点恢复执行。

---

### 环境准备

安装依赖：

```bash
npm install @langchain/langgraph @langchain/anthropic
```

配置 Anthropic API Key：

```typescript
process.env.ANTHROPIC_API_KEY = "YOUR_API_KEY";
```

> 💡 建议注册 [LangSmith](https://smith.langchain.com)，以可视化查看执行过程，调试并优化 LangGraph 项目性能。

---

### 构建工作流

```typescript
import { v4 as uuidv4 } from "uuid";
import * as z from "zod";
import { StateGraph, START, END } from "@langchain/langgraph";
import { ChatAnthropic } from "@langchain/anthropic";
import { MemorySaver } from "@langchain/langgraph";

const State = z.object({
  topic: z.string().optional(),
  joke: z.string().optional(),
});

const model = new ChatAnthropic({
  model: "claude-sonnet-4-5",
  temperature: 0,
});

// 定义工作流
const workflow = new StateGraph(State)
  .addNode("generateTopic", async (state) => {
    const msg = await model.invoke("给我一个搞笑的笑话主题");
    return { topic: msg.content };
  })
  .addNode("writeJoke", async (state) => {
    const msg = await model.invoke(`请写一个关于 ${state.topic} 的简短笑话`);
    return { joke: msg.content };
  })
  .addEdge(START, "generateTopic")
  .addEdge("generateTopic", "writeJoke")
  .addEdge("writeJoke", END);

const checkpointer = new MemorySaver();
const graph = workflow.compile({ checkpointer });
```

---

### 1. 运行图

```typescript
const config = { configurable: { thread_id: uuidv4() } };

const state = await graph.invoke({}, config);

console.log(state.topic);
console.log(state.joke);
```

**示例输出：**

```
主题: “洗衣机里袜子的秘密生活”

笑话:
我终于发现了洗衣机里袜子消失的真相——
原来它们都跑去和别人家的袜子私奔了！
```

---

### 2. 定位检查点

```typescript
const states = [];
for await (const state of graph.getStateHistory(config)) {
  states.push(state);
}

for (const state of states) {
  console.log(state.next);
  console.log(state.config.configurable?.checkpoint_id);
  console.log();
}
```

**输出：**

```
[]
1f02ac4a-ec9f-6524-8002-8f7b0bbeed0e
['writeJoke']
1f02ac4a-ce2a-6494-8001-cb2e2d651227
['generateTopic']
1f02ac4a-a4e0-630d-8000-b73c254ba748
```

选择第二个检查点（即生成主题后）：

```typescript
const selectedState = states[1];
console.log(selectedState.next);
console.log(selectedState.values);
```

输出：

```
['writeJoke']
{ topic: "洗衣机里的袜子秘密生活" }
```

---

### 3. 更新状态（可选）

你可以在某个检查点修改状态值，以探索不同的执行分支：

```typescript
const newConfig = await graph.updateState(selectedState.config, {
  topic: "小鸡",
});
console.log(newConfig);
```

输出：

```
{
  configurable: {
    thread_id: 'c62e2e03-c27b-4cb6-8cea-ea9bfedae006',
    checkpoint_id: '1f02ac4a-ecee-600b-8002-a1d21df32e4c'
  }
}
```

---

### 4. 从检查点恢复执行

```typescript
await graph.invoke(null, newConfig);
```

输出：

```
{
  topic: "小鸡",
  joke: "为什么小鸡加入了乐队？因为它的鼓点（drumsticks）超棒！"
}
```

---

## 总结

通过时间旅行（Time Travel），你可以：

* 🔁 **回放任意历史状态**，验证模型在不同阶段的推理；
* 🧠 **修改状态并重跑**，探索分支路径；
* 🪶 **实现可追溯的 Agent 执行历史**，帮助优化决策与调试过程。

---

> ✏️ [在 GitHub 上编辑此页面](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/use-time-travel.mdx)
> ⚙️ [将此文档接入 Claude、VSCode 等工具](/use-these-docs)，实现实时智能文档查询。
