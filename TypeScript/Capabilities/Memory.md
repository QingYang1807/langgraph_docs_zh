# 记忆（Memory）

AI 应用需要具备 **记忆（Memory）** 才能在多轮交互中共享上下文。
在 **LangGraph** 中，你可以添加两种类型的记忆：

* [添加短期记忆](#添加短期记忆)：作为智能体状态（[state](/oss/javascript/langgraph/graph-api#state)）的一部分，用于支持多轮对话。
* [添加长期记忆](#添加长期记忆)：在会话之间持久化存储用户或应用级别的数据。

---

## 添加短期记忆（Short-term Memory）

**短期记忆**（线程级持久化，[persistence](/oss/javascript/langgraph/persistence)）允许智能体跟踪多轮对话的上下文。

示例：

```typescript
import { MemorySaver, StateGraph } from "@langchain/langgraph";

const checkpointer = new MemorySaver();

const builder = new StateGraph(...);
const graph = builder.compile({ checkpointer });

await graph.invoke(
  { messages: [{ role: "user", content: "hi! i am Bob" }] },
  { configurable: { thread_id: "1" } }
);
```

---

### 在生产环境中使用

在生产环境中，建议使用数据库持久化检查点：

```typescript
import { PostgresSaver } from "@langchain/langgraph-checkpoint-postgres";

const DB_URI = "postgresql://postgres:postgres@localhost:5442/postgres?sslmode=disable";
const checkpointer = PostgresSaver.fromConnString(DB_URI);

const builder = new StateGraph(...);
const graph = builder.compile({ checkpointer });
```

📦 安装依赖：

```bash
npm install @langchain/langgraph-checkpoint-postgres
```

> ⚙️ 第一次使用 PostgreSQL 检查点时，需要执行 `checkpointer.setup()` 初始化数据库。

**完整示例：**

```typescript
import { ChatAnthropic } from "@langchain/anthropic";
import { StateGraph, MessagesZodMeta, START } from "@langchain/langgraph";
import { BaseMessage } from "@langchain/core/messages";
import { registry } from "@langchain/langgraph/zod";
import * as z from "zod";
import { PostgresSaver } from "@langchain/langgraph-checkpoint-postgres";

const MessagesZodState = z.object({
  messages: z
    .array(z.custom<BaseMessage>())
    .register(registry, MessagesZodMeta),
});

const model = new ChatAnthropic({ model: "claude-3-5-haiku-20241022" });
const DB_URI = "postgresql://postgres:postgres@localhost:5442/postgres?sslmode=disable";
const checkpointer = PostgresSaver.fromConnString(DB_URI);

const builder = new StateGraph(MessagesZodState)
  .addNode("call_model", async (state) => {
    const response = await model.invoke(state.messages);
    return { messages: [response] };
  })
  .addEdge(START, "call_model");

const graph = builder.compile({ checkpointer });

const config = { configurable: { thread_id: "1" } };

for await (const chunk of await graph.stream(
  { messages: [{ role: "user", content: "hi! I'm bob" }] },
  { ...config, streamMode: "values" }
)) {
  console.log(chunk.messages.at(-1)?.content);
}

for await (const chunk of await graph.stream(
  { messages: [{ role: "user", content: "what's my name?" }] },
  { ...config, streamMode: "values" }
)) {
  console.log(chunk.messages.at(-1)?.content);
}
```

---

### 在子图（Subgraph）中使用记忆

如果主图中包含子图，**只需要在父图编译时提供 `checkpointer`**。
LangGraph 会自动将其传递给所有子图。

```typescript
import { StateGraph, START, MemorySaver } from "@langchain/langgraph";
import * as z from "zod";

const State = z.object({ foo: z.string() });

const subgraphBuilder = new StateGraph(State)
  .addNode("subgraph_node_1", (state) => ({ foo: state.foo + "bar" }))
  .addEdge(START, "subgraph_node_1");

const subgraph = subgraphBuilder.compile();

const builder = new StateGraph(State)
  .addNode("node_1", subgraph)
  .addEdge(START, "node_1");

const checkpointer = new MemorySaver();
const graph = builder.compile({ checkpointer });
```

如果希望子图拥有独立记忆（例如在多智能体系统中），可以单独为子图配置：

```typescript
const subgraph = subgraphBuilder.compile({ checkpointer: true });
```

---

## 添加长期记忆（Long-term Memory）

**长期记忆**用于存储用户或应用的长期信息，例如用户偏好、身份或历史行为。

```typescript
import { InMemoryStore, StateGraph } from "@langchain/langgraph";

const store = new InMemoryStore();

const builder = new StateGraph(...);
const graph = builder.compile({ store });
```

---

### 在生产环境中使用

推荐使用 PostgreSQL 数据库存储长期记忆：

```typescript
import { PostgresStore } from "@langchain/langgraph-checkpoint-postgres/store";

const DB_URI = "postgresql://postgres:postgres@localhost:5442/postgres?sslmode=disable";
const store = PostgresStore.fromConnString(DB_URI);

const builder = new StateGraph(...);
const graph = builder.compile({ store });
```

> ⚙️ 第一次使用时需执行 `store.setup()` 初始化数据库结构。

---

### 使用语义搜索增强长期记忆

你可以在记忆存储中开启 **语义搜索（semantic search）**，
允许智能体根据语义相似度检索信息，而非仅靠关键词匹配。

```typescript
import { OpenAIEmbeddings } from "@langchain/openai";
import { InMemoryStore } from "@langchain/langgraph";

const embeddings = new OpenAIEmbeddings({ model: "text-embedding-3-small" });
const store = new InMemoryStore({
  index: { embeddings, dims: 1536 },
});

await store.put(["user_123", "memories"], "1", { text: "I love pizza" });
await store.put(["user_123", "memories"], "2", { text: "I am a plumber" });

const items = await store.search(["user_123", "memories"], {
  query: "I'm hungry",
  limit: 1,
});
```

---

## 管理短期记忆（Managing Short-term Memory）

在多轮长对话中，短期记忆可能超出 LLM 的上下文窗口。
常见策略包括：

* ✂️ [截断消息](#截断消息)：移除最早或最新的若干条消息；
* 🗑️ [删除消息](#删除消息)：永久删除部分历史；
* 🧾 [摘要消息](#摘要消息)：用摘要替代早期对话；
* 🧩 [管理检查点](#管理检查点)：查看、存储或删除历史状态；
* ⚙️ 自定义策略（如消息过滤等）。

---

### 截断消息（Trim Messages）

多数 LLM 对上下文窗口（token 数）有限制。
可以使用 `trimMessages()` 工具自动裁剪：

```typescript
import { trimMessages } from "@langchain/core/messages";

const callModel = async (state) => {
  const messages = trimMessages(state.messages, {
    strategy: "last",
    maxTokens: 128,
    startOn: "human",
    endOn: ["human", "tool"],
  });
  const response = await model.invoke(messages);
  return { messages: [response] };
};
```

---

### 删除消息（Delete Messages）

通过 `RemoveMessage` 从状态中移除指定消息：

```typescript
import { RemoveMessage } from "@langchain/core/messages";

const deleteMessages = (state) => {
  const messages = state.messages;
  if (messages.length > 2) {
    return {
      messages: messages
        .slice(0, 2)
        .map((m) => new RemoveMessage({ id: m.id })),
    };
  }
};
```

⚠️ **注意事项：**

* 确保删除后的对话仍符合 LLM 历史格式；
* 某些模型要求历史以 `user` 消息开头；
* 对含有 `tool` 调用的对话需保持配对。

---

### 摘要消息（Summarize Messages）

相比直接删除或裁剪，**摘要策略**能保留关键信息并减少上下文长度。

示例：

```typescript
import { RemoveMessage, HumanMessage } from "@langchain/core/messages";

const summarizeConversation = async (state) => {
  const summary = state.summary || "";
  const summaryMessage = summary
    ? `This is a summary of the conversation so far: ${summary}\n\nExtend the summary based on new messages above:`
    : "Summarize the conversation above:";
  const messages = [...state.messages, new HumanMessage({ content: summaryMessage })];
  const response = await model.invoke(messages);

  const deleteMessages = state.messages
    .slice(0, -2)
    .map((m) => new RemoveMessage({ id: m.id }));

  return { summary: response.content, messages: deleteMessages };
};
```

---

### 管理检查点（Checkpoints）

查看、历史记录与删除操作：

```typescript
// 查看当前线程的最新状态
await graph.getState({
  configurable: { thread_id: "1" },
});

// 遍历线程的历史记录
const history = [];
for await (const state of graph.getStateHistory({ configurable: { thread_id: "1" } })) {
  history.push(state);
}

// 删除线程的所有检查点
await checkpointer.deleteThread("1");
```

---

> ✏️ [在 GitHub 上编辑此页面](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/add-memory.mdx)
> ⚙️ [将文档接入 Claude、VSCode 等工具](/use-these-docs)，实现实时智能联动。
