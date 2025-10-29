# 使用函数式API

[**函数式API**](/oss/python/langgraph/functional-api)允许您以最小的现有代码更改，将LangGraph的核心功能（[持久化](/oss/python/langgraph/persistence)、[记忆](/oss/python/langgraph/add-memory)、[人在循环中](/oss/python/langgraph/interrupts)和[流式传输](/oss/python/langgraph/streaming)）添加到您的应用程序中。

<Tip>
  有关函数式API的概念信息，请参见[函数式API](/oss/python/langgraph/functional-api)。
</Tip>

## 创建简单的工作流

在定义`entrypoint`时，输入限制在函数的第一个参数。要传递多个输入，可以使用字典。

```python  theme={null}
@entrypoint(checkpointer=checkpointer)
def my_workflow(inputs: dict) -> int:
    value = inputs["value"]
    another_value = inputs["another_value"]
    ...

my_workflow.invoke({"value": 1, "another_value": 2})
```

<Accordion title="扩展示例：简单工作流">
  ```python  theme={null}
  import uuid
  from langgraph.func import entrypoint, task
  from langgraph.checkpoint.memory import InMemorySaver

  # 检查数字是否为偶数的任务
  @task
  def is_even(number: int) -> bool:
      return number % 2 == 0

  # 格式化消息的任务
  @task
  def format_message(is_even: bool) -> str:
      return "The number is even." if is_even else "The number is odd."

  # 创建用于持久化的检查点
  checkpointer = InMemorySaver()

  @entrypoint(checkpointer=checkpointer)
  def workflow(inputs: dict) -> str:
      """对数字进行分类的简单工作流。"""
      even = is_even(inputs["number"]).result()
      return format_message(even).result()

  # 使用唯一线程ID运行工作流
  config = {"configurable": {"thread_id": str(uuid.uuid4())}}
  result = workflow.invoke({"number": 7}, config=config)
  print(result)
  ```
</Accordion>

<Accordion title="扩展示例：使用LLM撰写文章">
  此示例演示了如何语法地使用`@task`和`@entrypoint`装饰器。如果提供了检查点，工作流结果将被持久化到检查点中。

  ```python  theme={null}
  import uuid
  from langchain.chat_models import init_chat_model
  from langgraph.func import entrypoint, task
  from langgraph.checkpoint.memory import InMemorySaver

  model = init_chat_model('openai:gpt-3.5-turbo')

  # 任务：使用LLM生成文章
  @task
  def compose_essay(topic: str) -> str:
      """关于给定主题生成文章。"""
      return model.invoke([
          {"role": "system", "content": "You are a helpful assistant that writes essays."},
          {"role": "user", "content": f"Write an essay about {topic}."}
      ]).content

  # 创建用于持久化的检查点
  checkpointer = InMemorySaver()

  @entrypoint(checkpointer=checkpointer)
  def workflow(topic: str) -> str:
      """使用LLM生成文章的简单工作流。"""
      return compose_essay(topic).result()

  # 执行工作流
  config = {"configurable": {"thread_id": str(uuid.uuid4())}}
  result = workflow.invoke("the history of flight", config=config)
  print(result)
  ```
</Accordion>

## 并行执行

可以通过并发调用任务并等待结果来并行执行任务。这对于提高IO密集型任务（例如调用LLM的API）的性能很有用。

```python  theme={null}
@task
def add_one(number: int) -> int:
    return number + 1

@entrypoint(checkpointer=checkpointer)
def graph(numbers: list[int]) -> list[str]:
    futures = [add_one(i) for i in numbers]
    return [f.result() for f in futures]
```

