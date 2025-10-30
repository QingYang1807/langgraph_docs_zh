# Guardrails（安全护栏）

> 为你的智能体实现安全检查与内容过滤

Guardrails（安全护栏）帮助你构建安全、合规的 AI 应用，通过在智能体执行的关键点进行验证和过滤。它们可以检测敏感信息、强制执行内容策略、验证输出并阻止不安全行为，防止问题发生。

常见用例包括：

* 防止个人身份信息（PII）泄露
* 检测并阻止提示注入攻击
* 阻止不适当或有害内容
* 强制执行业务规则与合规要求
* 验证输出质量与准确性

你可以使用 [middleware](/oss/python/langchain/middleware) 在战略点拦截执行来实现安全护栏 —— 在智能体开始前、完成后或在模型与工具调用前后。

<div style={{ display: "flex", justifyContent: "center" }}>
  <img src="https://mintcdn.com/langchain-5e9cc07a/RAP6mjwE5G00xYsA/oss/images/middleware_final.png?fit=max&auto=format&n=RAP6mjwE5G00xYsA&q=85&s=eb4404b137edec6f6f0c8ccb8323eaf1" alt="Middleware flow diagram" className="rounded-lg" data-og-width="500" width="500" data-og-height="560" height="560" data-path="oss/images/middleware_final.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/RAP6mjwE5G00xYsA/oss/images/middleware_final.png?w=280&fit=max&auto=format&n=RAP6mjwE5G00xYsA&q=85&s=483413aa87cf93323b0f47c0dd5528e8 280w, https://mintcdn.com/langchain-5e9cc07a/RAP6mjwE5G00xYsA/oss/images/middleware_final.png?w=560&fit=max&auto=format&n=RAP6mjwE5G00xYsA&q=85&s=41b7dd647447978ff776edafe5f42499 560w, https://mintcdn.com/langchain-5e9cc07a/RAP6mjwE5G00xYsA/oss/images/middleware_final.png?w=840&fit=max&auto=format&n=RAP6mjwE5G00xYsA&q=85&s=e9b14e264f68345de08ae76f032c52d4 840w, https://mintcdn.com/langchain-5e9cc07a/RAP6mjwE5G00xYsA/oss/images/middleware_final.png?w=1100&fit=max&auto=format&n=RAP6mjwE5G00xYsA&q=85&s=ec45e1932d1279b1beee4a4b016b473f 1100w, https://mintcdn.com/langchain-5e9cc07a/RAP6mjwE5G00xYsA/oss/images/middleware_final.png?w=1650&fit=max&auto=format&n=RAP6mjwE5G00xYsA&q=85&s=3bca5ebf8aa56632b8a9826f7f112e57 1650w, https://mintcdn.com/langchain-5e9cc07a/RAP6mjwE5G00xYsA/oss/images/middleware_final.png?w=2500&fit=max&auto=format&n=RAP6mjwE5G00xYsA&q=85&s=437f141d1266f08a95f030c2804691d9 2500w" />
</div>

安全护栏可以通过两种互补的方法实现：

<CardGroup cols={2}>
  <Card title="确定性（规则）护栏" icon="list-check">
    使用基于规则的逻辑，例如正则、关键字匹配或显式检查。速度快、可预测且成本低，但可能漏掉细微违规。
  </Card>

  <Card title="基于模型的护栏" icon="brain">
    使用 LLM 或分类器从语义上评估内容。能捕捉规则漏掉的微妙问题，但更慢且更昂贵。
  </Card>
</CardGroup>

