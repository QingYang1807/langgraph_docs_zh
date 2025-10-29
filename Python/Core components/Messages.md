# 消息（Messages）

在 LangChain 中，**消息（Messages）** 是模型上下文的基本单元。它们代表模型的输入和输出，携带对话状态所需的内容和元数据，使模型能够理解和响应用户。

每条消息对象包含以下三部分：

* <Icon icon="user" size={16} /> **角色（Role）**：标识消息类型（例如 `system`、`user`）
* <Icon icon="folder-closed" size={16} /> **内容（Content）**：消息的实际内容（文本、图像、音频、文档等）
* <Icon icon="tag" size={16} /> **元数据（Metadata）**：可选字段，如响应信息、消息 ID、token 使用量等

LangChain 提供了跨模型统一的标准消息类型，确保不同模型间的行为一致。

---

## 基本用法

最简单的方式是创建消息对象并将其传递给模型进行调用（invoke）。

```python
from langchain.chat_models import init_chat_model
from langchain.messages import HumanMessage, AIMessage, SystemMessage

model = init_chat_model("openai:gpt-5-nano")

system_msg = SystemMessage("你是一位乐于助人的助手。")
human_msg = HumanMessage("你好，最近怎么样？")

messages = [system_msg, human_msg]
response = model.invoke(messages)  # 返回 AIMessage
```

---

### 文本提示（Text Prompts）

文本提示仅为字符串，适合无需保留对话历史的简单任务。

```python
response = model.invoke("写一首关于春天的俳句")
```

**适用场景：**

* 仅一次性请求
* 不需要上下文对话历史
* 代码尽量简洁

---

### 消息提示（Message Prompts）

也可以通过传递消息对象列表来与模型交互：

```python
from langchain.messages import SystemMessage, HumanMessage, AIMessage

messages = [
    SystemMessage("你是一位诗歌专家"),
    HumanMessage("写一首关于春天的俳句"),
    AIMessage("樱花盛开……")
]
response = model.invoke(messages)
```

**适用场景：**

* 多轮对话
* 多模态内容（图像、音频、文件）
* 包含系统级指令

---

### 字典格式

也支持直接使用与 OpenAI 兼容的字典格式。

```python
messages = [
    {"role": "system", "content": "你是一位诗歌专家"},
    {"role": "user", "content": "写一首关于春天的俳句"},
    {"role": "assistant", "content": "樱花盛开……"}
]
response = model.invoke(messages)
```

---

## 消息类型（Message Types）

* <Icon icon="gear" size={16} /> **System Message**：设定模型行为与上下文
* <Icon icon="user" size={16} /> **Human Message**：用户输入
* <Icon icon="robot" size={16} /> **AI Message**：模型输出，包括文本、工具调用、元数据
* <Icon icon="wrench" size={16} /> **Tool Message**：代表工具调用的返回结果

---

### System Message

`SystemMessage` 用于为模型设置初始指令，定义语气、角色与输出规范。

```python
from langchain.messages import SystemMessage, HumanMessage

system_msg = SystemMessage("""
你是一名资深的 Python 开发者，精通 Web 框架。
请始终提供代码示例并解释推理过程。
回答要简洁但全面。
""")

messages = [
    system_msg,
    HumanMessage("如何创建一个 REST API？")
]
response = model.invoke(messages)
```

---

### Human Message

`HumanMessage` 代表用户的输入，可包含文本、图片、音频或其他内容。

```python
response = model.invoke([
    HumanMessage("什么是机器学习？")
])
```

或直接传字符串：

```python
response = model.invoke("什么是机器学习？")
```

可以添加元数据：

```python
human_msg = HumanMessage(
    content="你好！",
    name="alice",  # 用户名
    id="msg_123",  # 唯一ID
)
```

---

### AI Message

`AIMessage` 是模型输出对象，可包含多模态数据、工具调用和响应元信息。

```python
response = model.invoke("解释一下人工智能")
print(type(response))  # <class 'langchain_core.messages.AIMessage'>
```

