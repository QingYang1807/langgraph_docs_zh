# 流式传输（Streaming）

LangChain 实现了一套用于实时更新的流式传输系统。

流式传输对于提升基于大语言模型（LLM）的应用响应速度至关重要。通过在完整响应生成之前逐步展示输出内容，流式传输可以显著改善用户体验（UX），尤其在面对模型推理延迟时效果显著。

---

## 概述

LangChain 的流式系统可以让你的应用实时获取智能体（Agent）的执行反馈。

LangChain 的流式传输能做到以下几点：

* 🧠 **[流式展示智能体进度](#智能体进度)** —— 每当智能体执行一个步骤时，就会发送状态更新。
* 🔢 **[流式输出 LLM token](#llm-token)** —— 随着语言模型生成 token，实时输出。
* 📊 **[流式输出自定义更新](#自定义更新)** —— 发出自定义信号（例如：“已获取 10/100 条记录”）。
* 🧩 **[多模式流式传输](#多模式流式传输)** —— 支持 `updates`（进度更新）、`messages`（LLM token + 元数据）、`custom`（用户自定义数据）等多种模式。

---

## 智能体进度

若要流式传输智能体进度，可使用
[`stream`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.CompiledStateGraph.stream)
或
[`astream`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.CompiledStateGraph.astream)
方法，并设置 `stream_mode="updates"`。这会在每个步骤执行后发出事件。

例如，一个仅调用一次工具的智能体将产生如下更新：

* **LLM 节点**：[`AIMessage`](https://reference.langchain.com/python/langchain/messages/#langchain.messages.AIMessage)，表示发起工具调用请求
* **Tool 节点**：[`ToolMessage`](https://reference.langchain.com/python/langchain/messages/#langchain.messages.ToolMessage)，表示工具执行结果
* **LLM 节点**：最终生成的 AI 回复

```python title="流式展示智能体进度"
from langchain.agents import create_agent

def get_weather(city: str) -> str:
    """获取指定城市的天气"""
    return f"{city}的天气永远晴朗！"

agent = create_agent(
    model="openai:gpt-5-nano",
    tools=[get_weather],
)

for chunk in agent.stream(
    {"messages": [{"role": "user", "content": "旧金山的天气如何？"}]},
    stream_mode="updates",
):
    for step, data in chunk.items():
        print(f"步骤: {step}")
        print(f"内容: {data['messages'][-1].content_blocks}")
```

输出示例：

```shell
步骤: model
内容: [{'type': 'tool_call', 'name': 'get_weather', 'args': {'city': 'San Francisco'}}]

步骤: tools
内容: [{'type': 'text', 'text': "It's always sunny in San Francisco!"}]

步骤: model
内容: [{'type': 'text', 'text': "It's always sunny in San Francisco!"}]
```

---

## LLM Token

如果希望随着 LLM 生成 token 实时接收输出，可使用 `stream_mode="messages"`。

以下示例展示了流式输出工具调用过程及最终回复：

```python title="流式输出 LLM token"
from langchain.agents import create_agent

def get_weather(city: str) -> str:
    return f"It's always sunny in {city}!"

agent = create_agent(
    model="openai:gpt-5-nano",
    tools=[get_weather],
)

for token, metadata in agent.stream(
    {"messages": [{"role": "user", "content": "What is the weather in SF?"}]},
    stream_mode="messages",
):
    print(f"节点: {metadata['langgraph_node']}")
    print(f"内容: {token.content_blocks}\n")
```

输出结果（节选）：

```shell
node: model
content: [{'type': 'tool_call_chunk', 'name': 'get_weather', 'args': '{"city":"San Francisco"}'}]
...
node: tools
content: [{'type': 'text', 'text': "It's always sunny in San Francisco!"}]
...
node: model
content: [{'type': 'text', 'text': 'San Francisco weather: It\'s always sunny in San Francisco!'}]
```

---

## 自定义更新

若要从工具中流式输出自定义信息，可使用
[`get_stream_writer`](https://reference.langchain.com/python/langgraph/config/#langgraph.config.get_stream_writer)。

```python title="流式输出自定义更新"
from langchain.agents import create_agent
from langgraph.config import get_stream_writer

def get_weather(city: str) -> str:
    writer = get_stream_writer()
    writer(f"正在查找 {city} 的数据...")
    writer(f"已获取 {city} 的数据")
    return f"{city}天气永远晴朗！"

agent = create_agent(
    model="anthropic:claude-sonnet-4-5",
    tools=[get_weather],
)

for chunk in agent.stream(
    {"messages": [{"role": "user", "content": "旧金山的天气如何？"}]},
    stream_mode="custom"
):
    print(chunk)
```

输出示例：

```shell
正在查找 San Francisco 的数据...
已获取 San Francisco 的数据
```

> 💡 注意：
> 如果在工具函数中使用了 `get_stream_writer()`，则该工具无法在 LangGraph 执行上下文之外调用。

---

## 多模式流式传输

可以通过传入列表指定多种流模式，例如：
`stream_mode=["updates", "custom"]`

```python title="多模式流式传输"
from langchain.agents import create_agent
from langgraph.config import get_stream_writer

def get_weather(city: str) -> str:
    writer = get_stream_writer()
    writer(f"正在查找 {city} 的数据...")
    writer(f"已获取 {city} 的数据")
    return f"{city}天气永远晴朗！"

agent = create_agent(
    model="openai:gpt-5-nano",
    tools=[get_weather],
)

for mode, chunk in agent.stream(
    {"messages": [{"role": "user", "content": "旧金山的天气如何？"}]},
    stream_mode=["updates", "custom"]
):
    print(f"流模式: {mode}")
    print(f"内容: {chunk}\n")
```

输出结果包含两类信息：智能体执行进度（updates）与自定义日志（custom）。

---

## 禁用流式传输

在某些场景下，你可能希望关闭模型的 token 流式输出。
例如在多智能体系统中，可以只让部分智能体启用流式输出以减少干扰。

参见 [Models 指南](https://python.langchain.com/oss/langchain/models#disable-streaming) 了解如何禁用流式传输。

---

💡 **提示**
你可以通过 [MCP 接口](/use-these-docs) 将这些文档接入 Claude、VSCode 等环境，实现实时查询。
