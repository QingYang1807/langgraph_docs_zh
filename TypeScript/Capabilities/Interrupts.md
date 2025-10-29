# 中断（Interrupts）

中断功能允许你在图（Graph）执行的任意位置**暂停运行**，等待外部输入后再继续。这使得“人类参与环节”（Human-in-the-loop）成为可能——当流程需要人工确认或输入时，可以安全地暂停。
当触发中断时，LangGraph 会通过其 [持久化层（persistence）](/oss/javascript/langgraph/persistence) 保存当前图的状态，并无限期等待直到你恢复执行。

中断通过调用 `interrupt()` 函数实现，该函数可以在任意节点中调用，并接受任何 **可 JSON 序列化的值** 作为参数。当外部输入准备就绪后，可以通过 `Command` 恢复执行，此时传入的值会作为节点内 `interrupt()` 调用的返回值。

与静态断点不同，中断是**动态**的——可以在任意代码位置设置，甚至可以基于业务逻辑有条件触发。

**关键机制：**

* ✅ **Checkpoint 保持执行位置**：检查点会写入精确的图状态，确保恢复后能接着执行，即使是在错误状态下。
* 🧭 **`thread_id` 是状态指针**：通过 `{ configurable: { thread_id: ... } }` 传入 `invoke` 方法，用于指定恢复的线程状态。
* 💡 **中断输出通过 `__interrupt__` 返回**：`interrupt()` 的返回值会出现在结果对象的 `__interrupt__` 字段中，表示当前图正在等待的输入内容。

你选择的 `thread_id` 就像一个持久化的光标（cursor）：重复使用相同的 ID 表示从同一状态继续执行，而使用新的 ID 则会启动一个全新的线程。

---

## 使用 `interrupt` 暂停执行