<Accordion title="扩展示例：并行LLM调用">
  此示例演示了如何使用`@task`并行运行多个LLM调用。每个调用生成一个关于不同主题的段落，并将结果合并为单个文本输出。

  ```python  theme={null}
  import uuid
  from langchain.chat_models import init_chat_model
  from langgraph.func import entrypoint, task
  from langgraph.checkpoint.memory import InMemorySaver

  # 初始化LLM模型
  model = init_chat_model("openai:gpt-3.5-turbo")

  # 生成关于给定主题的段落的任务
  @task
  def generate_paragraph(topic: str) -> str:
      response = model.invoke([
          {"role": "system", "content": "You are a helpful assistant that writes educational paragraphs."},
          {"role": "user", "content": f"Write a paragraph about {topic}."}
      ])
      return response.content

  # 创建用于持久化的检查点
  checkpointer = InMemorySaver()

  @entrypoint(checkpointer=checkpointer)
  def workflow(topics: list[str]) -> str:
      """并行生成多个段落并将它们组合在一起。"""
      futures = [generate_paragraph(topic) for topic in topics]
      paragraphs = [f.result() for f in futures]
      return "\n\n".join(paragraphs)

  # 运行工作流
  config = {"configurable": {"thread_id": str(uuid.uuid4())}}
  result = workflow.invoke(["quantum computing", "climate change", "history of aviation"], config=config)
  print(result)
  ```

  此示例使用LangGraph的并发模型来改进执行时间，特别是当任务涉及像LLM完成这样的I/O操作时。
</Accordion>

## 调用图

**函数式API**和[**图API**](/oss/python/langgraph/graph-api)可以在同一应用程序中一起使用，因为它们共享相同的底层运行时。

```python  theme={null}
from langgraph.func import entrypoint
from langgraph.graph import StateGraph

builder = StateGraph()
...
some_graph = builder.compile()

@entrypoint()
def some_workflow(some_input: dict) -> int:
    # 调用使用图API定义的图
    result_1 = some_graph.invoke(...)
    # 调用使用图API定义的另一个图
    result_2 = another_graph.invoke(...)
    return {
        "result_1": result_1,
        "result_2": result_2
    }
```

<Accordion title="扩展示例：从函数式API调用简单图">
  ```python  theme={null}
  import uuid
  from typing import TypedDict
  from langgraph.func import entrypoint
  from langgraph.checkpoint.memory import InMemorySaver
  from langgraph.graph import StateGraph

  # 定义共享状态类型
  class State(TypedDict):
      foo: int

  # 定义一个简单的转换节点
  def double(state: State) -> State:
      return {"foo": state["foo"] * 2}

  # 使用图API构建图
  builder = StateGraph(State)
  builder.add_node("double", double)
  builder.set_entry_point("double")
  graph = builder.compile()

  # 定义函数式API工作流
  checkpointer = InMemorySaver()

  @entrypoint(checkpointer=checkpointer)
  def workflow(x: int) -> dict:
      result = graph.invoke({"foo": x})
      return {"bar": result["foo"]}

  # 执行工作流
  config = {"configurable": {"thread_id": str(uuid.uuid4())}}
  print(workflow.invoke(5, config=config))  # 输出: {'bar': 10}
  ```
</Accordion>

## 调用其他入口点

您可以在**entrypoint**或**task**内部调用其他**entrypoint**。

```python  theme={null}
@entrypoint() # 将自动使用父entrypoint的检查点
def some_other_workflow(inputs: dict) -> int:
    return inputs["value"]

@entrypoint(checkpointer=checkpointer)
def my_workflow(inputs: dict) -> int:
    value = some_other_workflow.invoke({"value": 1})
    return value
```

<Accordion title="扩展示例：调用另一个entrypoint">
  ```python  theme={null}
  import uuid
  from langgraph.func import entrypoint
  from langgraph.checkpoint.memory import InMemorySaver

  # 初始化检查点
  checkpointer = InMemorySaver()

  # 一个可重用的子工作流，用于乘法运算
  @entrypoint()
  def multiply(inputs: dict) -> int:
      return inputs["a"] * inputs["b"]

  # 调用子工作流的主工作流
  @entrypoint(checkpointer=checkpointer)
  def main(inputs: dict) -> dict:
      result = multiply.invoke({"a": inputs["x"], "b": inputs["y"]})
      return {"product": result}

  # 执行主工作流
  config = {"configurable": {"thread_id": str(uuid.uuid4())}}
  print(main.invoke({"x": 6, "y": 7}, config=config))  # 输出: {'product': 42}
  ```
