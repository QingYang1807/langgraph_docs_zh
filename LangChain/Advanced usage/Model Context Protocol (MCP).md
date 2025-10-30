# 模型上下文协议 (MCP)

[模型上下文协议 (MCP)](https://modelcontextprotocol.io/introduction) 是一个开放协议，标准化了应用程序向大语言模型(LLM)提供工具和上下文的方式。LangChain 代理可以使用 [`langchain-mcp-adapters`](https://github.com/langchain-ai/langchain-mcp-adapters) 库来使用在 MCP 服务器上定义的工具。

## 安装

要在 LangGraph 中使用 MCP 工具，请安装 `langchain-mcp-adapters` 库：

<CodeGroup>
  ```bash pip theme={null}
  pip install langchain-mcp-adapters
  ```

  ```bash uv theme={null}
  uv add langchain-mcp-adapters
  ```
</CodeGroup>

## 传输类型

MCP 支持不同的传输机制用于客户端-服务器通信：

* **stdio** – 客户端将服务器作为子进程启动并通过标准输入/输出进行通信。最适合本地工具和简单设置。
* **Streamable HTTP** – 服务器作为独立进程运行，处理 HTTP 请求。支持远程连接和多个客户端。
* **Server-Sent Events (SSE)** – 可流式 HTTP 的一个变体，针对实时流式通信进行了优化。

## 使用 MCP 工具

`langchain-mcp-adapters` 使代理能够使用在一个或多个 MCP 服务器上定义的工具。

```python 访问多个 MCP 服务器 icon="server" theme={null}
from langchain_mcp_adapters.client import MultiServerMCPClient  # [!code highlight]
from langchain.agents import create_agent


client = MultiServerMCPClient(  # [!code highlight]
    {
        "math": {
            "transport": "stdio",  # 本地子进程通信
            "command": "python",
            # math_server.py 文件的绝对路径
            "args": ["/path/to/math_server.py"],
        },
        "weather": {
            "transport": "streamable_http",  # 基于 HTTP 的远程服务器
            # 确保您的天气服务器在端口 8000 上启动
            "url": "http://localhost:8000/mcp",
        }
    }
)

tools = await client.get_tools()  # [!code highlight]
agent = create_agent(
    "anthropic:claude-sonnet-4-5",
    tools  # [!code highlight]
)
math_response = await agent.ainvoke(
    {"messages": [{"role": "user", "content": "what's (3 + 5) x 12?"}]}
)
weather_response = await agent.ainvoke(
    {"messages": [{"role": "user", "content": "what is the weather in nyc?"}]}
)
```

<Note>
  `MultiServerMCPClient` 默认是**无状态的**。每次工具调用都会创建一个新的 MCP `ClientSession`，执行工具，然后清理。
</Note>

## 自定义 MCP 服务器

要创建您自己的 MCP 服务器，可以使用 `mcp` 库。该库提供了一种简单的方式来定义[工具](https://modelcontextprotocol.io/docs/learn/server-concepts#tools-ai-actions)并将它们作为服务器运行。

<CodeGroup>
  ```bash pip theme={null}
  pip install mcp
  ```

  ```bash uv theme={null}
  uv add mcp
  ```
</CodeGroup>

使用以下参考实现来测试您的代理与 MCP 工具服务器。

```python title="数学服务器 (stdio 传输)" icon="floppy-disk" theme={null}
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("Math")

@mcp.tool()
def add(a: int, b: int) -> int:
    """Add two numbers"""
    return a + b

@mcp.tool()
def multiply(a: int, b: int) -> int:
    """Multiply two numbers"""
    return a * b

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

```python title="天气服务器 (可流式 HTTP 传输)" icon="wifi" theme={null}
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("Weather")

@mcp.tool()
async def get_weather(location: str) -> str:
    """Get weather for location."""
    return "It's always sunny in New York"

if __name__ == "__main__":
    mcp.run(transport="streamable-http")
```

## 状态化工具使用

对于在工具调用之间保持上下文的状态化服务器，使用 `client.session()` 创建持久的 `ClientSession`。

```python 使用 MCP ClientSession 进行状态化工具使用 theme={null}
from langchain_mcp_adapters.tools import load_mcp_tools

client = MultiServerMCPClient({...})
async with client.session("math") as session:
    tools = await load_mcp_tools(session)
```

## 其他资源

* [MCP 文档](https://modelcontextprotocol.io/introduction)
* [MCP 传输文档](https://modelcontextprotocol.io/docs/concepts/transports)
* [`langchain-mcp-adapters`](https://github.com/langchain-ai/langchain-mcp-adapters)

***

<Callout icon="pen-to-square" iconType="regular">
  [在 GitHub 上编辑此页面的源代码。](https://github.com/langchain-ai/docs/edit/main/src/oss/langchain/mcp.mdx)
</Callout>

<Tip icon="terminal" iconType="regular">
  [通过 MCP 以编程方式连接这些文档](/use-these-docs) 到 Claude、VSCode 等，获取实时答案。
</Tip>