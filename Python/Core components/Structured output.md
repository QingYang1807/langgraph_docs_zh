# 结构化输出

结构化输出允许智能体以特定、可预测的格式返回数据。与其解析自然语言响应，不如直接获得结构化的数据形式，比如 JSON 对象、Pydantic 模型或数据类（dataclasses），你的应用可以直接使用这些数据。

LangChain 的 [`create_agent`](https://reference.langchain.com/python/langchain/agents/#langchain.agents.create_agent) 会自动处理结构化输出。用户设置好所需的结构化输出模式（schema）后，当模型生成结构化数据时，系统会自动捕获、验证，并将其返回到智能体状态中的 `'structured_response'` 键下。

```python
def create_agent(
    ...
    response_format: Union[
        ToolStrategy[StructuredResponseT],
        ProviderStrategy[StructuredResponseT],
        type[StructuredResponseT],
    ]
```

## 响应格式（Response Format）

用于控制智能体如何返回结构化数据：

* **`ToolStrategy[StructuredResponseT]`**：使用工具调用实现结构化输出
* **`ProviderStrategy[StructuredResponseT]`**：使用模型提供商原生的结构化输出功能
* **`type[StructuredResponseT]`**：仅指定 schema 类型，LangChain 会根据模型能力自动选择最优策略
* **`None`**：无结构化输出

当直接提供 schema 类型时，LangChain 会自动选择：

* 若模型支持原生结构化输出（如 [OpenAI](/oss/python/integrations/providers/openai)、[Grok](/oss/python/integrations/providers/xai)），则使用 `ProviderStrategy`
* 其他情况则使用 `ToolStrategy`

最终的结构化响应会返回在智能体最终状态的 `structured_response` 键中。

---

## Provider Strategy（提供商策略）

某些模型提供商（目前只有 OpenAI 和 Grok）通过其 API 原生支持结构化输出。这是最可靠的方法（若可用时推荐使用）。

要使用此策略，请配置 `ProviderStrategy`：

```python
class ProviderStrategy(Generic[SchemaT]):
    schema: type[SchemaT]
```

**参数说明：**

<ParamField path="schema" required>
  定义结构化输出格式的模式。支持以下类型：
  * **Pydantic 模型**：继承自 `BaseModel` 的类，带字段验证  
  * **Dataclass**：带类型注解的 Python 数据类  
  * **TypedDict**：带类型定义的字典类  
  * **JSON Schema**：符合 JSON Schema 规范的字典  
</ParamField>

当你将 schema 类型直接传入 [`create_agent.response_format`](https://reference.langchain.com/python/langchain/agents/#langchain.agents.create_agent%28response_format%29) 且模型支持原生结构化输出时，LangChain 会自动使用 `ProviderStrategy`。

---

### 示例

#### Pydantic 模型

```python
from pydantic import BaseModel, Field
from langchain.agents import create_agent

class ContactInfo(BaseModel):
    """联系信息"""
    name: str = Field(description="姓名")
    email: str = Field(description="邮箱地址")
    phone: str = Field(description="电话号码")

agent = create_agent(
    model="openai:gpt-5",
    tools=tools,
    response_format=ContactInfo  # 自动选择 ProviderStrategy
)

result = agent.invoke({
    "messages": [{"role": "user", "content": "Extract contact info from: John Doe, john@example.com, (555) 123-4567"}]
})

result["structured_response"]
# ContactInfo(name='John Doe', email='john@example.com', phone='(555) 123-4567')
```

#### Dataclass

```python
from dataclasses import dataclass
from langchain.agents import create_agent

@dataclass
class ContactInfo:
    name: str
    email: str
    phone: str

agent = create_agent(
    model="openai:gpt-5",
    tools=tools,
    response_format=ContactInfo
)
```

#### TypedDict

```python
from typing_extensions import TypedDict
from langchain.agents import create_agent

class ContactInfo(TypedDict):
    name: str
    email: str
    phone: str

agent = create_agent(
    model="openai:gpt-5",
    tools=tools,
    response_format=ContactInfo
)
```

#### JSON Schema

```python
contact_info_schema = {
    "type": "object",
    "description": "联系信息",
    "properties": {
        "name": {"type": "string", "description": "姓名"},
        "email": {"type": "string", "description": "邮箱"},
        "phone": {"type": "string", "description": "电话"}
    },
    "required": ["name", "email", "phone"]
}

agent = create_agent(
    model="openai:gpt-5",
    tools=tools,
    response_format=contact_info_schema
)
```

> **说明：**
> 如果模型提供商原生支持结构化输出，那么直接写 `response_format=ProductReview` 与 `response_format=ToolStrategy(ProductReview)` 在功能上等效。若不支持结构化输出，LangChain 会自动回退到工具调用模式。

---

## Tool Calling Strategy（工具调用策略）

对于不支持原生结构化输出的模型，LangChain 会使用“工具调用”实现同样的功能。几乎所有现代模型都支持这种方式。

```python
class ToolStrategy(Generic[SchemaT]):
    schema: type[SchemaT]
    tool_message_content: str | None
    handle_errors: Union[
        bool,
        str,
        type[Exception],
        tuple[type[Exception], ...],
        Callable[[Exception], str],
    ]
```

**参数说明：**

* **`schema`**：定义结构化输出格式。支持 Pydantic、Dataclass、TypedDict、JSON Schema、Union 类型
* **`tool_message_content`**：当生成结构化输出时返回的自定义消息内容
* **`handle_errors`**：结构化输出验证失败时的错误处理策略

---

### 示例：Pydantic 模型

```python
from pydantic import BaseModel, Field
from typing import Literal
from langchain.agents import create_agent
from langchain.agents.structured_output import ToolStrategy

class ProductReview(BaseModel):
    """产品评论分析"""
    rating: int | None = Field(description="产品评分", ge=1, le=5)
    sentiment: Literal["positive", "negative"] = Field(description="情感倾向")
    key_points: list[str] = Field(description="评论要点")

agent = create_agent(
    model="openai:gpt-5",
    tools=tools,
    response_format=ToolStrategy(ProductReview)
)
```

其他如 Dataclass、TypedDict、JSON Schema、Union 类型的用法完全类似。

---

## 自定义工具消息内容

`tool_message_content` 参数允许你自定义结构化输出生成时显示在对话记录中的消息内容。

---

## 错误处理（Error Handling）

LangChain 提供自动重试机制来处理结构化输出错误，包括：

* 多个结构化输出错误（Multiple Structured Outputs Error）
* 模式验证错误（Schema Validation Error）

支持以下错误处理策略：

| 模式                | 说明            |
| ----------------- | ------------- |
| `True`            | 捕获所有错误并使用默认提示 |
| `str`             | 使用自定义错误消息     |
| `type[Exception]` | 仅捕获特定异常       |
| `tuple`           | 捕获多个异常类型      |
| `Callable`        | 自定义错误处理函数     |
| `False`           | 不捕获错误，直接抛出异常  |

---

### 示例：自定义错误消息

```python
ToolStrategy(
    schema=ProductRating,
    handle_errors="请提供1-5之间的有效评分并包含评论。"
)
```

### 示例：自定义错误处理函数

```python
def custom_error_handler(error: Exception) -> str:
    if isinstance(error, StructuredOutputValidationError):
        return "输出格式错误，请重试。"
    elif isinstance(error, MultipleStructuredOutputsError):
        return "检测到多个结构化输出，请选择最相关的一个。"
    else:
        return f"错误: {str(error)}"
```

---

**无错误处理模式：**

```python
response_format = ToolStrategy(
    schema=ProductRating,
    handle_errors=False
)
```

---

<Callout icon="pen-to-square" iconType="regular">
  [在 GitHub 上编辑此页面源代码](https://github.com/langchain-ai/docs/edit/main/src/oss/langchain/structured-output.mdx)
</Callout>

<Tip icon="terminal" iconType="regular">
  [通过 MCP 接入 Claude、VSCode 等，实现文档实时问答](/use-these-docs)
</Tip>
