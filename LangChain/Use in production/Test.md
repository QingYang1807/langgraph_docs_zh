# 测试

代理应用程序让大语言模型(LLM)自行决定解决问题的下一步骤。这种灵活性很强大，但模型的黑盒特性使得很难预测对代理某一部分的调整会如何影响其他部分。要构建就绪生产的代理，彻底的测试是必不可少的。

测试代理有几种方法：

* **[单元测试](#unit-testing)**：使用内存中的模拟对象独立测试代理的小型、确定性部分，以便快速和确定性地断言精确行为。

* **[集成测试](#integration-testing)**：使用真实的网络调用测试代理，以确认组件协同工作，凭据和模式一致，且延迟在可接受范围内。

代理应用程序往往更倾向于集成测试，因为它们将多个组件链接在一起，并且必须处理由于LLM的非确定性导致的不可靠性。

## 单元测试

### 模拟聊天模型

对于不需要API调用的逻辑，您可以使用内存中的存根来模拟响应。

LangChain提供了[`GenericFakeChatModel`](https://python.langchain.com/api_reference/core/language_models/langchain_core.language_models.fake_chat_models.GenericFakeChatModel.html)用于模拟文本响应。它接受一个响应迭代器(AIMessages或字符串)，每次调用返回一个。它支持常规和流式使用。

```python
from langchain_core.language_models.fake_chat_models import GenericFakeChatModel

model = GenericFakeChatModel(messages=iter([
    AIMessage(content="", tool_calls=[ToolCall(name="foo", args={"bar": "baz"}, id="call_1")]),
    "bar"
]))

model.invoke("hello")
# AIMessage(content='', ..., tool_calls=[{'name': 'foo', 'args': {'bar': 'baz'}, 'id': 'call_1', 'type': 'tool_call'}])
```

如果我们再次调用模型，它将返回迭代器中的下一个项目：

```python
model.invoke("hello, again!")
# AIMessage(content='bar', ...)
```

### InMemorySaver检查点

为了在测试期间启用持久性，您可以使用[`InMemorySaver`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.memory.InMemorySaver)检查点。这使您可以模拟多轮对话以测试依赖于状态的行为：

```python
from langgraph.checkpoint.memory import InMemorySaver

agent = create_agent(
    model,
    tools=[],
    checkpointer=InMemorySaver()
)

# 第一次调用
agent.invoke(HumanMessage(content="I live in Sydney, Australia."))

# 第二次调用：第一条消息被持久化(悉尼位置)，因此模型返回GMT+10时间
agent.invoke(HumanMessage(content="What's my local time?"))
```

## 集成测试

许多代理行为只有在使用真实LLM时才会显现，例如代理决定调用哪个工具、如何格式化响应，或者提示修改是否影响整个执行轨迹。LangChain的[`agentevals`](https://github.com/langchain-ai/agentevals)包提供了专门用于测试带有实时模型的代理轨迹的评估器。

AgentEvals允许您通过执行**轨迹匹配**或使用**LLM法官**来轻松评估代理的轨迹(消息的确切序列，包括工具调用)：

<Card title="轨迹匹配" icon="equals" arrow="true" href="#trajectory-match-evaluator">
  为给定输入硬编码参考轨迹，并通过逐步比较来验证运行。

  适用于测试定义良好的工作流程，您知道预期行为。当您对应该调用哪些工具以及按什么顺序调用有特定期望时使用。这种方法是确定性的、快速的，并且成本效益高，因为它不需要额外的LLM调用。
</Card>

<Card title="LLM作为法官" icon="gavel" arrow="true" href="#llm-as-judge-evaluator">
  使用LLM来定性地验证代理的执行轨迹。"法官"LLM根据提示评分标准(可以包括参考轨迹)审查代理的决定。

  更加灵活，可以评估效率、适当性等细微方面，但需要LLM调用且确定性较低。当您想评估代理轨迹的整体质量和合理性，而不严格要求工具调用或排序要求时使用。
</Card>

### 安装AgentEvals

```bash
pip install agentevals
```

或者直接克隆[AgentEvals仓库](https://github.com/langchain-ai/agentevals)。

### 轨迹匹配评估器

AgentEvals提供了`create_trajectory_match_evaluator`函数，用于将代理的轨迹与参考轨迹进行匹配。有四种模式可供选择：

| 模式 | 描述 | 用例 |
| --- | --- | --- |
| `strict` | 消息和工具调用的完全相同顺序匹配 | 测试特定序列(例如，授权前的策略查找) |
| `unordered` | 允许以任何顺序使用相同的工具调用 | 验证顺序不重要时的信息检索 |
| `subset` | 代理只调用参考中的工具(无额外工具) | 确保代理不超过预期范围 |
| `superset` | 代理至少调用参考工具(允许额外工具) | 验证是否采取了所需的最小行动 |

<Accordion title="严格匹配">
  `strict`模式确保轨迹包含相同顺序的相同消息和工具调用，尽管它允许消息内容有所不同。当您需要强制执行特定操作序列时很有用，例如要求在授权操作前进行策略查找。

  ```python
  from langchain.agents import create_agent
  from langchain.tools import tool
  from langchain.messages import HumanMessage, AIMessage, ToolMessage
  from agentevals.trajectory.match import create_trajectory_match_evaluator

  @tool
  def get_weather(city: str):
      """Get weather information for a city."""
      return f"It's 75 degrees and sunny in {city}."

  agent = create_agent("openai:gpt-4o", tools=[get_weather])

  evaluator = create_trajectory_match_evaluator(
      trajectory_match_mode="strict",
  )

  def test_weather_tool_called_strict():
      result = agent.invoke({
          "messages": [HumanMessage(content="What's the weather in San Francisco?")]
      })

      reference_trajectory = [
          HumanMessage(content="What's the weather in San Francisco?"),
          AIMessage(content="", tool_calls=[
              {"id": "call_1", "name": "get_weather", "args": {"city": "San Francisco"}}
          ]),
          ToolMessage(content="It's 75 degrees and sunny in San Francisco.", tool_call_id="call_1"),
          AIMessage(content="The weather in San Francisco is 75 degrees and sunny."),
      ]

      evaluation = evaluator(
          outputs=result["messages"],
          reference_outputs=reference_trajectory
      )
      # {
      #     'key': 'trajectory_strict_match',
      #     'score': True,
      #     'comment': None,
      # }
      assert evaluation["score"] is True
  ```
</Accordion>

<Accordion title="无序匹配">
  `unordered`模式允许以任何顺序使用相同的工具调用，当您要验证检索了特定信息但不关心顺序时很有帮助。例如，代理可能需要检查城市的天气和事件，但顺序不重要。

  ```python
  from langchain.agents import create_agent
  from langchain.tools import tool
  from langchain.messages import HumanMessage, AIMessage, ToolMessage
  from agentevals.trajectory.match import create_trajectory_match_evaluator


  @tool
  def get_weather(city: str):
      """Get weather information for a city."""
      return f"It's 75 degrees and sunny in {city}."

  @tool
  def get_events(city: str):
      """Get events happening in a city."""
      return f"Concert at the park in {city} tonight."

  agent = create_agent("openai:gpt-4o", tools=[get_weather, get_events])

  evaluator = create_trajectory_match_evaluator(
      trajectory_match_mode="unordered",
  )

  def test_multiple_tools_any_order():
      result = agent.invoke({
          "messages": [HumanMessage(content="What's happening in SF today?")]
      })

      # 参考显示工具调用顺序与实际执行不同
      reference_trajectory = [
          HumanMessage(content="What's happening in SF today?"),
          AIMessage(content="", tool_calls=[
              {"id": "call_1", "name": "get_events", "args": {"city": "SF"}},
              {"id": "call_2", "name": "get_weather", "args": {"city": "SF"}},
          ]),
          ToolMessage(content="Concert at the park in SF tonight.", tool_call_id="call_1"),
          ToolMessage(content="It's 75 degrees and sunny in SF.", tool_call_id="call_2"),
          AIMessage(content="Today in SF: 75 degrees and sunny with a concert at the park tonight."),
      ]

      evaluation = evaluator(
          outputs=result["messages"],
          reference_outputs=reference_trajectory,
      )
      # {
      #     'key': 'trajectory_unordered_match',
      #     'score': True,
      # }
      assert evaluation["score"] is True
  ```
</Accordion>

<Accordion title="子集和超集匹配">
  `superset`和`subset`模式匹配部分轨迹。`superset`模式验证代理至少调用了参考轨迹中的工具，允许额外的工具调用。`subset`模式确保代理没有调用参考轨迹之外的任何工具。

  ```python
  from langchain.agents import create_agent
  from langchain.tools import tool
  from langchain.messages import HumanMessage, AIMessage, ToolMessage
  from agentevals.trajectory.match import create_trajectory_match_evaluator


  @tool
  def get_weather(city: str):
      """Get weather information for a city."""
      return f"It's 75 degrees and sunny in {city}."

  @tool
  def get_detailed_forecast(city: str):
      """Get detailed weather forecast for a city."""
      return f"Detailed forecast for {city}: sunny all week."

  agent = create_agent("openai:gpt-4o", tools=[get_weather, get_detailed_forecast])

  evaluator = create_trajectory_match_evaluator(
      trajectory_match_mode="superset",
  )

  def test_agent_calls_required_tools_plus_extra():
      result = agent.invoke({
          "messages": [HumanMessage(content="What's the weather in Boston?")]
      })

      # 参考只要求get_weather，但代理可能调用额外工具
      reference_trajectory = [
          HumanMessage(content="What's the weather in Boston?"),
          AIMessage(content="", tool_calls=[
              {"id": "call_1", "name": "get_weather", "args": {"city": "Boston"}},
          ]),
          ToolMessage(content="It's 75 degrees and sunny in Boston.", tool_call_id="call_1"),
          AIMessage(content="The weather in Boston is 75 degrees and sunny."),
      ]

      evaluation = evaluator(
          outputs=result["messages"],
          reference_outputs=reference_trajectory,
      )
      # {
      #     'key': 'trajectory_superset_match',
      #     'score': True,
      #     'comment': None,
      # }
      assert evaluation["score"] is True
  ```
</Accordion>

<Info>
  您还可以设置`tool_args_match_mode`属性和/或`tool_args_match_overrides`来自定义评估器如何考虑实际轨迹与参考轨迹中工具调用的相等性。默认情况下，只有具有相同参数的相同工具调用才被视为相等。访问[仓库](https://github.com/langchain-ai/agentevals?tab=readme-ov-file#tool-args-match-modes)了解更多详情。
</Info>

### LLM作为法官评估器

您还可以使用`create_trajectory_llm_as_judge`函数使用LLM评估代理的执行路径。与轨迹匹配评估器不同，它不需要参考轨迹，但如果有的话可以提供。

<Accordion title="没有参考轨迹">
  ```python
  from langchain.agents import create_agent
  from langchain.tools import tool
  from langchain.messages import HumanMessage, AIMessage, ToolMessage
  from agentevals.trajectory.llm import create_trajectory_llm_as_judge, TRAJECTORY_ACCURACY_PROMPT

  @tool
  def get_weather(city: str):
      """Get weather information for a city."""
      return f"It's 75 degrees and sunny in {city}."

  agent = create_agent("openai:gpt-4o", tools=[get_weather])

  evaluator = create_trajectory_llm_as_judge(
      model="openai:o3-mini",
      prompt=TRAJECTORY_ACCURACY_PROMPT,
  )

  def test_trajectory_quality():
      result = agent.invoke({
          "messages": [HumanMessage(content="What's the weather in Seattle?")]
      })

      evaluation = evaluator(
          outputs=result["messages"],
      )
      # {
      #     'key': 'trajectory_accuracy',
      #     'score': True,
      #     'comment': 'The provided agent trajectory is reasonable...'
      # }
      assert evaluation["score"] is True
  ```
</Accordion>

<Accordion title="有参考轨迹">
  如果您有参考轨迹，可以添加一个额外变量到您的提示中并传入参考轨迹。下面，我们使用预构建的`TRAJECTORY_ACCURACY_PROMPT_WITH_REFERENCE`提示并配置`reference_outputs`变量：

  ```python
  evaluator = create_trajectory_llm_as_judge(
      model="openai:o3-mini",
      prompt=TRAJECTORY_ACCURACY_PROMPT_WITH_REFERENCE,
  )
  evaluation = judge_with_reference(
      outputs=result["messages"],
      reference_outputs=reference_trajectory,
  )
  ```
</Accordion>

<Info>
  要获得对LLM如何评估轨迹的更多可配置性，请访问[仓库](https://github.com/langchain-ai/agentevals?tab=readme-ov-file#trajectory-llm-as-judge)。
</Info>

### 异步支持

所有`agentevals`评估器都支持Python asyncio。对于使用工厂函数的评估器，通过在`create_`后添加`async`来提供异步版本。

<Accordion title="异步法官和评估器示例">
  ```python
  from agentevals.trajectory.llm import create_async_trajectory_llm_as_judge, TRAJECTORY_ACCURACY_PROMPT
  from agentevals.trajectory.match import create_async_trajectory_match_evaluator

  async_judge = create_async_trajectory_llm_as_judge(
      model="openai:o3-mini",
      prompt=TRAJECTORY_ACCURACY_PROMPT,
  )

  async_evaluator = create_async_trajectory_match_evaluator(
      trajectory_match_mode="strict",
  )

  async def test_async_evaluation():
      result = await agent.ainvoke({
          "messages": [HumanMessage(content="What's the weather?")]
      })

      evaluation = await async_judge(outputs=result["messages"])
      assert evaluation["score"] is True
  ```
</Accordion>

## LangSmith集成

为了随时间跟踪实验，您可以将评估器结果记录到[LangSmith](https://smith.langchain.com/)，这是一个用于构建生产级LLM应用程序的平台，包括追踪、评估和实验工具。

首先，通过设置所需的环境变量来设置LangSmith：

```bash
export LANGSMITH_API_KEY="your_langsmith_api_key"
export LANGSMITH_TRACING="true"
```

LangSmith提供了两种主要的评估运行方法：[pytest](/langsmith/pytest)集成和`evaluate`函数。

<Accordion title="使用pytest集成">
  ```python
  import pytest
  from langsmith import testing as t
  from agentevals.trajectory.llm import create_trajectory_llm_as_judge, TRAJECTORY_ACCURACY_PROMPT

  trajectory_evaluator = create_trajectory_llm_as_judge(
      model="openai:o3-mini",
      prompt=TRAJECTORY_ACCURACY_PROMPT,
  )

  @pytest.mark.langsmith
  def test_trajectory_accuracy():
      result = agent.invoke({
          "messages": [HumanMessage(content="What's the weather in SF?")]
      })

      reference_trajectory = [
          HumanMessage(content="What's the weather in SF?"),
          AIMessage(content="", tool_calls=[
              {"id": "call_1", "name": "get_weather", "args": {"city": "SF"}},
          ]),
          ToolMessage(content="It's 75 degrees and sunny in SF.", tool_call_id="call_1"),
          AIMessage(content="The weather in SF is 75 degrees and sunny."),
      ]

      # 将输入、输出和参考输出记录到LangSmith
      t.log_inputs({})
      t.log_outputs({"messages": result["messages"]})
      t.log_reference_outputs({"messages": reference_trajectory})

      trajectory_evaluator(
          outputs=result["messages"],
          reference_outputs=reference_trajectory
      )
  ```

  使用pytest运行评估：

  ```bash
  pytest test_trajectory.py --langsmith-output
  ```

  结果将自动记录到LangSmith。
</Accordion>

<Accordion title="使用evaluate函数">
  或者，您可以在LangSmith中创建数据集并使用`evaluate`函数：

  ```python
  from langsmith import Client
  from agentevals.trajectory.llm import create_trajectory_llm_as_judge, TRAJECTORY_ACCURACY_PROMPT

  client = Client()

  trajectory_evaluator = create_trajectory_llm_as_judge(
      model="openai:o3-mini",
      prompt=TRAJECTORY_ACCURACY_PROMPT,
  )

  def run_agent(inputs):
      """您的代理函数，返回轨迹消息。"""
      return agent.invoke(inputs)["messages"]

  experiment_results = client.evaluate(
      run_agent,
      data="your_dataset_name",
      evaluators=[trajectory_evaluator]
  )
  ```

  结果将自动记录到LangSmith。
</Accordion>

<Tip>
  要了解有关评估代理的更多信息，请参阅[LangSmith文档](/langsmith/pytest)。
</Tip>

## 录制和重放HTTP调用

调用真实LLM API的集成测试可能既慢又昂贵，特别是在CI/CD管道中频繁运行时。我们建议使用录制HTTP请求和响应的库，然后在后续运行中重放它们，而不进行实际的网络调用。

您可以使用[`vcrpy`](https://pypi.org/project/vcrpy/1.5.2/)来实现这一点。如果您使用`pytest`，[`pytest-recording`插件](https://pypi.org/project/pytest-recording/)提供了一种以最少配置启用此功能的方法。请求/响应被录制在磁带中，然后在后续运行中用于模拟真实网络调用。

设置您的`conftest.py`文件以过滤磁带中的敏感信息：

```py
import pytest

@pytest.fixture(scope="session")
def vcr_config():
    return {
        "filter_headers": [
            ("authorization", "XXXX"),
            ("x-api-key", "XXXX"),
            # ... 其他您想要屏蔽的头部
        ],
        "filter_query_parameters": [
            ("api_key", "XXXX"),
            ("key", "XXXX"),
        ],
    }
```

然后配置您的项目以识别`vcr`标记：

<CodeGroup>
  ```ini
  [pytest]
  markers =
      vcr: record/replay HTTP via VCR
  addopts = --record-mode=once
  ```

  ```toml
  [tool.pytest.ini_options]
  markers = [
    "vcr: record/replay HTTP via VCR"
  ]
  addopts = "--record-mode=once"
  ```
</CodeGroup>

<Info>
  `--record-mode=once`选项在第一次运行时记录HTTP交互，在后续运行中重放它们。
</Info>

现在，只需使用`vcr`标记装饰您的测试：

```python
@pytest.mark.vcr()
def test_agent_trajectory():
    # ...
```

第一次运行此测试时，您的代理将进行真实的网络调用，pytest将在`tests/cassettes`目录中生成一个磁带文件`test_agent_trajectory.yaml`。后续运行将使用该磁带模拟真实网络调用，前提是代理的请求与之前运行没有变化。如果它们确实发生了变化，测试将失败，您需要删除磁带并重新运行测试以记录新的交互。

<Warning>
  当您修改提示、添加新工具或更改预期轨迹时，您保存的磁带将过时，现有测试**将失败**。您应该删除相应的磁带文件并重新运行测试以记录新的交互。
</Warning>

***

<Callout icon="pen-to-square" iconType="regular">
  [在GitHub上编辑此页面的源代码。](https://github.com/langchain-ai/docs/edit/main/src/oss/langchain/test.mdx)
</Callout>

<Tip icon="terminal" iconType="regular">
  [通过MCP以编程方式连接这些文档](/use-these-docs)到Claude、VSCode等，以获得实时答案。
</Tip>