LangChain 提供内建的护栏（例如：[PII 检测](#pii-detection)、[人工审批环节（Human-in-the-loop）](#human-in-the-loop)）以及灵活的 middleware 系统，用于用任一方式构建自定义护栏。

## 内建护栏

### PII（个人身份信息）检测

LangChain 提供用于在对话中检测和处理个人身份信息（PII）的内建 middleware。该 middleware 能检测常见的 PII 类型，比如邮箱、信用卡、IP 地址等。

PII 检测 middleware 适用于需要合规的领域（例如医疗与金融）、需要清理日志的客服智能体，以及任何处理敏感用户数据的场景。

PII middleware 支持多种策略来处理检测到的 PII：

| 策略       | 描述                    | 示例                    |
| -------- | --------------------- | --------------------- |
| `redact` | 替换为 `[REDACTED_TYPE]` | `[REDACTED_EMAIL]`    |
| `mask`   | 部分遮盖（例如保留最后4位）        | `****-****-****-1234` |
| `hash`   | 替换为确定性哈希              | `a8f5f167...`         |
| `block`  | 检测到时抛出异常              | 抛出错误                  |

```python theme={null}
from langchain.agents import create_agent
from langchain.agents.middleware import PIIMiddleware


agent = create_agent(
    model="openai:gpt-4o",
    tools=[customer_service_tool, email_tool],
    middleware=[
        # Redact emails in user input before sending to model
        PIIMiddleware(
            "email",
            strategy="redact",
            apply_to_input=True,
        ),
        # Mask credit cards in user input
        PIIMiddleware(
            "credit_card",
            strategy="mask",
            apply_to_input=True,
        ),
        # Block API keys - raise error if detected
        PIIMiddleware(
            "api_key",
            detector=r"sk-[a-zA-Z0-9]{32}",
            strategy="block",
            apply_to_input=True,
        ),
    ],
)

# When user provides PII, it will be handled according to the strategy
result = agent.invoke({
    "messages": [{"role": "user", "content": "My email is john.doe@example.com and card is 4532-1234-5678-9010"}]
})
```

<Accordion title="内建 PII 类型与配置">
  **内建 PII 类型：**

* `email` - 电子邮件地址
* `credit_card` - 信用卡号（Luhn 校验）
* `ip` - IP 地址
* `mac_address` - MAC 地址
* `url` - URL

**配置选项：**

| 参数                      | 描述                                                     | 默认值          |
| ----------------------- | ------------------------------------------------------ | ------------ |
| `pii_type`              | 要检测的 PII 类型（内建或自定义）                                    | 必需           |
| `strategy`              | 处理检测到的 PII 的方式（`"block"`、`"redact"`、`"mask"`、`"hash"`） | `"redact"`   |
| `detector`              | 自定义检测函数或正则模式                                           | `None`（使用内建） |
| `apply_to_input`        | 在模型调用前检查用户消息                                           | `True`       |
| `apply_to_output`       | 在模型调用后检查 AI 消息                                         | `False`      |
| `apply_to_tool_results` | 在工具执行后检查工具结果消息                                         | `False`      |

</Accordion>

完整的 PII 检测能力请参见 [middleware 文档](/oss/python/langchain/middleware#pii-detection)。

### 人工审批环节（Human-in-the-loop）

LangChain 提供内建的 middleware，用于在执行敏感操作前要求人工批准。这是对高风险决策最有效的护栏之一。

Human-in-the-loop middleware 适用于金融交易、删除或修改生产数据、向外部发送通信，以及任何具有重大业务影响的操作。

```python theme={null}
from langchain.agents import create_agent
from langchain.agents.middleware import HumanInTheLoopMiddleware
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.types import Command


agent = create_agent(
    model="openai:gpt-4o",
    tools=[search_tool, send_email_tool, delete_database_tool],
    middleware=[
        HumanInTheLoopMiddleware(
            interrupt_on={
                # Require approval for sensitive operations
                "send_email": True,
                "delete_database": True,
                # Auto-approve safe operations
                "search": False,
            }
        ),
    ],
    # Persist the state across interrupts
    checkpointer=InMemorySaver(),
)

# Human-in-the-loop requires a thread ID for persistence
config = {"configurable": {"thread_id": "some_id"}}

# Agent will pause and wait for approval before executing sensitive tools
result = agent.invoke(
    {"messages": [{"role": "user", "content": "Send an email to the team"}]},
    config=config
)

result = agent.invoke(
    Command(resume={"decisions": [{"type": "approve"}]}),
    config=config  # Same thread ID to resume the paused conversation
)
```

<Tip>
  完整的审批工作流实现细节请参见 [human-in-the-loop 文档](/oss/python/langchain/human-in-the-loop)。
</Tip>

## 自定义护栏

对于更复杂的护栏，你可以创建自定义 middleware，在智能体执行前或执行后运行。这让你对验证逻辑、内容过滤与安全检查拥有全部控制权。

### 智能体前（before agent）护栏

使用 “before agent” 钩子在每次调用开始时进行验证。适用于会话级检查，如认证、限流或在任何处理开始前阻止不当请求。

<CodeGroup>
  ```python title="Class syntax" theme={null}
  from typing import Any

from langchain.agents.middleware import AgentMiddleware, AgentState, hook_config
from langgraph.runtime import Runtime

class ContentFilterMiddleware(AgentMiddleware):
"""Deterministic guardrail: Block requests containing banned keywords."""

```
  def __init__(self, banned_keywords: list[str]):
      super().__init__()
      self.banned_keywords = [kw.lower() for kw in banned_keywords]

  @hook_config(can_jump_to=["end"])
  def before_agent(self, state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
      # Get the first user message
      if not state["messages"]:
          return None

      first_message = state["messages"][0]
      if first_message.type != "human":
          return None

      content = first_message.content.lower()

      # Check for banned keywords
      for keyword in self.banned_keywords:
          if keyword in content:
              # Block execution before any processing
              return {
                  "messages": [{
                      "role": "assistant",
                      "content": "I cannot process requests containing inappropriate content. Please rephrase your request."
                  }],
                  "jump_to": "end"
              }

      return None
```

# Use the custom guardrail

from langchain.agents import create_agent

agent = create_agent(
model="openai:gpt-4o",
tools=[search_tool, calculator_tool],
middleware=[
ContentFilterMiddleware(banned_keywords=["hack", "exploit", "malware"]),
],
)

# This request will be blocked before any processing

result = agent.invoke({
"messages": [{"role": "user", "content": "How do I hack into a database?"}]
})

````

```python title="Decorator syntax" theme={null}
from typing import Any

from langchain.agents.middleware import before_agent, AgentState, hook_config
from langgraph.runtime import Runtime

banned_keywords = ["hack", "exploit", "malware"]

@before_agent(can_jump_to=["end"])
def content_filter(state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
    """Deterministic guardrail: Block requests containing banned keywords."""
    # Get the first user message
    if not state["messages"]:
        return None

    first_message = state["messages"][0]
    if first_message.type != "human":
        return None

    content = first_message.content.lower()

    # Check for banned keywords
    for keyword in banned_keywords:
        if keyword in content:
            # Block execution before any processing
            return {
                "messages": [{
                    "role": "assistant",
                    "content": "I cannot process requests containing inappropriate content. Please rephrase your request."
                }],
                "jump_to": "end"
            }

    return None

# Use the custom guardrail
from langchain.agents import create_agent

agent = create_agent(
    model="openai:gpt-4o",
    tools=[search_tool, calculator_tool],
    middleware=[content_filter],
)

# This request will be blocked before any processing
result = agent.invoke({
    "messages": [{"role": "user", "content": "How do I hack into a database?"}]
})
````

</CodeGroup>

### 智能体后（after agent）护栏

使用 “after agent” 钩子在返回给用户之前验证最终输出。适用于基于模型的安全检查、质量验证或对完整智能体响应的合规扫描。

<CodeGroup>
  ```python title="Class syntax" theme={null}
  from langchain.agents.middleware import AgentMiddleware, AgentState, hook_config
  from langgraph.runtime import Runtime
  from langchain_core.messages import AIMessage
  from langchain.chat_models import init_chat_model
  from typing import Any

class SafetyGuardrailMiddleware(AgentMiddleware):
"""Model-based guardrail: Use an LLM to evaluate response safety."""

```
  def __init__(self):
      super().__init__()
      self.safety_model = init_chat_model("openai:gpt-4o-mini")

  @hook_config(can_jump_to=["end"])
  def after_agent(self, state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
      # Get the final AI response
      if not state["messages"]:
          return None

      last_message = state["messages"][-1]
      if not isinstance(last_message, AIMessage):
          return None

      # Use a model to evaluate safety
      safety_prompt = f"""Evaluate if this response is safe and appropriate.
      Respond with only 'SAFE' or 'UNSAFE'.

      Response: {last_message.content}"""

      result = self.safety_model.invoke([{"role": "user", "content": safety_prompt}])

      if "UNSAFE" in result.content:
          return {
              "messages": [{
                  "role": "assistant",
                  "content": "I cannot provide that response. Please rephrase your request."
              }],
              "jump_to": "end"
          }

      return None
```

# Use the safety guardrail

from langchain.agents import create_agent

agent = create_agent(
model="openai:gpt-4o",
tools=[search_tool, calculator_tool],
middleware=[SafetyGuardrailMiddleware()],
)

result = agent.invoke({
"messages": [{"role": "user", "content": "How do I make explosives?"}]
})

````

```python title="Decorator syntax" theme={null}
from langchain.agents.middleware import after_agent, AgentState, hook_config
from langgraph.runtime import Runtime
from langchain_core.messages import AIMessage
from langchain.chat_models import init_chat_model
from typing import Any

safety_model = init_chat_model("openai:gpt-4o-mini")

@after_agent(can_jump_to=["end"])
def safety_guardrail(state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
    """Model-based guardrail: Use an LLM to evaluate response safety."""
    # Get the final AI response
    if not state["messages"]:
        return None

    last_message = state["messages"][-1]
    if not isinstance(last_message, AIMessage):
        return None

    # Use a model to evaluate safety
    safety_prompt = f"""Evaluate if this response is safe and appropriate.
    Respond with only 'SAFE' or 'UNSAFE'.

    Response: {last_message.content}"""

    result = safety_model.invoke([{"role": "user", "content": safety_prompt}])

    if "UNSAFE" in result.content:
        return {
            "messages": [{
                "role": "assistant",
                "content": "I cannot provide that response. Please rephrase your request."
            }],
            "jump_to": "end"
        }

    return None

# Use the safety guardrail
from langchain.agents import create_agent

agent = create_agent(
    model="openai:gpt-4o",
    tools=[search_tool, calculator_tool],
    middleware=[safety_guardrail],
)

result = agent.invoke({
    "messages": [{"role": "user", "content": "How do I make explosives?"}]
})
````

</CodeGroup>

### 组合多重护栏

你可以将多个护栏按顺序加入 middleware 数组，从而构建分层保护：它们按顺序执行，允许你形成多层防护。

```python theme={null}
from langchain.agents import create_agent
from langchain.agents.middleware import PIIMiddleware, HumanInTheLoopMiddleware

agent = create_agent(
    model="openai:gpt-4o",
    tools=[search_tool, send_email_tool],
    middleware=[
        # Layer 1: Deterministic input filter (before agent)
        ContentFilterMiddleware(banned_keywords=["hack", "exploit"]),

        # Layer 2: PII protection (before and after model)
        PIIMiddleware("email", strategy="redact", apply_to_input=True),
        PIIMiddleware("email", strategy="redact", apply_to_output=True),

        # Layer 3: Human approval for sensitive tools
        HumanInTheLoopMiddleware(interrupt_on={"send_email": True}),

        # Layer 4: Model-based safety check (after agent)
        SafetyGuardrailMiddleware(),
    ],
)
```

## 附加资源

* [Middleware 文档](/oss/python/langchain/middleware) - 自定义 middleware 的完整指南
* [Middleware API 参考](https://reference.langchain.com/python/langchain/middleware/) - Middleware 的完整接口说明
* [Human-in-the-loop](/oss/python/langchain/human-in-the-loop) - 为敏感操作添加人工复核
* [测试智能体（Testing agents）](/oss/python/langchain/test) - 测试安全机制的策略

---

<Callout icon="pen-to-square" iconType="regular">
  [在 GitHub 上编辑此页面的源文件。](https://github.com/langchain-ai/docs/edit/main/src/oss/langchain/guardrails.mdx)
</Callout>

<Tip icon="terminal" iconType="regular">
  [以编程方式将这些文档连接到 Claude、VSCode 等，通过 MCP 获取实时答案。](/use-these-docs)
</Tip>
