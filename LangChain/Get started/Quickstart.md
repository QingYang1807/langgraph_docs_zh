# 快速入门

这个快速入门教程将在几分钟内带你从简单设置到创建一个功能完备的AI代理。

## 构建基础代理

首先创建一个能够回答问题和调用工具的简单代理。该代理将使用Claude Sonnet 4.5作为语言模型，一个基本天气函数作为工具，以及一个简单的提示来指导其行为。

```python
from langchain.agents import create_agent

def get_weather(city: str) -> str:
    """获取给定城市的天气。"""
    return f"{city}的天气总是晴朗的！"

agent = create_agent(
    model="anthropic:claude-sonnet-4-5",
    tools=[get_weather],
    system_prompt="你是一个有用的助手",
)

# 运行代理
agent.invoke(
    {"messages": [{"role": "user", "content": "sf的天气怎么样"}]}
)
```

<Info>
  对于这个示例，你需要设置一个[Claude (Anthropic)](https://www.anthropic.com/)账户并获取API密钥。然后，在终端中设置`ANTHROPIC_API_KEY`环境变量。
</Info>

## 构建实际应用代理

接下来，构建一个实用的天气预报代理，展示关键的生产应用概念：

1. **详细的系统提示**以获得更好的代理行为
2. **创建工具**以集成外部数据
3. **模型配置**以获得一致的响应
4. **结构化输出**以获得可预测的结果
5. **对话记忆**以实现类似聊天的交互
6. **创建并运行代理**以创建一个功能完备的代理

让我们逐步介绍每个步骤：

<Steps>
  <Step title="定义系统提示">
    系统提示定义了你代理的角色和行为。保持具体和可操作性：

    ```python
    SYSTEM_PROMPT = """你是一个专家天气预报员，说话时喜欢使用双关语。

    你可以访问两个工具：

    - get_weather_for_location: 使用此工具获取特定位置的天气
    - get_user_location: 使用此工具获取用户的位置

    如果用户询问天气，确保你知道位置。如果你能从问题中看出他们是指他们所在的位置，请使用get_user_location工具来查找他们的位置。"""
    ```
  </Step>

  <Step title="创建工具">
    [工具](/oss/python/langchain/tools)允许模型通过调用你定义的函数与外部系统交互。
    工具可以依赖于[运行时上下文](/oss/python/langchain/runtime)，还可以与[代理记忆](/oss/python/langchain/short-term-memory)交互。

    注意下面的`get_user_location`工具如何使用运行时上下文：

    ```python
    from dataclasses import dataclass
    from langchain.tools import tool, ToolRuntime

    @tool
    def get_weather_for_location(city: str) -> str:
        """获取给定城市的天气。"""
        return f"{city}的天气总是晴朗的！"

    @dataclass
    class Context:
        """自定义运行时上下文模式。"""
        user_id: str

    @tool
    def get_user_location(runtime: ToolRuntime[Context, Any]) -> str:
        """根据用户ID检索用户信息。"""
        user_id = runtime.context.user_id
        return "Florida" if user_id == "1" else "SF"
    ```

    <Tip>
      工具应该有良好的文档：它们的名称、描述和参数名称将成为模型提示的一部分。
      LangChain的[`@tool`装饰器](https://reference.langchain.com/python/langchain/tools/#langchain.tools.tool)添加元数据并允许通过`ToolRuntime`参数进行运行时注入。
    </Tip>
  </Step>

  <Step title="配置你的模型">
    为你的用例设置合适的[参数](/oss/python/langchain/models#参数)来配置你的[语言模型](/oss/python/langchain/models)：

    ```python
    from langchain.chat_models import init_chat_model

    model = init_chat_model(
        "anthropic:claude-sonnet-4-5",
        temperature=0.5,
        timeout=10,
        max_tokens=1000
    )
    ```
  </Step>

  <Step title="定义响应格式">
    如果需要代理响应匹配特定模式，可以选择定义结构化响应格式。

    ```python
    from dataclasses import dataclass

    # 这里我们使用dataclass，但也支持Pydantic模型。
    @dataclass
    class ResponseFormat:
        """代理的响应模式。"""
        # 一个双关语响应（始终需要）
        punny_response: str
        # 如果有，任何关于天气的有趣信息
        weather_conditions: str | None = None
    ```
  </Step>

  <Step title="添加记忆">
    为你的代理添加[记忆](/oss/python/langchain/short-term-memory)以在交互之间保持状态。这使
    代理能够记住之前的对话和上下文。

    ```python
    from langgraph.checkpoint.memory import InMemorySaver

    checkpointer = InMemorySaver()
    ```

    <Info>
      在生产环境中，使用保存到数据库的持久化检查点器。
      有关更多详细信息，请参阅[添加和管理记忆](/oss/python/langgraph/add-memory#manage-short-term-memory)。
    </Info>
  </Step>

  <Step title="创建并运行代理">
    现在用所有组件组装你的代理并运行它！

    ```python
    agent = create_agent(
        model=model,
        system_prompt=SYSTEM_PROMPT,
        tools=[get_user_location, get_weather_for_location],
        context_schema=Context,
        response_format=ResponseFormat,
        checkpointer=checkpointer
    )

    # `thread_id`是给定对话的唯一标识符。
    config = {"configurable": {"thread_id": "1"}}

    response = agent.invoke(
        {"messages": [{"role": "user", "content": "外面的天气怎么样？"}]},
        config=config,
        context=Context(user_id="1")
    )

    print(response['structured_response'])
    # ResponseFormat(
    #     punny_response="佛罗里达今天又是一个'阳光明媚'的日子！阳光全天都在播放'日光广播'金曲！我认为这是进行'阳光庆祝'的完美天气！如果你希望下雨，恐怕那个想法已经'被冲走了' - 天气预报仍然'清晰'地棒极了！",
    #     weather_conditions="佛罗里达的天气总是晴朗的！"
    # )


    # 注意我们可以使用相同的`thread_id`继续对话。
    response = agent.invoke(
        {"messages": [{"role": "user", "content": "谢谢！"}]},
        config=config,
        context=Context(user_id="1")
    )

    print(response['structured_response'])
    # ResponseFormat(
    #     punny_response="'雷厉风行'地欢迎你！帮助你了解'当前'天气总是'轻而易举'的。我只是在'云'中等待，随时准备'淋浴'你更多的天气预报。在佛罗里达的阳光中度过一个'阳光灿烂'的日子吧！",
    #     weather_conditions=None
    # )
    ```
  </Step>
</Steps>

<Expandable title="完整示例代码">
  ```python
  from dataclasses import dataclass

  from langchain.agents import create_agent
  from langchain.chat_models import init_chat_model
  from langchain.tools import tool, ToolRuntime
  from langgraph.checkpoint.memory import InMemorySaver


  # 定义系统提示
  SYSTEM_PROMPT = """你是一个专家天气预报员，说话时喜欢使用双关语。

  你可以访问两个工具：

  - get_weather_for_location: 使用此工具获取特定位置的天气
  - get_user_location: 使用此工具获取用户的位置

  如果用户询问天气，确保你知道位置。如果你能从问题中看出他们是指他们所在的位置，请使用get_user_location工具来查找他们的位置。"""

  # 定义上下文模式
  @dataclass
  class Context:
      """自定义运行时上下文模式。"""
      user_id: str

  # 定义工具
  @tool
  def get_weather_for_location(city: str) -> str:
      """获取给定城市的天气。"""
      return f"{city}的天气总是晴朗的！"

  @tool
  def get_user_location(runtime: ToolRuntime[Context]) -> str:
      """根据用户ID检索用户信息。"""
      user_id = runtime.context.user_id
      return "Florida" if user_id == "1" else "SF"

  # 配置模型
  model = init_chat_model(
      "anthropic:claude-sonnet-4-5",
      temperature=0
  )

  # 定义响应格式
  @dataclass
  class ResponseFormat:
      """代理的响应模式。"""
      # 一个双关语响应（始终需要）
      punny_response: str
      # 如果有，任何关于天气的有趣信息
      weather_conditions: str | None = None

  # 设置记忆
  checkpointer = InMemorySaver()

  # 创建代理
  agent = create_agent(
      model=model,
      system_prompt=SYSTEM_PROMPT,
      tools=[get_user_location, get_weather_for_location],
      context_schema=Context,
      response_format=ResponseFormat,
      checkpointer=checkpointer
  )

  # 运行代理
  # `thread_id`是给定对话的唯一标识符。
  config = {"configurable": {"thread_id": "1"}}

  response = agent.invoke(
      {"messages": [{"role": "user", "content": "外面的天气怎么样？"}]},
      config=config,
      context=Context(user_id="1")
  )

  print(response['structured_response'])
  # ResponseFormat(
  #     punny_response="佛罗里达今天又是一个'阳光明媚'的日子！阳光全天都在播放'日光广播'金曲！我认为这是进行'阳光庆祝'的完美天气！如果你希望下雨，恐怕那个想法已经'被冲走了' - 天气预报仍然'清晰'地棒极了！",
  #     weather_conditions="佛罗里达的天气总是晴朗的！"
  # )


  # 注意我们可以使用相同的`thread_id`继续对话。
  response = agent.invoke(
      {"messages": [{"role": "user", "content": "谢谢！"}]},
      config=config,
      context=Context(user_id="1")
  )

  print(response['structured_response'])
  # ResponseFormat(
  #     punny_response="'雷厉风行'地欢迎你！帮助你了解'当前'天气总是'轻而易举'的。我只是在'云'中等待，随时准备'淋浴'你更多的天气预报。在佛罗里达的阳光中度过一个'阳光灿烂'的日子吧！",
  #     weather_conditions=None
  # )
  ```
</Expandable>

恭喜！你现在拥有一个能够：

* **理解上下文**并记住对话
* **智能使用多个工具**
* **以一致格式提供结构化响应**
* **通过上下文处理用户特定信息**
* **在交互之间保持对话状态**

的AI代理！

***

<Callout icon="pen-to-square" iconType="regular">
  [在GitHub上编辑此页面的源代码。](https://github.com/langchain-ai/docs/edit/main/src/oss/langchain/quickstart.mdx)
</Callout>

<Tip icon="terminal" iconType="regular">
  [通过编程方式连接这些文档](/use-these-docs)到Claude、VSCode等，通过MCP获得实时答案。
</Tip>