[`interrupt`](https://langchain-ai.github.io/langgraphjs/reference/functions/langgraph.interrupt-2.html) 函数会暂停图的执行，并将一个值返回给调用者。当在节点内调用时，LangGraph 会保存当前状态并等待恢复。

要使用 `interrupt`，你需要：

1. **检查点管理器（checkpointer）**：用于保存图的执行状态（生产环境建议使用持久化存储）
2. **线程 ID**：用于告诉运行时要恢复的线程是哪一个
3. **调用 `interrupt()`**：在需要暂停的位置执行（参数必须是可 JSON 序列化的）

```typescript
import { interrupt } from "@langchain/langgraph";

async function approvalNode(state: State) {
  // 暂停执行，等待外部批准
  const approved = interrupt("是否批准此操作？");

  // 当使用 Command({ resume: ... }) 恢复时，返回值将进入该变量
  return { approved };
}
```

调用 `interrupt()` 时，会发生以下步骤：

1. 执行暂停在该行；
2. 图的状态通过 checkpointer 保存；
3. 返回值作为 `__interrupt__` 字段出现在返回结果中；
4. 图进入“等待”状态，直到收到恢复命令；
5. 当恢复执行时，传入的值将作为 `interrupt()` 的返回值。

---

## 恢复中断

当图被中断后，可以通过传入包含 `Command` 的 `invoke` 调用来恢复执行。
传入的值会被返回给节点中的 `interrupt()` 调用，从而继续执行。

```typescript
import { Command } from "@langchain/langgraph";

const config = { configurable: { thread_id: "thread-1" } };

// 首次执行，触发中断
const result = await graph.invoke({ input: "data" }, config);
console.log(result.__interrupt__);
// -> [{ value: '是否批准此操作？' }]

// 恢复执行（resume 值将作为 interrupt() 的返回值）
await graph.invoke(new Command({ resume: true }), config);
```

**注意事项：**

* 必须使用相同的 `thread_id`；
* `Command(resume=...)` 中的值将作为 `interrupt()` 的返回值；
* 节点会从头重新执行，因此中断前的代码会再次运行；
* 恢复值必须是可 JSON 序列化的。

---

## 常见使用模式

中断的核心价值在于“让外部输入接入执行流”，适用于：

* ✅ **审批流程**：执行关键动作前暂停（API 调用、数据库变更、财务操作等）
* ✍️ **人工审核修改**：允许人类在 LLM 输出后进行调整
* 🔧 **工具调用拦截**：在执行工具函数前先暂停审批
* 🛡 **输入验证**：暂停等待人工输入并校验合法性

---

### 示例：审批 / 拒绝流程

```typescript
import { interrupt, Command } from "@langchain/langgraph";

function approvalNode(state: State): Command {
  const isApproved = interrupt({
    question: "是否继续执行？",
    details: state.actionDetails
  });

  return new Command({ goto: isApproved ? "proceed" : "cancel" });
}
```

恢复时：

```typescript
// 批准
await graph.invoke(new Command({ resume: true }), config);

// 拒绝
await graph.invoke(new Command({ resume: false }), config);
```

---

### 示例：人工审核并修改内容

```typescript
import { interrupt } from "@langchain/langgraph";

function reviewNode(state: State) {
  const edited = interrupt({
    instruction: "请审阅并修改以下内容",
    content: state.generatedText
  });
  return { generatedText: edited };
}
```

恢复时传入编辑后的文本：

```typescript
await graph.invoke(new Command({ resume: "修改后的文本" }), config);
```

---

### 示例：工具函数中的中断

```typescript
import { tool } from "@langchain/core/tools";
import { interrupt } from "@langchain/langgraph";
import * as z from "zod";

const sendEmailTool = tool(
  async ({ to, subject, body }) => {
    const response = interrupt({
      action: "send_email",
      to, subject, body,
      message: "确认发送此邮件？",
    });

    if (response?.action === "approve") {
      return `邮件已发送至 ${to}`;
    }
    return "用户取消了发送";
  },
  {
    name: "send_email",
    description: "发送电子邮件",
    schema: z.object({
      to: z.string(),
      subject: z.string(),
      body: z.string(),
    }),
  },
);
```

---

### 示例：验证人类输入

```typescript
import { interrupt } from "@langchain/langgraph";

function getAgeNode(state: State) {
  let prompt = "你的年龄是？";

  while (true) {
    const answer = interrupt(prompt);

    if (typeof answer === "number" && answer > 0) {
      return { age: answer };
    } else {
      prompt = `'${answer}' 不是有效年龄，请输入正整数。`;
    }
  }
}
```

---

## 中断规则与最佳实践

1. **不要在 try/catch 中包裹 `interrupt()`**
   它内部是通过抛出异常来暂停执行的，如果被捕获则无法中断。

2. **不要打乱中断调用顺序**
   多个中断必须保持固定顺序，否则恢复时会出现索引错乱。

3. **只传递简单可序列化的值**
   避免函数、类实例、非 JSON 数据结构。

4. **中断前的副作用操作需幂等（idempotent）**
   因为恢复时节点会从头执行，中断前的操作可能重复运行。

---

## 子图中的中断行为

如果在父图节点中调用子图，且子图包含 `interrupt()`，那么：

* 父图会从调用子图的节点开头重新执行；
* 子图也会从包含 `interrupt()` 的节点重新执行。

---

## 调试与静态断点

除了动态 `interrupt()`，LangGraph 还支持**静态中断**（breakpoint），可用于调试执行流程：

```typescript
const graph = builder.compile({
  interruptBefore: ["node_a"],     // 执行前暂停
  interruptAfter: ["node_b"],      // 执行后暂停
  checkpointer,
});
```

恢复时传入 `null` 即可继续执行到下一个断点。

---

## LangGraph Studio 可视化调试

可以在 [LangGraph Studio](/langsmith/studio) 的 UI 中设置断点并可视化执行状态。
这对于调试复杂的人机协作流程尤其有用。

---

> 🧩 [在 GitHub 上编辑本文档](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/interrupts.mdx)
> ⚡ [可在 Claude、VSCode 等环境中以编程方式连接本页](/use-these-docs)
