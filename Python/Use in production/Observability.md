# 可观测性

可观测性对于理解代理在生产环境中的行为至关重要。通过 LangChain 的 [`create_agent`](https://reference.langchain.com/python/langchain/agents/#langchain.agents.create_agent)，您可以通过 [LangSmith](https://smith.langchain.com/) 获得内置的可观测性功能 - 这是一个强大的平台，用于追踪、调试、评估和监控您的 LLM 应用程序。

追踪记录了代理采取的每个步骤，从初始的用户输入到最终响应，包括所有工具调用、模型交互和决策点。这使您能够调试代理、评估性能和监控使用情况。

## 先决条件

开始之前，请确保您具备以下条件：

* 一个 [LangSmith 账户](https://smith.langchain.com/)（注册免费）

## 启用追踪

所有 LangChain 代理都自动支持 LangSmith 追踪。要启用它，请设置以下环境变量：

```bash  theme={null}
export LANGSMITH_TRACING=true
export LANGSMITH_API_KEY=<您的-api-key>
```

<Info>
  您可以从 [LangSmith 设置](https://smith.langchain.com/settings) 获取您的 API 密钥。
</Info>

## 快速开始

无需额外代码即可将追踪记录到 LangSmith。只需像往常一样运行您的代理代码：

```python  theme={null}
from langchain.agents import create_agent


def send_email(to: str, subject: str, body: str):
    """向收件人发送电子邮件。"""
    # ... 电子邮件发送逻辑
    return f"已向 {to} 发送电子邮件"

def search_web(query: str):
    """在网上搜索信息。"""
    # ... 网络搜索逻辑
    return f"搜索结果：{query}"

agent = create_agent(
    model="openai:gpt-4o",
    tools=[send_email, search_web],
    system_prompt="您是一个有用的助手，可以发送电子邮件和搜索网络。"
)

# 运行代理 - 所有步骤将自动被追踪
response = agent.invoke({
    "messages": [{"role": "user", "content": "搜索最新的 AI 新闻并发送摘要到 john@example.com"}]
})
```

默认情况下，追踪将记录到名称为 `default` 的项目中。要配置自定义项目名称，请参阅 [记录到项目](#记录到项目)。

## 选择性追踪

您可以使用 LangSmith 的 `tracing_context` 上下文管理器选择追踪特定的调用或应用程序部分：

```python  theme={null}
import langsmith as ls

# 这将被追踪
with ls.tracing_context(enabled=True):
    agent.invoke({"messages": [{"role": "user", "content": "发送测试邮件到 alice@example.com"}]})

# 这不会被追踪（如果未设置 LANGSMITH_TRACING）
agent.invoke({"messages": [{"role": "user", "content": "发送另一封邮件"}]})
```

## 记录到项目

<Accordion title="静态方式">
  您可以通过设置 `LANGSMITH_PROJECT` 环境变量为整个应用程序设置自定义项目名称：

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
          "messages": [{"role": "user", "content": "发送欢迎邮件"}]
      })
  ```
</Accordion>

## 向追踪添加元数据

您可以使用自定义元数据和标签注释您的追踪：

```python  theme={null}
response = agent.invoke(
    {"messages": [{"role": "user", "content": "发送欢迎邮件"}]},
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

`tracing_context` 也接受标签和元数据以实现细粒度控制：

```python  theme={null}
with ls.tracing_context(
    project_name="email-agent-test",
    enabled=True,
    tags=["production", "email-assistant", "v1.0"],
    metadata={"user_id": "user_123", "session_id": "session_456", "environment": "production"}):
    response = agent.invoke(
        {"messages": [{"role": "user", "content": "发送欢迎邮件"}]}
    )
```

这些自定义元数据和标签将附加到 LangSmith 中的追踪记录上。

<Tip>
  要了解有关如何使用追踪来调试、评估和监控代理的更多信息，请参阅 [LangSmith 文档](/langsmith/home)。
</Tip>

***

<Callout icon="pen-to-square" iconType="regular">
  [在 GitHub 上编辑此页面的源代码。](https://github.com/langchain-ai/docs/edit/main/src/oss/langchain/observability.mdx)
</Callout>

<Tip icon="terminal" iconType="regular">
  [通过 MCP 以编程方式连接这些文档](/use-these-docs) 到 Claude、VSCode 等，获取实时答案。
</Tip>