# 智能体中的上下文工程

## 概述

构建智能体（或任何LLM应用）的难点在于使它们足够可靠。虽然它们在原型阶段可能工作正常，但在实际使用场景中常常失败。

### 为什么智能体会失败？

当智能体失败时，通常是因为智能体内部的LLM调用采取了错误的操作/没有按预期执行。LLM失败的原因有两个：

1. 底层LLM能力不足
2. 未向LLM传递"正确的"上下文

通常情况下，实际上是第二个原因导致智能体不可靠。

**上下文工程**是以正确的格式提供正确的信息和工具，使LLM能够完成任务。这是AI工程师的首要工作。这种缺乏"正确"上下文的情况是构建更可靠智能体的主要障碍，而LangChain的智能体抽象在设计上就特别便于上下文工程的实施。

<Tip>
  上下文工程新手？请先从[概念概述](/oss/python/concepts/context)开始，了解不同类型的上下文及其使用时机。
</Tip>

### 智能体循环

典型的智能体循环包括两个主要步骤：

1. **模型调用** - 使用提示词和可用工具调用LLM，返回响应或执行工具的请求
2. **工具执行** - 执行LLM请求的工具，返回工具结果

<div style={{ display: "flex", justifyContent: "center" }}>
  <img src="https://mintcdn.com/langchain-5e9cc07a/Tazq8zGc0yYUYrDl/oss/images/core_agent_loop.png?fit=max&auto=format&n=Tazq8zGc0yYUYrDl&q=85&s=ac72e48317a9ced68fd1be64e89ec063" alt="核心智能体循环图" className="rounded-lg" data-og-width="300" width="300" data-og-height="268" height="268" data-path="oss/images/core_agent_loop.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/Tazq8zGc0yYUYrDl/oss/images/core_agent_loop.png?w=280&fit=max&auto=format&n=Tazq8zGc0yYUYrDl&q=85&s=a4c4b766b6678ef52a6ed556b1a0b032 280w, https://mintcdn.com/langchain-5e9cc07a/Tazq8zGc0yYUYrDl/oss/images/core_agent_loop.png?w=560&fit=max&auto=format&n=Tazq8zGc0yYUYrDl&q=85&s=111869e6e99a52c0eff60a1ef7ddc49c 560w, https://mintcdn.com/langchain-5e9cc07a/Tazq8zGc0yYUYrDl/oss/images/core_agent_loop.png?w=840&fit=max&auto=format&n=Tazq8zGc0yYUYrDl&q=85&s=6c1e21de7b53bd0a29683aca09c6f86e 840w, https://mintcdn.com/langchain-5e9cc07a/Tazq8zGc0yYUYrDl/oss/images/core_agent_loop.png?w=1100&fit=max&auto=format&n=Tazq8zGc0yYUYrDl&q=85&s=88bef556edba9869b759551c610c60f4 1100w, https://mintcdn.com/langchain-5e9cc07a/Tazq8zGc0yYUYrDl/oss/images/core_agent_loop.png?w=1650&fit=max&auto=format&n=Tazq8zGc0yYUYrDl&q=85&s=9b0bdd138e9548eeb5056dc0ed2d4a4b 1650w, https://mintcdn.com/langchain-5e9cc07a/Tazq8zGc0yYUYrDl/oss/images/core_agent_loop.png?w=2500&fit=max&auto=format&n=Tazq8zGc0yYUYrDl&q=85&s=41eb4f053ed5e6b0ba5bad2badf6d755 2500w" />
</div>

这个循环会持续进行，直到LLM决定结束。

### 你可以控制的内容

要构建可靠的智能体，你需要控制智能体循环每一步发生的事，以及步骤之间发生的事。