</Accordion>

## 流式传输

**函数式API**使用与**图API**相同的流式传输机制。请阅读[**流式传输指南**](/oss/python/langgraph/streaming)部分以获取更多详细信息。

使用流式传输API流式传输更新和自定义数据的示例。

```python  theme={null}
from langgraph.func import entrypoint
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.config import get_stream_writer   # [!code highlight]

checkpointer = InMemorySaver()

@entrypoint(checkpointer=checkpointer)
def main(inputs: dict) -> int:
    writer = get_stream_writer()   # [!code highlight]
    writer("Started processing")   # [!code highlight]
    result = inputs["x"] * 2
    writer(f"Result is {result}")   # [!code highlight]
    return result

config = {"configurable": {"thread_id": "abc"}}

for mode, chunk in main.stream(   # [!code highlight]
    {"x": 5},
    stream_mode=["custom", "updates"],   # [!code highlight]
    config=config
):
    print(f"{mode}: {chunk}")
```

1. 从`langgraph.config`导入[`get_stream_writer`](https://reference.langchain.com/python/langgraph/config/#langgraph.config.get_stream_writer)。
2. 在entrypoint中获取流式写入器实例。
3. 在计算开始前发出自定义数据。
4. 在计算结果后发出另一个自定义消息。
5. 使用`.stream()`处理流式输出。
6. 指定要使用的流式传输模式。

```pycon  theme={null}
('updates', {'add_one': 2})
('updates', {'add_two': 3})
('custom', 'hello')
('custom', 'world')
('updates', {'main': 5})
```

<Warning>
  **Python < 3.11时的异步**
  如果使用Python < 3.11编写异步代码，使用[`get_stream_writer`](https://reference.langchain.com/python/langgraph/config/#langgraph.config.get_stream_writer)将不起作用。请直接使用`StreamWriter`类。有关详细信息，请参见[Python < 3.11时的异步](/oss/python/langgraph/streaming#async)。

  ```python  theme={null}
  from langgraph.types import StreamWriter

  @entrypoint(checkpointer=checkpointer)
  async def main(inputs: dict, writer: StreamWriter) -> int:  # [!code highlight]
  ...
  ```
</Warning>

## 重试策略

```python  theme={null}
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.func import entrypoint, task
from langgraph.types import RetryPolicy

# 此变量仅用于演示目的以模拟网络故障。
# 它不会出现在您的实际代码中。
attempts = 0

# 让我们将RetryPolicy配置为在ValueError时重试。
# 默认的RetryPolicy针对重试特定网络错误进行了优化。
retry_policy = RetryPolicy(retry_on=ValueError)

@task(retry_policy=retry_policy)
def get_info():
    global attempts
    attempts += 1

    if attempts < 2:
        raise ValueError('Failure')
    return "OK"

checkpointer = InMemorySaver()

@entrypoint(checkpointer=checkpointer)
def main(inputs, writer):
    return get_info().result()

config = {
    "configurable": {
        "thread_id": "1"
    }
}

main.invoke({'any_input': 'foobar'}, config=config)
```

```pycon  theme={null}
'OK'
```

## 缓存任务

```python  theme={null}
import time
from langgraph.cache.memory import InMemoryCache
from langgraph.func import entrypoint, task
from langgraph.types import CachePolicy


@task(cache_policy=CachePolicy(ttl=120))    # [!code highlight]
def slow_add(x: int) -> int:
    time.sleep(1)
    return x * 2


@entrypoint(cache=InMemoryCache())
def main(inputs: dict) -> dict[str, int]:
    result1 = slow_add(inputs["x"]).result()
    result2 = slow_add(inputs["x"]).result()
    return {"result1": result1, "result2": result2}


for chunk in main.stream({"x": 5}, stream_mode="updates"):
    print(chunk)

#> {'slow_add': 10}
#> {'slow_add': 10, '__metadata__': {'cached': True}}
#> {'main': {'result1': 10, 'result2': 10}}
```

1. `ttl`以秒为单位指定。缓存在此时间后将失效。

## 错误后恢复

```python  theme={null}
import time
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.func import entrypoint, task
from langgraph.types import StreamWriter

# 此变量仅用于演示目的以模拟网络故障。
# 它不会出现在您的实际代码中。
attempts = 0

@task()
def get_info():
    """
    模拟一个第一次失败但随后成功的任务。
    第一次尝试引发异常，后续尝试返回"OK"。
    """
    global attempts
    attempts += 1

    if attempts < 2:
        raise ValueError("Failure")  # 第一次尝试模拟失败
    return "OK"

# 初始化一个内存检查点用于持久化
checkpointer = InMemorySaver()

@task
def slow_task():
    """
    通过引入1秒延迟模拟一个运行缓慢的任务。
    """
    time.sleep(1)
    return "Ran slow task."

@entrypoint(checkpointer=checkpointer)
def main(inputs, writer: StreamWriter):
    """
    主工作流函数，按顺序运行slow_task和get_info任务。
  
    参数：
    - inputs: 包含工作流输入值的字典。
    - writer: 用于流式传输自定义数据的StreamWriter。
  
    工作流首先执行`slow_task`，然后尝试执行`get_info`，
    这将在第一次调用时失败。
    """
    slow_task_result = slow_task().result()  # 对slow_task的阻塞调用
    get_info().result()  # 第一次尝试时将在此处引发异常
    return slow_task_result

# 具有唯一线程标识符的工作流执行配置
config = {
    "configurable": {
        "thread_id": "1"  # 唯一标识符，用于跟踪工作流执行
    }
}

# 此调用将因`get_info`任务失败而引发异常
try:
    # 第一次调用将因`get_info`任务失败而引发异常
    main.invoke({'any_input': 'foobar'}, config=config)
except ValueError:
    pass  # 优雅地处理失败
```

当我们恢复执行时，我们不需要重新运行`slow_task，因为其结果已保存在检查点中。

```python  theme={null}
main.invoke(None, config=config)
```

```pycon  theme={null}
'Ran slow task.'
```

## 人在循环中

函数式API使用[`interrupt`](https://reference.langchain.com/python/langgraph/types/#langgraph.types.interrupt)函数和`Command`原语支持[人在循环中](/oss/python/langgraph/interrupts)工作流。

### 基本人机交互工作流

我们将创建三个[任务](/oss/python/langgraph/functional-api#task)：

1. 附加`"bar"`。
2. 暂停以获取人工输入。恢复时，附加人工输入。
3. 附加`"qux"`。

```python  theme={null}
from langgraph.func import entrypoint, task
from langgraph.types import Command, interrupt


@task
def step_1(input_query):
    """附加bar。"""
    return f"{input_query} bar"


@task
def human_feedback(input_query):
    """附加用户输入。"""
    feedback = interrupt(f"Please provide feedback: {input_query}")
    return f"{input_query} {feedback}"


@task
def step_3(input_query):
    """附加qux。"""
    return f"{input_query} qux"
```

我们现在可以在一个[entrypoint](/oss/python/langgraph/functional-api#entrypoint)中组合这些任务：

```python  theme={null}
from langgraph.checkpoint.memory import InMemorySaver

checkpointer = InMemorySaver()


@entrypoint(checkpointer=checkpointer)
def graph(input_query):
    result_1 = step_1(input_query).result()
    result_2 = human_feedback(result_1).result()
    result_3 = step_3(result_2).result()

    return result_3
```

[interrupt()](/oss/python/langgraph/interrupts#pause-using-interrupt)在任务内部调用，使人类能够检查和编辑前一个任务的输出。先前任务的结果（在这种情况下是`step_1`）被持久化，因此在[`interrupt`](https://reference.langchain.com/python/langgraph/types/#langgraph.types.interrupt)之后它们不会再次运行。

让我们发送一个查询字符串：

```python  theme={null}
config = {"configurable": {"thread_id": "1"}}

for event in graph.stream("foo", config):
    print(event)
    print("\n")
```

注意，在`step_1`之后我们暂停了[`interrupt`](https://reference.langchain.com/python/langgraph/types/#langgraph.types.interrupt)。该中断提供了恢复运行的说明。要恢复，我们发出一个包含`human_feedback`任务期望的数据的[`Command`](/oss/python/langgraph/interrupts#resuming-interrupts)。

```python  theme={null}
# 继续执行
for event in graph.stream(Command(resume="baz"), config):
    print(event)
    print("\n")
```

恢复后，运行继续通过剩余步骤并按预期终止。

### 审查工具调用

为了在执行前审查工具调用，我们添加一个`review_tool_call`函数，该函数调用[`interrupt`](https://reference.langchain.com/python/langgraph/interrupts#pause-using-interrupt)。当此函数被调用时，执行将暂停，直到我们发出命令恢复它。

对于给定的工具调用，我们的函数将[`interrupt`](https://reference.langchain.com/python/langgraph/types/#langgraph.types.interrupt)以供人工审查。此时我们可以：

* 接受工具调用
* 修改工具调用并继续
* 生成自定义工具消息（例如，指示模型重新格式化其工具调用）

```python  theme={null}
from typing import Union

def review_tool_call(tool_call: ToolCall) -> Union[ToolCall, ToolMessage]:
    """审查工具调用，返回验证后的版本。"""
    human_review = interrupt(
        {
            "question": "Is this correct?",
            "tool_call": tool_call,
        }
    )
    review_action = human_review["action"]
    review_data = human_review.get("data")
    if review_action == "continue":
        return tool_call
    elif review_action == "update":
        updated_tool_call = {**tool_call, **{"args": review_data}}
        return updated_tool_call
    elif review_action == "feedback":
        return ToolMessage(
            content=review_data, name=tool_call["name"], tool_call_id=tool_call["id"]
        )
```

我们现在可以更新我们的[entrypoint](/oss/python/langgraph/functional-api#entrypoint)以审查生成的工具调用。如果工具调用被接受或修改，我们像以前一样执行。否则，我们只附加[`ToolMessage`](https://reference.langchain.com/python/langchain/messages/#langchain.messages.ToolMessage)由人工提供。先前任务的结果（在这种情况下是初始模型调用）被持久化，因此在[`interrupt`](https://reference.langchain.com/python/langgraph/types/#langgraph.types.interrupt)之后它们不会再次运行。

```python  theme={null}
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.graph.message import add_messages
from langgraph.types import Command, interrupt


checkpointer = InMemorySaver()


@entrypoint(checkpointer=checkpointer)
def agent(messages, previous):
    if previous is not None:
        messages = add_messages(previous, messages)

    model_response = call_model(messages).result()
    while True:
        if not model_response.tool_calls:
            break

        # 审查工具调用
        tool_results = []
        tool_calls = []
        for i, tool_call in enumerate(model_response.tool_calls):
            review = review_tool_call(tool_call)
            if isinstance(review, ToolMessage):
                tool_results.append(review)
            else:  # 是验证后的工具调用
                tool_calls.append(review)
                if review != tool_call:
                    model_response.tool_calls[i] = review  # 更新消息

        # 执行剩余的工具调用
        tool_result_futures = [call_tool(tool_call) for tool_call in tool_calls]
        remaining_tool_results = [fut.result() for fut in tool_result_futures]

        # 添加到消息列表
        messages = add_messages(
            messages,
            [model_response, *tool_results, *remaining_tool_results],
        )

        # 再次调用模型
        model_response = call_model(messages).result()

    # 生成最终响应
    messages = add_messages(messages, model_response)
    return entrypoint.final(value=model_response, save=messages)
```

## 短期记忆

短期记忆允许跨同一**thread id**的**不同调用**存储信息。有关详细信息，请参见[短期记忆](/oss/python/langgraph/functional-api#short-term-memory)。

### 管理检查点

您可以查看和删除检查器存储的信息。

<a id="checkpoint" />

#### 查看线程状态

```python  theme={null}
config = {
    "configurable": {
        "thread_id": "1",  # [!code highlight]
        # 可选地为特定检查点提供ID，
        # 否则显示最新的检查点
        # "checkpoint_id": "1f029ca3-1f5b-6704-8004-820c16b69a5a"  # [!code highlight]

    }
}
graph.get_state(config)  # [!code highlight]
```

```
StateSnapshot(
    values={'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today?), HumanMessage(content="what's my name?"), AIMessage(content='Your name is Bob.')]}, next=(),
    config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1f5b-6704-8004-820c16b69a5a'}},
    metadata={
        'source': 'loop',
        'writes': {'call_model': {'messages': AIMessage(content='Your name is Bob.')}},
        'step': 4,
        'parents': {},
        'thread_id': '1'
    },
    created_at='2025-05-05T16:01:24.680462+00:00',
    parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1790-6b0a-8003-baf965b6a38f'}},
    tasks=(),
    interrupts=()
)
```

<a id="checkpoints" />

#### 查看线程历史

```python  theme={null}
config = {
    "configurable": {
        "thread_id": "1"  # [!code highlight]
    }
}
list(graph.get_state_history(config))  # [!code highlight]
```

```
[
    StateSnapshot(
        values={'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?'), HumanMessage(content="what's my name?"), AIMessage(content='Your name is Bob.')]},
        next=(),
        config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1f5b-6704-8004-820c16b69a5a'}},
        metadata={'source': 'loop', 'writes': {'call_model': {'messages': AIMessage(content='Your name is Bob.')}}, 'step': 4, 'parents': {}, 'thread_id': '1'},
        created_at='2025-05-05T16:01:24.680462+00:00',
        parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1790-6b0a-8003-baf965b6a38f'}},
        tasks=(),
        interrupts=()
    ),
    StateSnapshot(
        values={'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?'), HumanMessage(content="what's my name?")]},
        next=('call_model',),
        config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1790-6b0a-8003-baf965b6a38f'}},
        metadata={'source': 'loop', 'writes': None, 'step': 3, 'parents': {}, 'thread_id': '1'},
        created_at='2025-05-05T16:01:23.863421+00:00',
        parent_config={...}
        tasks=(PregelTask(id='8ab4155e-6b15-b885-9ce5-bed69a2c305c', name='call_model', path=('__pregel_pull', 'call_model'), error=None, interrupts=(), state=None, result={'messages': AIMessage(content='Your name is Bob.')}),),
        interrupts=()
    ),
    StateSnapshot(
        values={'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?')]},
        next=('__start__',),
        config={...},
        metadata={'source': 'input', 'writes': {'__start__': {'messages': [{'role': 'user', 'content': "what's my name?"}]}}, 'step': 2, 'parents': {}, 'thread_id': '1'},
        created_at='2025-05-05T16:01:23.863173+00:00',
        parent_config={...}
        tasks=(PregelTask(id='24ba39d6-6db1-4c9b-f4c5-682aeaf38dcd', name='__start__', path=('__pregel_pull', '__start__'), error=None, interrupts=(), state=None, result={'messages': [{'role': 'user', 'content': "what's my name?"}]}),),
        interrupts=()
    ),
    StateSnapshot(
        values={'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?')]},
        next=(),
        config={...},
        metadata={'source': 'loop', 'writes': {'call_model': {'messages': AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?')}}, 'step': 1, 'parents': {}, 'thread_id': '1'},
        created_at='2025-05-05T16:01:23.862295+00:00',
        parent_config={...}
        tasks=(),
        interrupts=()
    ),
    StateSnapshot(
        values={'messages': [HumanMessage(content="hi! I'm bob")]},
        next=('call_model',),
        config={...},
        metadata={'source': 'loop', 'writes': None, 'step': 0, 'parents': {}, 'thread_id': '1'},
        created_at='2025-05-05T16:01:22.278960+00:00',
        parent_config={...}
        tasks=(PregelTask(id='8cbd75e0-3720-b056-04f7-71ac805140a0', name='call_model', path=('__pregel_pull', 'call_model'), error=None, interrupts=(), state=None, result={'messages': AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?')}),),
        interrupts=()
    ),
    StateSnapshot(
        values={'messages': []},
        next=('__start__',),
        config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-0870-6ce2-bfff-1f3f14c3e565'}},
        metadata={'source': 'input', 'writes': {'__start__': {'messages': [{'role': 'user', 'content': "hi! I'm bob"}]}}, 'step': -1, 'parents': {}, 'thread_id': '1'},
        created_at='2025-05-05T16:01:22.277497+00:00',
        parent_config=None,
        tasks=(PregelTask(id='d458367b-8265-812c-18e2-33001d199ce6', name='__start__', path=('__pregel_pull', '__start__'), error=None, interrupts=(), state=None, result={'messages': [{'role': 'user', 'content': "hi! I'm bob"}]}),),
        interrupts=()
    )
]
```

### 解耦返回值与保存值

使用`entrypoint.final`将对调用者的返回值与检查点中持久化的值解耦。这在以下情况下很有用：

* 您想返回计算结果（例如，摘要或状态），但保存不同的内部值以供下一次调用使用。
* 您需要控制在下一次运行时传递给previous参数的内容。

```python  theme={null}
from langgraph.func import entrypoint
from langgraph.checkpoint.memory import InMemorySaver

checkpointer = InMemorySaver()

@entrypoint(checkpointer=checkpointer)
def accumulate(n: int, *, previous: int | None) -> entrypoint.final[int, int]:
    previous = previous or 0
    total = previous + n
    # 向调用者返回*previous*值，但将*new*总计保存到检查点中。
    return entrypoint.final(value=previous, save=total)

config = {"configurable": {"thread_id": "my-thread"}}

print(accumulate.invoke(1, config=config))  # 0
print(accumulate.invoke(2, config=config))  # 1
print(accumulate.invoke(3, config=config))  # 3
```

### 聊天机器人示例

一个使用函数式API和[`InMemorySaver`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.memory.InMemorySaver)检查器的简单聊天机器人示例。

机器人能够记住先前的对话并从中断处继续。

```python  theme={null}
from langchain.messages import BaseMessage
from langgraph.graph import add_messages
from langgraph.func import entrypoint, task
from langgraph.checkpoint.memory import InMemorySaver
from langchain_anthropic import ChatAnthropic

model = ChatAnthropic(model="claude-sonnet-4-5")

@task
def call_model(messages: list[BaseMessage]):
    response = model.invoke(messages)
    return response

checkpointer = InMemorySaver()

@entrypoint(checkpointer=checkpointer)
def workflow(inputs: list[BaseMessage], *, previous: list[BaseMessage]):
    if previous:
        inputs = add_messages(previous, inputs)

    response = call_model(inputs).result()
    return entrypoint.final(value=response, save=add_messages(inputs, response))

config = {"configurable": {"thread_id": "1"}}
input_message = {"role": "user", "content": "hi! I'm bob"}
for chunk in workflow.stream([input_message], config, stream_mode="values"):
    chunk.pretty_print()

input_message = {"role": "user", "content": "what's my name?"}
for chunk in workflow.stream([input_message], config, stream_mode="values"):
    chunk.pretty_print()
```

## 长期记忆

[长期记忆](/oss/python/concepts/memory#long-term-memory)允许跨不同的**thread id**存储信息。这对于在一个对话中学习关于给定用户的信息并在另一个对话中使用它可能很有用。

## 工作流

* 有关使用函数式API构建工作流的更多示例，请参见[工作流和代理](/oss/python/langgraph/workflows-agents)指南。

## 与其他库集成

* [使用函数式API将LangGraph功能添加到其他框架](/langsmith/autogen-integration)：将LangGraph功能（如持久化、记忆和流式传输）添加到其他没有内置这些功能的代理框架中。

***

<Callout icon="pen-to-square" iconType="regular">
  [在GitHub上编辑此页面的源代码。](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/use-functional-api.mdx)
</Callout>

<Tip icon="terminal" iconType="regular">
  [通过MCP以编程方式连接这些文档](/use-these-docs)到Claude、VSCode等，以获取实时答案。
</Tip>