可以手动创建 AIMessage 加入对话历史：

```python
from langchain.messages import AIMessage, SystemMessage, HumanMessage

ai_msg = AIMessage("当然可以帮你！")

messages = [
    SystemMessage("你是一位乐于助人的助手"),
    HumanMessage("你能帮我吗？"),
    ai_msg,
    HumanMessage("太好了！2+2 等于几？")
]
response = model.invoke(messages)
```

#### 工具调用（Tool Calls）

模型在执行工具调用时，相关调用信息会记录在 `AIMessage` 中：

```python
from langchain.chat_models import init_chat_model

def get_weather(location: str) -> str:
    """获取指定地点的天气"""
    ...

model = init_chat_model("openai:gpt-5-nano")
model_with_tools = model.bind_tools([get_weather])
response = model_with_tools.invoke("巴黎天气怎么样？")

for tool_call in response.tool_calls:
    print(tool_call)
```

#### Token 使用统计

`usage_metadata` 字段包含输入输出 token 数量：

```python
response.usage_metadata
# {'input_tokens': 8, 'output_tokens': 304, 'total_tokens': 312, ...}
```

#### 流式传输（Streaming）

流式模式下会返回 `AIMessageChunk`：

```python
chunks = []
for chunk in model.stream("你好"):
    chunks.append(chunk)
    print(chunk.text)
```

---

### Tool Message

当模型执行了工具调用后，使用 `ToolMessage` 向模型返回结果：

```python
from langchain.messages import ToolMessage, HumanMessage, AIMessage

ai_message = AIMessage(tool_calls=[{
    "name": "get_weather",
    "args": {"location": "San Francisco"},
    "id": "call_123"
}])

tool_message = ToolMessage(
    content="晴朗，22°C",
    tool_call_id="call_123"
)

messages = [
    HumanMessage("旧金山的天气怎么样？"),
    ai_message,
    tool_message
]
response = model.invoke(messages)
```

---

## 消息内容（Message Content）

消息的 `content` 字段代表传入模型的数据载体，可以是：

1. 字符串
2. 原生内容列表（例如 OpenAI 的格式）
3. LangChain 标准内容块（Content Blocks）

```python
from langchain.messages import HumanMessage

# 简单文本
HumanMessage("你好")

# 原生格式（含图像）
HumanMessage(content=[
    {"type": "text", "text": "你好"},
    {"type": "image_url", "image_url": {"url": "https://example.com/image.jpg"}}
])
```

---

### 标准内容块（Standard Content Blocks）

LangChain 提供统一格式的内容块表示，包括文本、推理、图像、音频、视频、文件、工具调用等类型。
所有消息的 `content_blocks` 属性都可统一解析为结构化数据。

例如：

```python
[
  {"type": "text", "text": "你好"},
  {"type": "image", "url": "https://example.com/image.jpg"}
]
```

---

### 多模态输入（Multimodal）

LangChain 支持文本、图像、音频、视频、PDF 等多种输入类型。示例：

```python
message = {
    "role": "user",
    "content": [
        {"type": "text", "text": "描述这张图片的内容。"},
        {"type": "image", "url": "https://example.com/image.jpg"}
    ]
}
```

支持 base64、文件ID、URL 三种方式。
不同模型支持的格式可能不同，请参考模型提供商文档。

---

## 聊天模型中的使用（Use with Chat Models）

聊天模型接收一系列消息对象作为输入，并返回 `AIMessage` 作为输出。
多数场景是无状态交互，通过不断增长的消息列表来维持对话。

更多内容请参考：

* [对话历史与记忆管理](/oss/python/langchain/short-term-memory)
* [上下文窗口优化策略](/oss/python/langchain/short-term-memory#common-patterns)

---

📘 [在 GitHub 上编辑本文档](https://github.com/langchain-ai/docs/edit/main/src/oss/langchain/messages.mdx)
💡 [通过 MCP 将本文档连接至 Claude、VSCode 等工具，实时查询](/use-these-docs)