| 上下文类型                                  | 你控制的内容                                                                 | 临时性或持久性 |
| --------------------------------------------- | ---------------------------------------------------------------------------- | ----------------------- |
| **[模型上下文](#模型上下文)**           | 进入模型调用的内容（指令、消息历史、工具、响应格式）   | 临时性               |
| **[工具上下文](#工具上下文)**             | 工具可以访问和产生的内容（对状态、存储、运行时上下文的读取/写入）    | 持久性              |
| **[生命周期上下文](#生命周期上下文)** | 模型和工具调用之间发生的事情（摘要、护栏、日志等） | 持久性              |

<CardGroup>
  <Card title="临时性上下文" icon="bolt" iconType="duotone">
    LLM在单次调用中看到的内容。你可以修改消息、工具或提示词，而不改变保存在状态中的内容。
  </Card>

  <Card title="持久性上下文" icon="database" iconType="duotone">
    在多个轮次中保存在状态中的内容。生命周期钩子和工具写入会永久修改这些内容。
  </Card>
</CardGroup>

### 数据源

在整个过程中，你的智能体访问（读取/写入）不同的数据源：

| 数据源         | 也称为        | 范围               | 示例                                                                   |
| ------------------- | -------------------- | ------------------- | -------------------------------------------------------------------------- |
| **运行时上下文** | 静态配置 | 对话范围 | 用户ID、API密钥、数据库连接、权限、环境设置 |
| **状态**           | 短期记忆    | 对话范围 | 当前消息、上传的文件、认证状态、工具结果      |
| **存储**           | 长期记忆     | 跨对话  | 用户偏好、提取的见解、记忆、历史数据            |

### 工作原理

LangChain的[中间件](/oss/python/langchain/middleware)是底层机制，它使使用LangChain的开发者能够实际进行上下文工程。

中间件允许你钩入智能体生命周期的任何步骤并：

* 更新上下文
* 跳转到智能体生命周期的不同步骤

在本指南中，你将看到中间件API被频繁用作实现上下文工程目标的手段。

## 模型上下文

控制每次模型调用的内容——指令、可用工具、使用哪个模型以及输出格式。这些决策直接影响可靠性和成本。

<CardGroup cols={2}>
  <Card title="系统提示词" icon="message-lines" href="#系统提示词">
    开发者给LLM的基础指令。
  </Card>

  <Card title="消息" icon="comments" href="#消息">
    发送给LLM的完整消息列表（对话历史）。
  </Card>

  <Card title="工具" icon="wrench" href="#工具">
    智能体可用于采取行动的实用工具。
  </Card>

  <Card title="模型" icon="brain-circuit" href="#模型">
    要调用的实际模型（包括配置）。
  </Card>

  <Card title="响应格式" icon="brackets-curly" href="#响应格式">
    模型最终响应的模式规范。
  </Card>
</CardGroup>

所有这些类型的模型上下文都可以从**状态**（短期记忆）、**存储**（长期记忆）或**运行时上下文**（静态配置）中获取。

### 系统提示词

系统提示词设置LLM的行为和能力。不同的用户、上下文或对话阶段需要不同的指令。成功的智能体会利用记忆、偏好和配置来为当前对话状态提供正确的指令。

<Tabs>
  <Tab title="状态">
    从状态中访问消息数量或对话上下文：

    ```python  theme={null}
    from langchain.agents import create_agent
    from langchain.agents.middleware import dynamic_prompt, ModelRequest

    @dynamic_prompt
    def state_aware_prompt(request: ModelRequest) -> str:
        # request.messages 是 request.state["messages"] 的快捷方式
        message_count = len(request.messages)

        base = "你是一个有帮助的助手。"

        if message_count > 10:
            base += "\n这是一个长对话 - 请特别简洁。"

        return base

    agent = create_agent(
        model="openai:gpt-4o",
        tools=[...],
        middleware=[state_aware_prompt]
    )
    ```
  </Tab>

  <Tab title="存储">
    从长期记忆中访问用户偏好：

    ```python  theme={null}
    from dataclasses import dataclass
    from langchain.agents import create_agent
    from langchain.agents.middleware import dynamic_prompt, ModelRequest
    from langgraph.store.memory import InMemoryStore

    @dataclass
    class Context:
        user_id: str

    @dynamic_prompt
    def store_aware_prompt(request: ModelRequest) -> str:
        user_id = request.runtime.context.user_id

        # 从存储读取：获取用户偏好
        store = request.runtime.store
        user_prefs = store.get(("preferences",), user_id)

        base = "你是一个有帮助的助手。"

        if user_prefs:
            style = user_prefs.value.get("communication_style", "balanced")
            base += f"\n用户偏好{style}风格的回答。"

        return base

    agent = create_agent(
        model="openai:gpt-4o",
        tools=[...],
        middleware=[store_aware_prompt],
        context_schema=Context,
        store=InMemoryStore()
    )
    ```
  </Tab>

  <Tab title="运行时上下文">
    从运行时上下文中访问用户ID或配置：

    ```python  theme={null}
    from dataclasses import dataclass
    from langchain.agents import create_agent
    from langchain.agents.middleware import dynamic_prompt, ModelRequest

    @dataclass
    class Context:
        user_role: str
        deployment_env: str

    @dynamic_prompt
    def context_aware_prompt(request: ModelRequest) -> str:
        # 从运行时上下文读取：用户角色和环境
        user_role = request.runtime.context.user_role
        env = request.runtime.context.deployment_env

        base = "你是一个有帮助的助手。"

        if user_role == "admin":
            base += "\n你有管理员权限。你可以执行所有操作。"
        elif user_role == "viewer":
            base += "\n你只有只读权限。请引导用户只进行读取操作。"

        if env == "production":
            base += "\n对任何数据修改要特别小心。"

        return base

    agent = create_agent(
        model="openai:gpt-4o",
        tools=[...],
        middleware=[context_aware_prompt],
        context_schema=Context
    )
    ```
  </Tab>
</Tabs>

### 消息

消息构成了发送给LLM的提示词。
管理消息内容至关重要，以确保LLM有正确的信息来做出良好响应。

<Tabs>
  <Tab title="状态">
    当与当前查询相关时，从状态中注入上传的文件上下文：

    ```python  theme={null}
    from langchain.agents import create_agent
    from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse
    from typing import Callable

    @wrap_model_call
    def inject_file_context(
        request: ModelRequest,
        handler: Callable[[ModelRequest], ModelResponse]
    ) -> ModelResponse:
        """注入关于用户在此会话中上传的文件的上下文。"""
        # 从状态读取：获取上传的文件元数据
        uploaded_files = request.state.get("uploaded_files", [])  # [!code highlight]

        if uploaded_files:
            # 构建关于可用文件的上下文
            file_descriptions = []
            for file in uploaded_files:
                file_descriptions.append(
                    f"- {file['name']} ({file['type']}): {file['summary']}"
                )

            file_context = f"""你在本次对话中可以访问的文件：
    {chr(10).join(file_descriptions)}

    回答问题时请参考这些文件。"""

            # 在最近消息之前注入文件上下文
            messages = [  # [!code highlight]
                *request.messages
                {"role": "user", "content": file_context},
            ]
            request = request.override(messages=messages)  # [!code highlight]

        return handler(request)

    agent = create_agent(
        model="openai:gpt-4o",
        tools=[...],
        middleware=[inject_file_context]
    )
    ```
  </Tab>

  <Tab title="存储">
    从存储中注入用户的电子邮件写作风格以指导起草：

    ```python  theme={null}
    from dataclasses import dataclass
    from langchain.agents import create_agent
    from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse
    from typing import Callable
    from langgraph.store.memory import InMemoryStore

    @dataclass
    class Context:
        user_id: str

    @wrap_model_call
    def inject_writing_style(
        request: ModelRequest,
        handler: Callable[[ModelRequest], ModelResponse]
    ) -> ModelResponse:
        """从存储中注入用户的电子邮件写作风格。"""
        user_id = request.runtime.context.user_id  # [!code highlight]

        # 从存储读取：获取用户的写作风格示例
        store = request.runtime.store  # [!code highlight]
        writing_style = store.get(("writing_style",), user_id)  # [!code highlight]

        if writing_style:
            style = writing_style.value
            # 从存储的示例构建风格指南
            style_context = f"""你的写作风格：
    - 语气：{style.get('tone', 'professional')}
    - 典型问候语："{style.get('greeting', 'Hi')}"
    - 典型结束语："{style.get('sign_off', 'Best')}"
    - 你写的示例邮件：
    {style.get('example_email', '')}"""

            # 附加在末尾 - 模型更关注最后的消息
            messages = [
                *request.messages,
                {"role": "user", "content": style_context}
            ]
            request = request.override(messages=messages)  # [!code highlight]

        return handler(request)

    agent = create_agent(
        model="openai:gpt-4o",
        tools=[...],
        middleware=[inject_writing_style],
        context_schema=Context,
        store=InMemoryStore()
    )
    ```
  </Tab>

  <Tab title="运行时上下文">
    根据用户的司法管辖区从运行时上下文中注入合规规则：

    ```python  theme={null}
    from dataclasses import dataclass
    from langchain.agents import create_agent
    from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse
    from typing import Callable

    @dataclass
    class Context:
        user_jurisdiction: str
        industry: str
        compliance_frameworks: list[str]

    @wrap_model_call
    def inject_compliance_rules(
        request: ModelRequest,
        handler: Callable[[ModelRequest], ModelResponse]
    ) -> ModelResponse:
        """从运行时上下文注入合规约束。"""
        # 从运行时上下文读取：获取合规要求
        jurisdiction = request.runtime.context.user_jurisdiction  # [!code highlight]
        industry = request.runtime.context.industry  # [!code highlight]
        frameworks = request.runtime.context.compliance_frameworks  # [!code highlight]

        # 构建合规约束
        rules = []
        if "GDPR" in frameworks:
            rules.append("- 处理个人数据前必须获得明确同意")
            rules.append("- 用户有权删除数据")
        if "HIPAA" in frameworks:
            rules.append("- 未经授权不得共享患者健康信息")
            rules.append("- 必须使用安全、加密的通信")
        if industry == "finance":
            rules.append("- 没有适当的免责声明不得提供财务建议")

        if rules:
            compliance_context = f"""{jurisdiction}的合规要求：
    {chr(10).join(rules)}"""

            # 附加在末尾 - 模型更关注最后的消息
            messages = [
                *request.messages,
                {"role": "user", "content": compliance_context}
            ]
            request = request.override(messages=messages)  # [!code highlight]

        return handler(request)

    agent = create_agent(
        model="openai:gpt-4o",
        tools=[...],
        middleware=[inject_compliance_rules],
        context_schema=Context
    )
    ```
  </Tab>
</Tabs>

<Note>
  **临时性与持久性消息更新：**

  上面的示例使用`wrap_model_call`进行**临时性**更新 - 修改发送给模型的消息仅针对单次调用，而不改变保存在状态中的内容。

  对于修改状态的**持久性**更新（如[生命周期上下文](#摘要)中的摘要示例），使用`before_model`或`after_model`等生命周期钩子来永久更新对话历史。有关更多详细信息，请参见[中间件文档](/oss/python/langchain/middleware)。
</Note>

### 工具

工具让模型能够与数据库、API和外部系统交互。你定义和选择工具的方式直接影响模型是否能有效完成任务。

#### 定义工具

每个工具都需要清晰的名称、描述、参数名称和参数描述。这些不仅仅是元数据——它们引导模型关于何时以及如何使用工具的推理。

```python  theme={null}
from langchain.tools import tool

@tool(parse_docstring=True)
def search_orders(
    user_id: str,
    status: str,
    limit: int = 10
) -> str:
    """按状态搜索用户订单。

    当用户询问订单历史或想要检查订单状态时使用此工具。
    始终按提供的状态进行筛选。

    Args:
        user_id: 用户的唯一标识符
        status: 订单状态：'pending'（待处理）、'shipped'（已发货）或'delivered'（已送达）
        limit: 返回的最大结果数
    """
    # 实现在此
    pass
```

#### 选择工具

并非每个工具都适合每种情况。太多工具可能会使模型不知所措（上下文过载）并增加错误；太少则限制能力。动态工具选择根据认证状态、用户权限、功能标志或对话阶段调整可用工具集。

<Tabs>
  <Tab title="状态">
    仅在达到某些对话里程碑后启用高级工具：

    ```python  theme={null}
    from langchain.agents import create_agent
    from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse
    from typing import Callable

    @wrap_model_call
    def state_based_tools(
        request: ModelRequest,
        handler: Callable[[ModelRequest], ModelResponse]
    ) -> ModelResponse:
        """根据对话状态筛选工具。"""
        # 从状态读取：检查用户是否已认证
        state = request.state  # [!code highlight]
        is_authenticated = state.get("authenticated", False)  # [!code highlight]
        message_count = len(state["messages"])

        # 仅在认证后启用敏感工具
        if not is_authenticated:
            tools = [t for t in request.tools if t.name.startswith("public_")]
            request = request.override(tools=tools)  # [!code highlight]
        elif message_count < 5:
            # 在对话早期限制工具
            tools = [t for t in request.tools if t.name != "advanced_search"]
            request = request.override(tools=tools)  # [!code highlight]

        return handler(request)

    agent = create_agent(
        model="openai:gpt-4o",
        tools=[public_search, private_search, advanced_search],
        middleware=[state_based_tools]
    )
    ```
  </Tab>

  <Tab title="存储">
    根据存储中的用户偏好或功能标志筛选工具：

    ```python  theme={null}
    from dataclasses import dataclass
    from langchain.agents import create_agent
    from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse
    from typing import Callable
    from langgraph.store.memory import InMemoryStore

    @dataclass
    class Context:
        user_id: str

    @wrap_model_call
    def store_based_tools(
        request: ModelRequest,
        handler: Callable[[ModelRequest], ModelResponse]
    ) -> ModelResponse:
        """根据存储偏好筛选工具。"""
        user_id = request.runtime.context.user_id

        # 从存储读取：获取用户已启用的功能
        store = request.runtime.store
        feature_flags = store.get(("features",), user_id)

        if feature_flags:
            enabled_features = feature_flags.value.get("enabled_tools", [])
            # 只包含为该用户启用的工具
            tools = [t for t in request.tools if t.name in enabled_features]
            request = request.override(tools=tools)

        return handler(request)

    agent = create_agent(
        model="openai:gpt-4o",
        tools=[search_tool, analysis_tool, export_tool],
        middleware=[store_based_tools],
        context_schema=Context,
        store=InMemoryStore()
    )
    ```
  </Tab>

  <Tab title="运行时上下文">
    根据运行时上下文中的用户权限筛选工具：

    ```python  theme={null}
    from dataclasses import dataclass
    from langchain.agents import create_agent
    from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse
    from typing import Callable

    @dataclass
    class Context:
        user_role: str

    @wrap_model_call
    def context_based_tools(
        request: ModelRequest,
        handler: Callable[[ModelRequest], ModelResponse]
    ) -> ModelResponse:
        """根据运行时上下文权限筛选工具。"""
        # 从运行时上下文读取：获取用户角色
        user_role = request.runtime.context.user_role

        if user_role == "admin":
            # 管理员获得所有工具
            pass
        elif user_role == "editor":
            # 编辑员不能删除
            tools = [t for t in request.tools if t.name != "delete_data"]
            request = request.override(tools=tools)
        else:
            # 查看者获得只读工具
            tools = [t for t in request.tools if t.name.startswith("read_")]
            request = request.override(tools=tools)

        return handler(request)

    agent = create_agent(
        model="openai:gpt-4o",
        tools=[read_data, write_data, delete_data],
        middleware=[context_based_tools],
        context_schema=Context
    )
    ```
  </Tab>
</Tabs>

有关更多示例，请参见[动态选择工具](/oss/python/langchain/middleware#dynamically-selecting-tools)。

### 模型

不同的模型具有不同的优势、成本和上下文窗口。为手头的任务选择合适的模型，在智能体运行期间可能会发生变化。

<Tabs>
  <Tab title="状态">
    根据状态中的对话长度使用不同的模型：

    ```python  theme={null}
    from langchain.agents import create_agent
    from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse
    from langchain.chat_models import init_chat_model
    from typing import Callable

    # 在中间件外部初始化模型一次
    large_model = init_chat_model("anthropic:claude-sonnet-4-5")
    standard_model = init_chat_model("openai:gpt-4o")
    efficient_model = init_chat_model("openai:gpt-4o-mini")

    @wrap_model_call
    def state_based_model(
        request: ModelRequest,
        handler: Callable[[ModelRequest], ModelResponse]
    ) -> ModelResponse:
        """根据状态对话长度选择模型。"""
        # request.messages 是 request.state["messages"] 的快捷方式
        message_count = len(request.messages)  # [!code highlight]

        if message_count > 20:
            # 长对话 - 使用较大上下文窗口的模型
            model = large_model
        elif message_count > 10:
            # 中等对话
            model = standard_model
        else:
            # 短对话 - 使用高效模型
            model = efficient_model

        request = request.override(model=model)  # [!code highlight]

        return handler(request)

    agent = create_agent(
        model="openai:gpt-4o-mini",
        tools=[...],
        middleware=[state_based_model]
    )
    ```
  </Tab>

  <Tab title="存储">
    使用存储中用户偏好的模型：

    ```python  theme={null}
    from dataclasses import dataclass
    from langchain.agents import create_agent
    from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse
    from langchain.chat_models import init_chat_model
    from typing import Callable
    from langgraph.store.memory import InMemoryStore

    @dataclass
    class Context:
        user_id: str

    # 初始化可用模型一次
    MODEL_MAP = {
        "gpt-4o": init_chat_model("openai:gpt-4o"),
        "gpt-4o-mini": init_chat_model("openai:gpt-4o-mini"),
        "claude-sonnet": init_chat_model("anthropic:claude-sonnet-4-5"),
    }

    @wrap_model_call
    def store_based_model(
        request: ModelRequest,
        handler: Callable[[ModelRequest], ModelResponse]
    ) -> ModelResponse:
        """根据存储偏好选择模型。"""
        user_id = request.runtime.context.user_id

        # 从存储读取：获取用户偏好的模型
        store = request.runtime.store
        user_prefs = store.get(("preferences",), user_id)

        if user_prefs:
            preferred_model = user_prefs.value.get("preferred_model")
            if preferred_model and preferred_model in MODEL_MAP:
                request = request.override(model=MODEL_MAP[preferred_model])

        return handler(request)

    agent = create_agent(
        model="openai:gpt-4o",
        tools=[...],
        middleware=[store_based_model],
        context_schema=Context,
        store=InMemoryStore()
    )
    ```
  </Tab>

  <Tab title="运行时上下文">
    根据运行时上下文中的成本限制或环境选择模型：

    ```python  theme={null}
    from dataclasses import dataclass
    from langchain.agents import create_agent
    from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse
    from langchain.chat_models import init_chat_model
    from typing import Callable

    @dataclass
    class Context:
        cost_tier: str
        environment: str

    # 在中间件外部初始化模型一次
    premium_model = init_chat_model("anthropic:claude-sonnet-4-5")
    standard_model = init_chat_model("openai:gpt-4o")
    budget_model = init_chat_model("openai:gpt-4o-mini")

    @wrap_model_call
    def context_based_model(
        request: ModelRequest,
        handler: Callable[[ModelRequest], ModelResponse]
    ) -> ModelResponse:
        """根据运行时上下文选择模型。"""
        # 从运行时上下文读取：成本等级和环境
        cost_tier = request.runtime.context.cost_tier
        environment = request.runtime.context.environment

        if environment == "production" and cost_tier == "premium":
            # 生产环境高级用户获得最佳模型
            model = premium_model
        elif cost_tier == "budget":
            # 预算等级获得高效模型
            model = budget_model
        else:
            # 标准等级
            model = standard_model

        request = request.override(model=model)

        return handler(request)

    agent = create_agent(
        model="openai:gpt-4o",
        tools=[...],
        middleware=[context_based_model],
        context_schema=Context
    )
    ```
  </Tab>
</Tabs>

有关更多示例，请参见[动态模型](/oss/python/langchain/agents#dynamic-model)。

### 响应格式

结构化输出将非结构化文本转换为已验证的结构化数据。当提取特定字段或返回 downstream 系统的数据时，自由形式的文本是不够的。

**工作原理：**当您提供模式作为响应格式时，模型的最终响应保证符合该模式。智能体运行模型/工具调用循环，直到模型完成调用工具，然后将最终响应强制转换为提供的格式。

#### 定义格式

模式定义指导模型。字段名称、类型和描述指定输出应遵循的确切格式。

```python  theme={null}
from pydantic import BaseModel, Field

class CustomerSupportTicket(BaseModel):
    """从客户消息中提取的结构化票据信息。"""

    category: str = Field(
        description="问题类别：'billing'（账单）、'technical'（技术）、'account'（账户）或'product'（产品）"
    )
    priority: str = Field(
        description="紧急程度：'low'（低）、'medium'（中）、'high'（高）或'critical'（紧急）"
    )
    summary: str = Field(
        description="客户问题的单句摘要"
    )
    customer_sentiment: str = Field(
        description="客户情绪：'frustrated'（沮丧）、'neutral'（中性）或'satisfied'（满意）"
    )
```

#### 选择格式

动态响应格式选择根据用户偏好、对话阶段或角色调整模式——在早期返回简单格式，随着复杂度增加返回详细格式。

<Tabs>
  <Tab title="状态">
    根据对话状态配置结构化输出：

    ```python  theme={null}
    from langchain.agents import create_agent
    from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse
    from pydantic import BaseModel, Field
    from typing import Callable

    class SimpleResponse(BaseModel):
        """用于早期对话的简单响应。"""
        answer: str = Field(description="简要回答")

    class DetailedResponse(BaseModel):
        """用于已建立对话的详细响应。"""
        answer: str = Field(description="详细回答")
        reasoning: str = Field(description="推理说明")
        confidence: float = Field(description="0-1的置信度分数")

    @wrap_model_call
    def state_based_output(
        request: ModelRequest,
        handler: Callable[[ModelRequest], ModelResponse]
    ) -> ModelResponse:
        """根据状态选择输出格式。"""
        # request.messages 是 request.state["messages"] 的快捷方式
        message_count = len(request.messages)  # [!code highlight]

        if message_count < 3:
            # 早期对话 - 使用简单格式
            request = request.override(response_format=SimpleResponse)  # [!code highlight]
        else:
            # 已建立对话 - 使用详细格式
            request = request.override(response_format=DetailedResponse)  # [!code highlight]

        return handler(request)

    agent = create_agent(
        model="openai:gpt-4o",
        tools=[...],
        middleware=[state_based_output]
    )
    ```
  </Tab>

  <Tab title="存储">
    根据存储中的用户偏好配置输出格式：

    ```python  theme={null}
    from dataclasses import dataclass
    from langchain.agents import create_agent
    from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse
    from pydantic import BaseModel, Field
    from typing import Callable
    from langgraph.store.memory import InMemoryStore

    @dataclass
    class Context:
        user_id: str

    class VerboseResponse(BaseModel):
        """带有详细信息的冗长响应。"""
        answer: str = Field(description="详细回答")
        sources: list[str] = Field(description="使用的来源")

    class ConciseResponse(BaseModel):
        """简洁响应。"""
        answer: str = Field(description="简要回答")

    @wrap_model_call
    def store_based_output(
        request: ModelRequest,
        handler: Callable[[ModelRequest], ModelResponse]
    ) -> ModelResponse:
        """根据存储偏好选择输出格式。"""
        user_id = request.runtime.context.user_id

        # 从存储读取：获取用户偏好的响应风格
        store = request.runtime.store
        user_prefs = store.get(("preferences",), user_id)

        if user_prefs:
            style = user_prefs.value.get("response_style", "concise")
            if style == "verbose":
                request = request.override(response_format=VerboseResponse)
            else:
                request = request.override(response_format=ConciseResponse)

        return handler(request)

    agent = create_agent(
        model="openai:gpt-4o",
        tools=[...],
        middleware=[store_based_output],
        context_schema=Context,
        store=InMemoryStore()
    )
    ```
  </Tab>

  <Tab title="运行时上下文">
    根据运行时上下文（如用户角色或环境）配置输出格式：

    ```python  theme={null}
    from dataclasses import dataclass
    from langchain.agents import create_agent
    from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse
    from pydantic import BaseModel, Field
    from typing import Callable

    @dataclass
    class Context:
        user_role: str
        environment: str

    class AdminResponse(BaseModel):
        """带有技术细节的管理员响应。"""
        answer: str = Field(description="回答")
        debug_info: dict = Field(description="调试信息")
        system_status: str = Field(description="系统状态")

    class UserResponse(BaseModel):
        """用于普通用户的简单响应。"""
        answer: str = Field(description="回答")

    @wrap_model_call
    def context_based_output(
        request: ModelRequest,
        handler: Callable[[ModelRequest], ModelResponse]
    ) -> ModelResponse:
        """根据运行时上下文选择输出格式。"""
        # 从运行时上下文读取：用户角色和环境
        user_role = request.runtime.context.user_role
        environment = request.runtime.context.environment

        if user_role == "admin" and environment == "production":
            # 生产环境管理员获得详细输出
            request = request.override(response_format=AdminResponse)
        else:
            # 普通用户获得简单输出
            request = request.override(response_format=UserResponse)

        return handler(request)

    agent = create_agent(
        model="openai:gpt-4o",
        tools=[...],
        middleware=[context_based_output],
        context_schema=Context
    )
    ```
  </Tab>
</Tabs>

## 工具上下文

工具的特殊之处在于它们既读取又写入上下文。

在最基本的情况下，当工具执行时，它接收LLM的请求参数并返回工具消息。工具执行其工作并产生结果。

工具还可以获取模型完成和完成任务所需的重要信息。

### 读取

大多数实际世界的工具需要的不仅仅是LLM的参数。它们需要用户ID进行数据库查询、API密钥访问外部服务，或当前会话状态来做决策。工具从状态、存储和运行时上下文中读取以访问这些信息。

<Tabs>
  <Tab title="状态">
    从状态中读取以检查当前会话信息：

    ```python  theme={null}
    from langchain.tools import tool, ToolRuntime
    from langchain.agents import create_agent

    @tool
    def check_authentication(
        runtime: ToolRuntime
    ) -> str:
        """检查用户是否已认证。"""
        # 从状态读取：检查当前认证状态
        current_state = runtime.state
        is_authenticated = current_state.get("authenticated", False)

        if is_authenticated:
            return "用户已认证"
        else:
            return "用户未认证"

    agent = create_agent(
        model="openai:gpt-4o",
        tools=[check_authentication]
    )
    ```
  </Tab>

  <Tab title="存储">
    从存储中读取以访问持久化的用户偏好：

    ```python  theme={null}
    from dataclasses import dataclass
    from langchain.tools import tool, ToolRuntime
    from langchain.agents import create_agent
    from langgraph.store.memory import InMemoryStore

    @dataclass
    class Context:
        user_id: str

    @tool
    def get_preference(
        preference_key: str,
        runtime: ToolRuntime[Context]
    ) -> str:
        """从存储获取用户偏好。"""
        user_id = runtime.context.user_id

        # 从存储读取：获取现有偏好
        store = runtime.store
        existing_prefs = store.get(("preferences",), user_id)

        if existing_prefs:
            value = existing_prefs.value.get(preference_key)
            return f"{preference_key}: {value}" if value else f"未设置{preference_key}的偏好"
        else:
            return "未找到偏好"

    agent = create_agent(
        model="openai:gpt-4o",
        tools=[get_preference],
        context_schema=Context,
        store=InMemoryStore()
    )
    ```
  </Tab>

  <Tab title="运行时上下文">
    从运行时上下文中读取配置（如API密钥和用户ID）：

    ```python  theme={null}
    from dataclasses import dataclass
    from langchain.tools import tool, ToolRuntime
    from langchain.agents import create_agent

    @dataclass
    class Context:
        user_id: str
        api_key: str
        db_connection: str

    @tool
    def fetch_user_data(
        query: str,
        runtime: ToolRuntime[Context]
    ) -> str:
        """使用运行时上下文配置获取数据。"""
        # 从运行时上下文读取：获取API密钥和数据库连接
        user_id = runtime.context.user_id
        api_key = runtime.context.api_key
        db_connection = runtime.context.db_connection

        # 使用配置获取数据
        results = perform_database_query(db_connection, query, api_key)

        return f"为用户{user_id}找到{len(results)}个结果"

    agent = create_agent(
        model="openai:gpt-4o",
        tools=[fetch_user_data],
        context_schema=Context
    )

    # 使用运行时上下文调用
    result = agent.invoke(
        {"messages": [{"role": "user", "content": "获取我的数据"}]},
        context=Context(
            user_id="user_123",
            api_key="sk-...",
            db_connection="postgresql://..."
        )
    )
    ```
  </Tab>
</Tabs>

### 写入

工具结果可用于帮助智能体完成给定任务。工具既可以向模型直接返回结果，
又可以更新智能体的记忆，使重要上下文对未来步骤可用。

<Tabs>
  <Tab title="状态">
    使用命令写入状态以跟踪特定于会话的信息：

    ```python  theme={null}
    from langchain.tools import tool, ToolRuntime
    from langchain.agents import create_agent
    from langgraph.types import Command

    @tool
    def authenticate_user(
        password: str,
        runtime: ToolRuntime
    ) -> Command:
        """认证用户并更新状态。"""
        # 执行认证（简化版）
        if password == "correct":
            # 使用命令写入状态：标记为已认证
            return Command(
                update={"authenticated": True},
            )
        else:
            return Command(update={"authenticated": False})

    agent = create_agent(
        model="openai:gpt-4o",
        tools=[authenticate_user]
    )
    ```
  </Tab>

  <Tab title="存储">
    写入存储以跨会话持久化数据：

    ```python  theme={null}
    from dataclasses import dataclass
    from langchain.tools import tool, ToolRuntime
    from langchain.agents import create_agent
    from langgraph.store.memory import InMemoryStore

    @dataclass
    class Context:
        user_id: str

    @tool
    def save_preference(
        preference_key: str,
        preference_value: str,
        runtime: ToolRuntime[Context]
    ) -> str:
        """将用户偏好保存到存储。"""
        user_id = runtime.context.user_id

        # 读取现有偏好
        store = runtime.store
        existing_prefs = store.get(("preferences",), user_id)

        # 与新偏好合并
        prefs = existing_prefs.value if existing_prefs else {}
        prefs[preference_key] = preference_value

        # 写入存储：保存更新的偏好
        store.put(("preferences",), user_id, prefs)

        return f"已保存偏好：{preference_key} = {preference_value}"

    agent = create_agent(
        model="openai:gpt-4o",
        tools=[save_preference],
        context_schema=Context,
        store=InMemoryStore()
    )
    ```
  </Tab>
</Tabs>

有关在工具中访问状态、存储和运行时上下文的全面示例，请参见[工具](/oss/python/langchain/tools)。

## 生命周期上下文

控制核心智能体步骤**之间**发生的事情——拦截数据流以实现横切关注点，如摘要、护栏和日志记录。

正如你在[模型上下文](#模型上下文)和[工具上下文](#工具上下文)中看到的，[中间件](/oss/python/langchain/middleware)是使上下文工程实用的机制。中间件允许你钩入智能体生命周期的任何步骤，并：

1. **更新上下文** - 修改状态和存储以持久化更改、更新对话历史或保存见解
2. **在生命周期中跳转** - 根据上下文移动到智能体周期的不同步骤（例如，如果满足条件则跳过工具执行，使用修改的上下文重复模型调用）

<div style={{ display: "flex", justifyContent: "center" }}>
  <img src="https://mintcdn.com/langchain-5e9cc07a/RAP6mjwE5G00xYsA/oss/images/middleware_final.png?fit=max&auto=format&n=RAP6mjwE5G00xYsA&q=85&s=eb4404b137edec6f6f0c8ccb8323eaf1" alt="智能体循环中的中间件钩子" className="rounded-lg" data-og-width="500" width="500" data-og-height="560" height="560" data-path="oss/images/middleware_final.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/RAP6mjwE5G00xYsA/oss/images/middleware_final.png?w=280&fit=max&auto=format&n=RAP6mjwE5G00xYsA&q=85&s=483413aa87cf93323b0f47c0dd5528e8 280w, https://mintcdn.com/langchain-5e9cc07a/RAP6mjwE5G00xYsA/oss/images/middleware_final.png?w=560&fit=max&auto=format&n=RAP6mjwE5G00xYsA&q=85&s=41b7dd647447978ff776edafe5f42499 560w, https://mintcdn.com/langchain-5e9cc07a/RAP6mjwE5G00xYsA/oss/images/middleware_final.png?w=840&fit=max&auto=format&n=RAP6mjwE5G00xYsA&q=85&s=e9b14e264f68345de08ae76f032c52d4 840w, https://mintcdn.com/langchain-5e9cc07a/RAP6mjwE5G00xYsA/oss/images/middleware_final.png?w=1100&fit=max&auto=format&n=RAP6mjwE5G00xYsA&q=85&s=ec45e1932d1279b1beee4a4b016b473f 1100w, https://mintcdn.com/langchain-5e9cc07a/RAP6mjwE5G00xYsA/oss/images/middleware_final.png?w=1650&fit=max&auto=format&n=RAP6mjwE5G00xYsA&q=85&s=3bca5ebf8aa56632b8a9826f7f112e57 1650w, https://mintcdn.com/langchain-5e9cc07a/RAP6mjwE5G00xYsA/oss/images/middleware_final.png?w=2500&fit=max&auto=format&n=RAP6mjwE5G00xYsA&q=85&s=437f141d1266f08a95f030c2804691d9 2500w" />
</div>

### 示例：摘要

最常见的生命周期模式之一是在对话历史过长时自动压缩对话历史。与[模型上下文](#消息)中显示的临时消息修剪不同，摘要**持久性地更新状态**——永久地将旧消息替换为摘要，为所有未来轮次保存。

LangChain为此提供了内置的中间件：

```python  theme={null}
from langchain.agents import create_agent
from langchain.agents.middleware import SummarizationMiddleware

agent = create_agent(
    model="openai:gpt-4o",
    tools=[...],
    middleware=[
        SummarizationMiddleware(
            model="openai:gpt-4o-mini",
            max_tokens_before_summary=4000,  # 在4000个令牌时触发摘要
            messages_to_keep=20,  # 摘要后保留最后20条消息
        ),
    ],
)
```

当对话超过令牌限制时，`SummarizationMiddleware`自动：

1. 使用单独的LLM调用对较旧的消息进行摘要
2. 在状态中将它们替换为摘要消息（永久性）
3. 保持最近的消息完整以便上下文使用

摘要对话历史被永久更新——未来轮次将看到摘要而不是原始消息。

<Note>
  有关内置中间件的完整列表、可用钩子以及如何创建自定义中间件，请参见[中间件文档](/oss/python/langchain/middleware)。
</Note>

## 最佳实践

1. **从简单开始** - 从静态提示词和工具开始，仅在需要时添加动态功能
2. **增量测试** - 一次添加一个上下文工程功能
3. **监控性能** - 跟踪模型调用、令牌使用和延迟
4. **使用内置中间件** - 利用[`SummarizationMiddleware`](/oss/python/langchain/middleware#summarization)、[`LLMToolSelectorMiddleware`](/oss/python/langchain/middleware#llm-tool-selector)等
5. **记录上下文策略** - 明确说明正在传递什么上下文以及为什么
6. **理解临时性与持久性**：模型上下文更改是临时性的（每次调用），而生命周期上下文更改会持久化到状态

## 相关资源

* [上下文概念概述](/oss/python/concepts/context) - 了解上下文类型及其使用时机
* [中间件](/oss/python/langchain/middleware) - 完整的中间件指南
* [工具](/oss/python/langchain/tools) - 工具创建和上下文访问
* [记忆](/oss/python/concepts/memory) - 短期和长期记忆模式
* [智能体](/oss/python/langchain/agents) - 核心智能体概念

***

<Callout icon="pen-to-square" iconType="regular">
  [在GitHub上编辑此页面的源代码。](https://github.com/langchain-ai/docs/edit/main/src/oss/langchain/context-engineering.mdx)
</Callout>

<Tip icon="terminal" iconType="regular">
  [以编程方式连接这些文档](/use-these-docs)到Claude、VSCode等，通过MCP获得实时答案。
</Tip>