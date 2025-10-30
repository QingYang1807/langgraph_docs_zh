# 可观测性

跟踪（Traces）是您的应用程序从输入到输出所采取的一系列步骤。这些单独的步骤中的每一步都由一次运行（run）表示。您可以使用[LangSmith](https://smith.langchain.com/)来可视化这些执行步骤。要使用它，请[为您的应用程序启用跟踪](/langsmith/trace-with-langgraph)。这使您能够执行以下操作：

* [调试本地运行的应用程序](/langsmith/observability-studio#debug-langsmith-traces)
* [评估应用程序性能](/oss/python/langchain/evals)
* [监控应用程序](/langsmith/dashboards)

## 先决条件

在开始之前，请确保您已具备以下条件：

* 一个[LangSmith账户](https://smith.langchain.com/)（免费注册）

## 启用跟踪

要为您的应用程序启用跟踪，请设置以下环境变量：

```python  theme={null}
export LANGSMITH_TRACING=true
export LANGSMITH_API_KEY=<your-api-key>
```

默认情况下，跟踪将记录到名为`default`的项目中。要配置自定义项目名称，请参阅[记录到项目](#log-to-a-project)。

有关更多信息，请参阅[使用LangGraph进行跟踪](/langsmith/trace-with-langgraph)。

## 选择性跟踪

您可以选择使用LangSmith的`tracing_context`上下文管理器来跟踪特定的调用或应用程序部分：

```python  theme={null}
import langsmith as ls

# 这部分将被跟踪
with ls.tracing_context(enabled=True):
    agent.invoke({"messages": [{"role": "user", "content": "Send a test email to alice@example.com"}]})

# 这部分将不会被跟踪（如果未设置LANGSMITH_TRACING）
agent.invoke({"messages": [{"role": "user", "content": "Send another email"}]})
```

## 记录到项目

<Accordion title="静态方式">
  您可以通过设置`LANGSMITH_PROJECT`环境变量为整个应用程序设置自定义项目名称：

  ```bash  theme={null}
  export LANGSMITH_PROJECT=my-agent-project
  ```
</Accordion>

<Accordion title="动态方式">
  您可以为特定操作以编程方式设置项目名称：

  ```python  theme={null}
  import langsmith as ls

  with ls.tracing_context(project_name="email-agent-test", enabled=True):
      response = agent.invoke({
          "messages": [{"role": "user", "content": "Send a welcome email"}]
      })
  ```
</Accordion>

## 向跟踪添加元数据

您可以使用自定义元数据和标签来注释您的跟踪：

```python  theme={null}
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

`tracing_context`也接受标签和元数据以进行细粒度控制：

```python  theme={null}
with ls.tracing_context(
    project_name="email-agent-test",
    enabled=True,
    tags=["production", "email-assistant", "v1.0"],
    metadata={"user_id": "user_123", "session_id": "session_456", "environment": "production"}):
    response = agent.invoke(
        {"messages": [{"role": "user", "content": "Send a welcome email"}]}
    )
```

这些自定义元数据和标签将附加到LangSmith中的跟踪记录上。

<Tip>
  要了解如何使用跟踪来调试、评估和监控您的代理，请参阅[LangSmith文档](/langsmith/home)。
</Tip>

## 使用匿名化器防止敏感数据被记录在跟踪中

您可能希望屏蔽敏感数据，防止其被记录到LangSmith中。您可以创建[匿名化器](/langsmith/mask-inputs-outputs#rule-based-masking-of-inputs-and-outputs)并通过配置将其应用到您的图中。此示例将从发送到LangSmith的跟踪中编辑任何匹配社会保障号码格式XXX-XX-XXXX的内容。

```python Python theme={null}
from langchain_core.tracers.langchain import LangChainTracer
from langgraph.graph import StateGraph, MessagesState
from langsmith import Client
from langsmith.anonymizer import create_anonymizer

anonymizer = create_anonymizer([
    # 匹配社会保障号码
    { "pattern": r"\b\d{3}-?\d{2}-?\d{4}\b", "replace": "<ssn>" }
])

tracer_client = Client(anonymizer=anonymizer)
tracer = LangChainTracer(client=tracer_client)
# 定义图
graph = (
    StateGraph(MessagesState)
    ...
    .compile()
    .with_config({'callbacks': [tracer]})
)
```

***

<Callout icon="pen-to-square" iconType="regular">
  [在GitHub上编辑此页面的源代码。](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/observability.mdx)
</Callout>

<Tip icon="terminal" iconType="regular">
  [通过MCP以编程方式连接这些文档](/use-these-docs)到Claude、VSCode等，以获取实时答案。
</Tip>