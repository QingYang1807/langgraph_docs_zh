以下是这篇文档的流畅中文翻译👇

---

# 可观测性（Observability）

**Trace（追踪）** 是指应用程序从输入到输出过程中执行的一系列步骤。
每一个独立的执行步骤被称为一个 **run（运行）**。
你可以使用 [LangSmith](https://smith.langchain.com/) 来可视化这些执行步骤。

启用追踪后（见：[在应用中启用追踪](/langsmith/trace-with-langgraph)），你可以做到以下几件事：

* [调试本地运行的应用程序](/langsmith/observability-studio#debug-langsmith-traces)
* [评估应用程序性能](/oss/javascript/langchain/evals)
* [监控应用运行状态](/langsmith/dashboards)

---

## 前置条件

在开始之前，请确保你具备以下条件：

* 一个 [LangSmith 账户](https://smith.langchain.com/)（免费注册）

---

## 启用追踪

要为应用程序启用追踪，请设置以下环境变量：

```bash
export LANGSMITH_TRACING=true
export LANGSMITH_API_KEY=<your-api-key>
```

默认情况下，追踪数据将被记录到名为 `default` 的项目中。
如果想自定义项目名称，请参考：[记录到指定项目](#log-to-a-project)。

更多信息可参考：[使用 LangGraph 进行追踪](/langsmith/trace-with-langgraph)。

---

## 按需追踪（选择性开启）

你也可以使用 LangSmith 的 `tracing_context` 上下文管理器，仅对特定调用或应用部分启用追踪：

```python
import langsmith as ls

# 这个调用将被追踪
with ls.tracing_context(enabled=True):
    agent.invoke({"messages": [{"role": "user", "content": "Send a test email to alice@example.com"}]})

# 如果 LANGSMITH_TRACING 未设置，这个调用将不会被追踪
agent.invoke({"messages": [{"role": "user", "content": "Send another email"}]})
```

---

## 记录到指定项目

### 静态方式

为整个应用设置自定义项目名：

```bash
export LANGSMITH_PROJECT=my-agent-project
```

### 动态方式

你也可以在代码中动态指定项目名，仅对某些操作生效：

```python
import langsmith as ls

with ls.tracing_context(project_name="email-agent-test", enabled=True):
    response = agent.invoke({
        "messages": [{"role": "user", "content": "Send a welcome email"}]
    })
```

---

## 为追踪添加元数据

你可以为追踪附加自定义 **元数据（metadata）** 和 **标签（tags）**：

```python
response = agent.invoke(
    {"messages": [{"role": "user", "content": "Send a welcome email"}]},
    config={
        "tags": ["production", "email-assistant", "v1.0"],
        "metadata": {
            "user_id": "user_123",
            "session_id": "session_456",
            "environment": "production"
        }
    }
)
```

`tracing_context` 同样支持传入标签和元数据：

```python
with ls.tracing_context(
    project_name="email-agent-test",
    enabled=True,
    tags=["production", "email-assistant", "v1.0"],
    metadata={"user_id": "user_123", "session_id": "session_456", "environment": "production"}
):
    response = agent.invoke({"messages": [{"role": "user", "content": "Send a welcome email"}]})
```

这些自定义标签与元数据会被附加到 LangSmith 中的追踪记录上。

> 💡 想了解更多关于如何利用追踪进行调试、评估与监控，请参阅 [LangSmith 文档](/langsmith/home)。

---

## 使用匿名化工具（anonymizer）防止敏感数据被记录

在某些情况下，你可能希望在追踪中屏蔽敏感信息。
你可以创建 [anonymizer（匿名化器）](/langsmith/mask-inputs-outputs#rule-based-masking-of-inputs-and-outputs)，并在配置中将其应用到图中。

例如，以下示例会将符合美国社会安全号（SSN）格式 `XXX-XX-XXXX` 的内容替换为 `<ssn>`：

```typescript
import { StateGraph } from "@langchain/langgraph";
import { LangChainTracer } from "@langchain/core/tracers/tracer_langchain";
import { StateAnnotation } from "./state.js";
import { createAnonymizer } from "langsmith/anonymizer";
import { Client } from "langsmith";

const anonymizer = createAnonymizer([
    // 匹配 SSN 格式
    { pattern: /\b\d{3}-?\d{2}-?\d{4}\b/, replace: "<ssn>" }
]);

const langsmithClient = new Client({ anonymizer });
const tracer = new LangChainTracer({ client: langsmithClient });

export const graph = new StateGraph(StateAnnotation)
  .compile()
  .withConfig({
    callbacks: [tracer],
});
```

---

✏️ [在 GitHub 上编辑此页面源码](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/observability.mdx)

💡 [将本文档通过 MCP 连接到 Claude、VSCode 等工具](/use-these-docs)，实现实时文档查询。

---

是否希望我帮你把这篇翻译润色成正式的中文开发文档格式（比如加上标题层次、代码注释和小节摘要）？这样你可以直接放到你项目的文档系统里。
