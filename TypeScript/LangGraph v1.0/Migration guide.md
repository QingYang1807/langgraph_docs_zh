# LangGraph v1 迁移指南

本指南介绍了 LangGraph v1 的主要变化，并指导你从旧版本迁移。
若想了解完整更新内容，请参阅 [版本发布说明](/oss/javascript/releases/langgraph-v1)。

---

## 升级命令

根据你使用的包管理工具，执行以下任意命令：

```bash
# npm
npm install @langchain/langgraph@latest @langchain/core@latest

# pnpm
pnpm add @langchain/langgraph@latest @langchain/core@latest

# yarn
yarn add @langchain/langgraph@latest @langchain/core@latest

# bun
bun add @langchain/langgraph@latest @langchain/core@latest
```

---

## 变更概览

| 模块区域                        | 变更内容                                                |
| --------------------------- | --------------------------------------------------- |
| React 预构建                   | `createReactAgent` 已弃用，改用 LangChain 的 `createAgent` |
| 中断（Interrupts）              | 通过 `interrupts` 配置支持强类型中断                           |
| `toLangGraphEventStream` 移除 | 使用 `graph.stream` 并指定 `encoding` 格式                 |
| `useStream`                 | 现支持自定义传输层                                           |

---

## 弃用说明：`createReactAgent` → `createAgent`

在 LangGraph v1 中，`createReactAgent` 已被弃用。
推荐使用 LangChain 的 `createAgent` 方法，它基于 LangGraph 实现，并新增了灵活的中间件系统。

参考 LangChain v1 文档以了解更多：

* [版本说明](/oss/javascript/releases/langchain-v1#createagent)
* [迁移指南](/oss/javascript/migrate/langchain-v1#createagent)

### 对比示例

```typescript
// ✅ v1 新写法
import { createAgent } from "langchain";

const agent = createAgent({
  model,
  tools,
  systemPrompt: "You are a helpful assistant.", // [!code highlight]
});
```

```typescript
// ⚠️ v0 旧写法
import { createReactAgent } from "@langchain/langgraph/prebuilts";

const agent = createReactAgent({
  model,
  tools,
  prompt: "You are a helpful assistant.", // [!code highlight]
});
```

---

## 强类型中断（Typed Interrupts）

现在你可以在图构建阶段定义中断类型，从而为传入与返回的中断数据提供严格的类型约束。

```typescript
// ✅ v1 新写法
import { StateGraph, interrupt } from "@langchain/langgraph";
import * as z from "zod";

const State = z.object({ foo: z.string() });

const graphConfig = {
  interrupts: {
    approve: interrupt<{ reason: string }, { messages: string[] }>(),
  },
};

const graph = new StateGraph(State, graphConfig)
  .addNode("node", async (state, runtime) => {
    const value = runtime.interrupt.approve({ reason: "review" }); // [!code highlight]
    return { foo: value };
  })
  .compile();
```

```typescript
// ⚠️ v0 旧写法
import { StateGraph } from "@langchain/langgraph";

const graph = new StateGraph(State)
  .addNode("node", async (state, runtime) => {
    const value = runtime.interrupt.approve({ reason: "review" }); // [!code highlight]
    return state;
  })
  .compile();
```

👉 详情请参阅 [中断机制（Interrupts）](/oss/javascript/langgraph/interrupts)。

---

## 流事件编码（Event Stream Encoding）

低级工具 `toLangGraphEventStream` 已被移除。
流式响应现在由 SDK 直接处理；若使用底层客户端，可通过 `graph.stream` 的 `encoding` 参数选择输出格式。

```typescript
// ✅ v1 新写法
const stream = await graph.stream(input, {
  encoding: "text/event-stream",
  streamMode: ["values", "messages"],
});

return new Response(stream, {
  headers: { "Content-Type": "text/event-stream" }, // [!code highlight]
});
```

```typescript
// ⚠️ v0 旧写法
return toLangGraphEventStreamResponse({
  stream: graph.streamEvents(input, {
    version: "v2",
    streamMode: ["values", "messages"],
  }),
});
```

---

## 重大变更（Breaking Changes）

### 1. 移除 Node.js 18 支持

所有 LangGraph 包现在要求 **Node.js 版本 ≥ 20**。
Node.js 18 已于 2025 年 3 月[正式停止维护](https://nodejs.org/en/about/releases/)。

---

### 2. 新的构建输出机制

所有 LangGraph 包的构建产物已从直接输出 TypeScript 文件改为使用 **打包器（bundler）**。
如果你之前从 `dist/` 目录导入文件（⚠️不推荐），请更新为新的模块系统导入方式。

---

## 附录

✏️ [在 GitHub 上编辑本页源码](https://github.com/langchain-ai/docs/edit/main/src/oss/javascript/migrate/langgraph-v1.mdx)

💻 [通过 MCP 将本文档接入 Claude、VSCode 等工具，实现实时问答](/use-these-docs)
