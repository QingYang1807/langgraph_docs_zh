# Studio

本指南将指导您如何使用 **Studio** 在本地可视化、交互和调试您的代理。

Studio 是我们免费且强大的代理 IDE，它与 [LangSmith](/langsmith/home) 集成，支持追踪、评估和提示工程。您可以精确地看到您的代理如何思考，追踪每个决策，并交付更智能、更可靠的代理。

<Frame>
  <iframe className="w-full aspect-video rounded-xl" src="https://www.youtube.com/embed/Mi1gSlHwZLM?si=zA47TNuTC5aH0ahd" title="Studio" frameBorder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowFullScreen />
</Frame>

## 先决条件

开始之前，请确保您已具备以下条件：

* [LangSmith](https://smith.langchain.com/settings) 的 API 密钥（免费注册）

## 设置本地 LangGraph 服务器

### 1. 安装 LangGraph CLI

```shell  theme={null}
# 需要 Python >= 3.11
pip install --upgrade "langgraph-cli[inmem]"
```

### 2. 准备您的代理

我们将使用以下简单代理作为示例：

```python title="agent.py" theme={null}
from langchain.agents import create_agent

def send_email(to: str, subject: str, body: str):
    """发送邮件"""
    email = {
        "to": to,
        "subject": subject,
        "body": body
    }
    # ... 邮件发送逻辑

    return f"邮件已发送给 {to}"

agent = create_agent(
    "openai:gpt-4o",
    tools=[send_email],
    system_prompt="您是一个邮件助手。始终使用 send_email 工具。",
)
```

### 3. 环境变量

在您的项目根目录创建一个 `.env` 文件并填写必要的 API 密钥。我们需要将 `LANGSMITH_API_KEY` 环境变量设置为从 [LangSmith](https://smith.langchain.com/settings) 获取的 API 密钥。

<Warning>
  请确保不要将 `.env` 提交到 Git 等版本控制系统！
</Warning>

```bash .env theme={null}
LANGSMITH_API_KEY=lsv2...
```

### 4. 创建 LangGraph 配置文件

在您的应用目录中，创建一个配置文件 `langgraph.json`：

```json title="langgraph.json" theme={null}
{
  "dependencies": ["."],
  "graphs": {
    "agent": "./src/agent.py:agent"
  },
  "env": ".env"
}
```

[`create_agent`](https://reference.langchain.com/python/langchain/agents/#langchain.agents.create_agent) 自动返回一个编译后的 LangGraph 图，我们可以将其传递给配置文件中的 `graphs` 键。

<Info>
  有关配置文件 JSON 对象中每个键的详细解释，请参阅 [LangGraph 配置文件参考](/langsmith/cli#configuration-file)。
</Info>

到目前为止，我们的项目结构如下：

```bash  theme={null}
my-app/
├── src
│   └── agent.py
├── .env
└── langgraph.json
```

### 5. 安装依赖项

在新的 LangGraph 应用根目录中，安装依赖项：

<CodeGroup>
  ```shell pip theme={null}
  pip install -e .
  ```

  ```shell uv theme={null}
  uv sync
  ```
</CodeGroup>

### 6. 在 Studio 中查看您的代理

启动您的 LangGraph 服务器：

```shell  theme={null}
langgraph dev
```

<Warning>
  Safari 会阻止到 Studio 的 `localhost` 连接。要解决此问题，请使用 `--tunnel` 运行上述命令，通过安全隧道访问 Studio。
</Warning>

您的代理可以通过 API (`http://127.0.0.1:2024`) 和 Studio UI `https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:2024` 访问：

<Frame>
    <img src="https://mintcdn.com/langchain-5e9cc07a/TCDks4pdsHdxWmuJ/oss/images/studio_create-agent.png?fit=max&auto=format&n=TCDks4pdsHdxWmuJ&q=85&s=ebd259e9fa24af7d011dfcc568f74be2" alt="Studio UI 中的代理视图" data-og-width="2836" width="2836" data-og-height="1752" height="1752" data-path="oss/images/studio_create-agent.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/TCDks4pdsHdxWmuJ/oss/images/studio_create-agent.png?w=280&fit=max&auto=format&n=TCDks4pdsHdxWmuJ&q=85&s=cf9c05bdd08661d4d546c540c7a28cbe 280w, https://mintcdn.com/langchain-5e9cc07a/TCDks4pdsHdxWmuJ/oss/images/studio_create-agent.png?w=560&fit=max&auto=format&n=TCDks4pdsHdxWmuJ&q=85&s=484b2fd56957d048bd89280ce97065a0 560w, https://mintcdn.com/langchain-5e9cc07a/TCDks4pdsHdxWmuJ/oss/images/studio_create-agent.png?w=840&fit=max&auto=format&n=TCDks4pdsHdxWmuJ&q=85&s=92991302ac24604022ab82ac22729f68 840w, https://mintcdn.com/langchain-5e9cc07a/TCDks4pdsHdxWmuJ/oss/images/studio_create-agent.png?w=1100&fit=max&auto=format&n=TCDks4pdsHdxWmuJ&q=85&s=ed366abe8dabc42a9d7c300a591e1614 1100w, https://mintcdn.com/langchain-5e9cc07a/TCDks4pdsHdxWmuJ/oss/images/studio_create-agent.png?w=1650&fit=max&auto=format&n=TCDks4pdsHdxWmuJ&q=85&s=d5865d3c4b0d26e9d72e50d474547a63 1650w, https://mintcdn.com/langchain-5e9cc07a/TCDks4pdsHdxWmuJ/oss/images/studio_create-agent.png?w=2500&fit=max&auto=format&n=TCDks4pdsHdxWmuJ&q=85&s=6b254add2df9cc3c10ac0c2bcb3a589c 2500w" />
</Frame>

Studio 使代理的每个步骤都易于观察。重放任何输入并检查确切的提示、工具参数、返回值以及标记/延迟指标。如果工具抛出异常，Studio 会记录异常及其周围状态，从而减少您的调试时间。

保持开发服务器运行，编辑提示或工具签名，并观察 Studio 的热重载。从任何步骤重新运行对话线程以验证行为更改。有关更多详细信息，请参阅[管理线程](/langsmith/use-studio#edit-thread-history)。

随着代理的增长，相同的视图可以从单工具演示扩展到多节点图，使决策保持清晰和可重现。

<Tip>
  要深入了解 Studio，请查看[概述页面](/langsmith/studio)。
</Tip>

<Tip>
  有关本地和已部署代理的更多信息，请参阅[设置本地 LangGraph 服务器](/oss/python/langchain/studio#setup-local-langgraph-server)和[部署](/oss/python/langchain/deploy)。
</Tip>

***

<Callout icon="pen-to-square" iconType="regular">
  [在 GitHub 上编辑此页面的源代码。](https://github.com/langchain-ai/docs/edit/main/src/oss/langchain/studio.mdx)
</Callout>

<Tip icon="terminal" iconType="regular">
  [通过 MCP 以编程方式连接这些文档](/use-these-docs)，与 Claude、VSCode 等连接，获取实时答案。
</Tip>