# 功能API概述

**功能API**允许您以对现有代码的最小改动，将LangGraph的关键功能——[持久化](/oss/python/langgraph/persistence)、[记忆](/oss/python/langgraph/add-memory)、[人在回路中](/oss/python/langgraph/interrupts)和[流式处理](/oss/python/langgraph/streaming)——添加到您的应用程序中。

它旨在将这些功能集成到现有代码中，这些代码可能使用标准的语言原语进行分支和控制流，例如`if`语句、`for`循环和函数调用。与许多需要将代码重构为显式管道或DAG的数据编排框架不同，功能API允许您在不强制执行严格的执行模型的情况下，合并这些功能。

功能API使用两个关键构建块：

* **`@entrypoint`** – 将函数标记为工作流的起点，封装逻辑并管理执行流程，包括处理长时间运行的任务和中断。
* **[`@task`](https://reference.langchain.com/python/langgraph/func/#langgraph.func.task)** – 代表一个独立的工作单元，如API调用或数据处理步骤，可以在入口点内异步执行。任务返回一个类似future的对象，可以等待或同步解析。

这为具有状态管理和流式处理的工作流构建提供了最小抽象。

<Tip>
  有关如何使用功能API的信息，请参见[使用功能API](/oss/python/langgraph/use-functional-api)。
</Tip>

## 功能API与图API对比

对于更喜欢声明式方法的用户，LangGraph的[图API](/oss/python/langgraph/graph-api)允许您使用图范式定义工作流。两个API共享相同的底层运行时，因此您可以在同一应用程序中同时使用它们。

以下是一些主要区别：

* **控制流**：功能API不需要考虑图结构。您可以使用标准的Python构造来定义工作流。这通常会减少您需要编写的代码量。
* **短期记忆**：**图API**需要声明一个[**状态**](/oss/python/langgraph/graph-api#state)，并且可能需要定义[**reducer**](/oss/python/langgraph/graph-api#reducers)来管理图状态的更新。`@entrypoint`和`@tasks`不需要显式的状态管理，因为它们的状态限定在函数范围内，不跨函数共享。
* **检查点**：两个API都生成并使用检查点。在**图API**中，每个[超级步骤](/oss/python/langgraph/graph-api)后都会生成一个新的检查点。在功能API中，当任务执行时，它们的结果被保存到与给定入口点关联的现有检查点中，而不是创建新的检查点。
* **可视化**：图API使得将工作流可视化为图形变得容易，这对于调试、理解工作流程以及与他人分享很有用。功能API不支持可视化，因为图是在运行时动态生成的。

## 示例

下面我们演示一个简单的应用程序，它写一篇论文并[中断](/oss/python/langgraph/interrupts)以请求人工审查。

```python  theme={null}
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.func import entrypoint, task
from langgraph.types import interrupt

@task
def write_essay(topic: str) -> str:
    """写一篇关于给定主题的论文。"""
    time.sleep(1) # 长时间运行任务的占位符。
    return f"关于主题的论文：{topic}"

@entrypoint(checkpointer=InMemorySaver())
def workflow(topic: str) -> dict:
    """一个写论文并请求审查的简单工作流。"""
    essay = write_essay("cat").result()
    is_approved = interrupt({
        # 提供给中断的任何可JSON序列化的有效负载。
        # 当从工作流流式传输数据时，它将在客户端显示为中断。
        "essay": essay, # 我们需要审查的论文。
        # 我们可以添加任何其他需要的信息。
        # 例如，引入一个名为"action"的键和一些指令。
        "action": "请批准/拒绝该论文",
    })

    return {
        "essay": essay, # 生成的论文
        "is_approved": is_approved, # 来自HIL的响应
    }
```

<Accordion title="详细解释">
  这个工作流将写一篇关于"猫"主题的论文，然后暂停以获取人类的审查。工作流可以被无限期地中断，直到提供审查为止。

  当工作流恢复时，它从头开始执行，但由于`writeEssay`任务的结果已经保存，任务结果将从检查点加载，而不是重新计算。

  ```python  theme={null}
  import time
  import uuid
  from langgraph.func import entrypoint, task
  from langgraph.types import interrupt
  from langgraph.checkpoint.memory import InMemorySaver


  @task
  def write_essay(topic: str) -> str:
      """写一篇关于给定主题的论文。"""
      time.sleep(1)  # 这是长时间运行任务的占位符。
      return f"关于主题的论文：{topic}"

  @entrypoint(checkpointer=InMemorySaver())
  def workflow(topic: str) -> dict:
      """一个写论文并请求审查的简单工作流。"""
      essay = write_essay("cat").result()
      is_approved = interrupt(
          {
              # 提供给中断的任何可JSON序列化的有效负载。
              # 当从工作流流式传输数据时，它将在客户端显示为中断。
              "essay": essay,  # 我们需要审查的论文。
              # 我们可以添加任何其他需要的信息。
              # 例如，引入一个名为"action"的键和一些指令。
              "action": "请批准/拒绝该论文",
          }
      )
      return {
          "essay": essay,  # 生成的论文
          "is_approved": is_approved,  # 来自HIL的响应
      }


  thread_id = str(uuid.uuid4())
  config = {"configurable": {"thread_id": thread_id}}
  for item in workflow.stream("cat", config):
      print(item)
  # > {'write_essay': '关于主题的论文：cat'}
  # > {
  # >     '__interrupt__': (
  # >        Interrupt(
  # >            value={
  # >                'essay': '关于主题的论文：cat',
  # >                'action': '请批准/拒绝该论文'
  # >            },
  # >            id='b9b2b9d788f482663ced6dc755c9e981'
  # >        ),
  # >    )
  # > }
  ```

  一篇论文已经写好，准备进行审查。一旦提供了审查，我们就可以恢复工作流：

  ```python  theme={null}
  from langgraph.types import Command

  # 从用户获取审查（例如，通过UI）
  # 在这种情况下，我们使用bool，但这可以是任何可JSON序列化的值。
  human_review = True

  for item in workflow.stream(Command(resume=human_review), config):
      print(item)
  ```

  ```pycon  theme={null}
  {'workflow': {'essay': '关于主题的论文：cat', 'is_approved': False}}
  ```

  工作流已完成，审查已添加到论文中。
</Accordion>

## 入口点

[`@entrypoint`](https://reference.langchain.com/python/langgraph/func/#langgraph.func.entrypoint)装饰器可用于从函数创建工作流。它封装工作流逻辑并管理执行流程，包括处理*长时间运行的任务*和[中断](/oss/python/langgraph/interrupts)。

### 定义

**入口点**通过使用`@entrypoint`装饰器装饰函数来定义。

函数**必须接受单个位置参数**，该参数作为工作流输入。如果您需要传递多个数据片段，请使用字典作为第一个参数的输入类型。

使用`@entrypoint`装饰函数会生成一个[`Pregel`](https://reference.langchain.com/python/langgraph/pregel/#langgraph.pregel.Pregel.stream)实例，有助于管理工作流的执行（例如，处理流式传输、恢复和检查点）。

您通常需要将**检查点器**传递给`@entrypoint`装饰器，以启用持久化并使用**人在回路中**等功能。

<Tabs>
  <Tab title="同步">
    ```python  theme={null}
    from langgraph.func import entrypoint

    @entrypoint(checkpointer=checkpointer)
    def my_workflow(some_input: dict) -> int:
        # 一些可能涉及长时间运行任务（如API调用）的逻辑，
        # 并且可能被中断以实现人在回路中。
        ...
        return result
    ```
  </Tab>

  <Tab title="异步">
    ```python  theme={null}
    from langgraph.func import entrypoint

    @entrypoint(checkpointer=checkpointer)
    async def my_workflow(some_input: dict) -> int:
        # 一些可能涉及长时间运行任务（如API调用）的逻辑，
        # 并且可能被中断以实现人在回路中
        ...
        return result
    ```
  </Tab>
</Tabs>

<Warning>
  **序列化**
  入口点的**输入**和**输出**必须是可JSON序列化的，以支持检查点。请参见[序列化](#serialization)部分了解更多详情。
</Warning>

### 可注入参数

在声明`entrypoint`时，您可以请求访问将在运行时自动注入的附加参数。这些参数包括：

| 参数 | 描述 |
| ------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **previous** | 访问与给定线程的先前`checkpoint`关联的状态。请参见[短期记忆](#short-term-memory)。 |
| **store** | [BaseStore]\[langgraph.store.base.BaseStore]的实例。用于[长期记忆](/oss/python/langgraph/use-functional-api#long-term-memory)。 |
| **writer** | 用于在使用Async Python < 3.11时访问StreamWriter。有关详细信息，请参见[使用功能API进行流式处理](/oss/python/langgraph/use-functional-api#streaming)。 |
| **config** | 用于访问运行时配置。有关信息，请参见[RunnableConfig](https://python.langchain.com/docs/concepts/runnables/#runnableconfig)。 |

<Warning>
  使用适当的名称和类型声明参数。
</Warning>

<Accordion title="请求可注入参数">
  ```python  theme={null}
  from langchain_core.runnables import RunnableConfig
  from langgraph.func import entrypoint
  from langgraph.store.base import BaseStore
  from langgraph.store.memory import InMemoryStore

  in_memory_store = InMemoryStore(...)  # InMemoryStore实例，用于长期记忆

  @entrypoint(
      checkpointer=checkpointer,  # 指定检查点器
      store=in_memory_store  # 指定存储
  )
  def my_workflow(
      some_input: dict,  # 输入（例如，通过`invoke`传递）
      *,
      previous: Any = None, # 用于短期记忆
      store: BaseStore,  # 用于长期记忆
      writer: StreamWriter,  # 用于流式传输自定义数据
      config: RunnableConfig  # 用于传递给入口点的配置
  ) -> ...:
  ```
</Accordion>

### 执行

使用[`@entrypoint`](#entrypoint)会产生一个[`Pregel`](https://reference.langchain.com/python/langgraph/pregel/#langgraph.pregel.Pregel.stream)对象，可以使用`invoke`、`ainvoke`、`stream`和`astream`方法执行。

<Tabs>
  <Tab title="调用">
    ```python  theme={null}
    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }
    my_workflow.invoke(some_input, config)  # 同步等待结果
    ```
  </Tab>

  <Tab title="异步调用">
    ```python  theme={null}
    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }
    await my_workflow.ainvoke(some_input, config)  # 异步等待结果
    ```
  </Tab>

  <Tab title="流式传输">
    ```python  theme={null}
    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    for chunk in my_workflow.stream(some_input, config):
        print(chunk)
    ```
  </Tab>

  <Tab title="异步流式传输">
    ```python  theme={null}
    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    async for chunk in my_workflow.astream(some_input, config):
        print(chunk)
    ```
  </Tab>
</Tabs>

### 恢复

在[中断](https://reference.langchain.com/python/langgraph/types/#langgraph.types.interrupt)后恢复执行可以通过将**恢复**值传递给[`Command`](https://reference.langchain.com/python/langgraph/types/#langgraph.types.Command)原语来完成。

<Tabs>
  <Tab title="调用">
    ```python  theme={null}
    from langgraph.types import Command

    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    my_workflow.invoke(Command(resume=some_resume_value), config)
    ```
  </Tab>

  <Tab title="异步调用">
    ```python  theme={null}
    from langgraph.types import Command

    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    await my_workflow.ainvoke(Command(resume=some_resume_value), config)
    ```
  </Tab>

  <Tab title="流式传输">
    ```python  theme={null}
    from langgraph.types import Command

    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    for chunk in my_workflow.stream(Command(resume=some_resume_value), config):
        print(chunk)
    ```
  </Tab>

  <Tab title="异步流式传输">
    ```python  theme={null}
    from langgraph.types import Command

    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    async for chunk in my_workflow.astream(Command(resume=some_resume_value), config):
        print(chunk)
    ```
  </Tab>
</Tabs>

**错误后恢复**

要在错误后恢复，使用`None`和相同的**线程id**（配置）运行`entrypoint`。

这假设底层的**错误**已经解决，并且可以成功执行。

<Tabs>
  <Tab title="调用">
    ```python  theme={null}

    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    my_workflow.invoke(None, config)
    ```
  </Tab>

  <Tab title="异步调用">
    ```python  theme={null}

    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    await my_workflow.ainvoke(None, config)
    ```
  </Tab>

  <Tab title="流式传输">
    ```python  theme={null}

    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    for chunk in my_workflow.stream(None, config):
        print(chunk)
    ```
  </Tab>

  <Tab title="异步流式传输">
    ```python  theme={null}

    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    async for chunk in my_workflow.astream(None, config):
        print(chunk)
    ```
  </Tab>
</Tabs>

### 短期记忆

当使用`checkpointer`定义`entrypoint`时，它会将信息存储在相同**线程id**的连续调用之间的[检查点](/oss/python/langgraph/persistence#checkpoints)中。

这允许使用`previous`参数从先前的调用访问状态。

默认情况下，`previous`参数是先前调用的返回值。

```python  theme={null}
@entrypoint(checkpointer=checkpointer)
def my_workflow(number: int, *, previous: Any = None) -> int:
    previous = previous or 0
    return number + previous

config = {
    "configurable": {
        "thread_id": "some_thread_id"
    }
}

my_workflow.invoke(1, config)  # 1（先前为None）
my_workflow.invoke(2, config)  # 3（先前为前一次调用的1）
```

#### `entrypoint.final`

[`entrypoint.final`](https://reference.langchain.com/python/langgraph/func/#langgraph.func.entrypoint.final)是一个特殊原语，可以从入口点返回，并允许将**保存在检查点中的值**与**入口点的返回值****解耦**。

第一个值是入口点的返回值，第二个值是保存在检查点中的值。类型注解为`entrypoint.final[return_type, save_type]`。

```python  theme={null}
@entrypoint(checkpointer=checkpointer)
def my_workflow(number: int, *, previous: Any = None) -> entrypoint.final[int, int]:
    previous = previous or 0
    # 这将返回先前的值给调用者，保存
    # 2 * number到检查点，它将在下一次调用中
    # 用于`previous`参数。
    return entrypoint.final(value=previous, save=2 * number)

config = {
    "configurable": {
        "thread_id": "1"
    }
}

my_workflow.invoke(3, config)  # 0（先前为None）
my_workflow.invoke(1, config)  # 6（先前为前一次调用的3 * 2）
```

## 任务

**任务**代表一个独立的工作单元，如API调用或数据处理步骤。它有两个关键特征：

* **异步执行**：任务被设计为异步执行，允许多个操作并发运行而不阻塞。
* **检查点**：任务结果被保存到检查点，从而能够从最后保存的状态恢复工作流。（更多细节请参见[持久化](/oss/python/langgraph/persistence)）。

### 定义

任务使用`@task`装饰器定义，它包装了一个常规的Python函数。

```python  theme={null}
from langgraph.func import task

@task()
def slow_computation(input_value):
    # 模拟长时间运行的操作
    ...
    return result
```

<Warning>
  **序列化**
  任务的**输出**必须是可JSON序列化的，以支持检查点。
</Warning>

### 执行

**任务**只能从**入口点**、另一个**任务**或[状态图节点](/oss/python/langgraph/graph-api#nodes)内部调用。

任务*不能*直接从主应用程序代码中调用。

当您调用一个**任务**时，它会立即返回一个future对象。future是稍后可用的结果的占位符。

要获取**任务**的结果，您可以同步等待（使用`result()`）或异步等待（使用`await`）。

<Tabs>
  <Tab title="同步调用">
    ```python  theme={null}
    @entrypoint(checkpointer=checkpointer)
    def my_workflow(some_input: int) -> int:
        future = slow_computation(some_input)
        return future.result()  # 同步等待结果
    ```
  </Tab>

  <Tab title="异步调用">
    ```python  theme={null}
    @entrypoint(checkpointer=checkpointer)
    async def my_workflow(some_input: int) -> int:
        return await slow_computation(some_input)  # 异步等待结果
    ```
  </Tab>
</Tabs>

## 何时使用任务

**任务**在以下场景中很有用：

* **检查点**：当您需要将长时间运行的操作结果保存到检查点时，这样在恢复工作流时不需要重新计算它。
* **人在回路中**：如果您正在构建需要人工干预的工作流，您必须使用**任务**来封装任何随机性（例如API调用），以确保工作流可以正确恢复。有关详细信息，请参见[确定性](#determinism)部分。
* **并行执行**：对于I/O密集型任务，**任务**启用并行执行，允许多个操作并发运行而不阻塞（例如，调用多个API）。
* **可观测性**：将操作包装在**任务**中，提供了一种跟踪工作流进度并使用[LangSmith](https://docs.smith.langchain.com/)监控单个操作执行的方法。
* **可重试工作**：当工作需要重试以处理故障或不一致性时，**任务**提供了一种封装和管理重试逻辑的方法。

## 序列化

LangGraph中序列化有两个关键方面：

1. `entrypoint`的输入和输出必须是可JSON序列化的。
2. `task`的输出必须是可JSON序列化的。

这些要求对于启用检查点和工作流恢复是必要的。使用Python基本类型如字典、列表、字符串、数字和布尔值来确保您的输入和输出是可序列化的。

序列化确保工作流状态（如任务结果和中间值）可以可靠地保存和恢复。这对于启用人在回路中的交互、容错和并行执行至关重要。

当工作流配置了检查点器时，提供不可序列化的输入或输出将导致运行时错误。

## 确定性

要利用**人在回路中**等功能，任何随机性都应该封装在**任务**内部。这确保了当执行暂停（例如人在回路中）然后恢复时，它将遵循相同的*步骤序列*，即使**任务**结果是非确定性的。

LangGraph通过在执行时持久化**任务**和[**子图**](/oss/python/langgraph/use-subgraphs)结果来实现这种行为。设计良好的工作流确保恢复执行遵循相同的*记录步骤序列*，允许正确检索先前计算的结果而无需重新执行它们。这对于长时间运行的**任务**或具有非确定性结果的**任务**特别有用，因为它避免了重复先前的工作，并允许从本质上相同的状态恢复。

虽然工作流的不同运行可以产生不同的结果，但恢复**特定**的运行应始终遵循相同的记录步骤序列。这允许LangGraph高效地查找在图被中断之前执行的**任务**和**子图**结果，而无需重新计算它们。

## 幂等性

幂等性确保多次运行相同的操作产生相同的结果。如果由于故障导致步骤重新运行，这有助于防止重复的API调用和冗余处理。始终将API调用放在**任务**函数中以进行检查点，并设计它们以在重新执行时是幂等的。重新执行可能发生在**任务**开始但未成功完成的情况下。然后，如果工作流恢复，**任务**将再次运行。使用幂等键或验证现有结果以避免重复。

## 常见陷阱

### 处理副作用

将副作用（例如写入文件、发送电子邮件）封装在任务中，以确保在恢复工作流时不会多次执行它们。

<Tabs>
  <Tab title="不正确">
    在此示例中，副作用（写入文件）直接包含在工作流中，因此在恢复工作流时它将第二次执行。

    ```python  theme={null}
    @entrypoint(checkpointer=checkpointer)
    def my_workflow(inputs: dict) -> int:
        # 当恢复工作流时，此代码将第二次执行。
        # 这可能不是您想要的。
        with open("output.txt", "w") as f:  # [!code highlight]
            f.write("副作用已执行")  # [!code highlight]
        value = interrupt("question")
        return value
    ```
  </Tab>

  <Tab title="正确">
    在此示例中，副作用被封装在任务中，确保在恢复时执行一致。

    ```python  theme={null}
    from langgraph.func import task

    @task  # [!code highlight]
    def write_to_file():  # [!code highlight]
        with open("output.txt", "w") as f:
            f.write("副作用已执行")

    @entrypoint(checkpointer=checkpointer)
    def my_workflow(inputs: dict) -> int:
        # 副作用现在封装在任务中。
        write_to_file().result()
        value = interrupt("question")
        return value
    ```
  </Tab>
</Tabs>

### 非确定性控制流

每次可能给出不同结果的操作（如获取当前时间或随机数）应该封装在任务中，以确保在恢复时返回相同的结果。

* 在任务中：获取随机数(5) → 中断 → 恢复 → (再次返回5) → ...
* 不在任务中：获取随机数(5) → 中断 → 恢复 → 获取新随机数(7) → ...

当使用多个中断调用构建**人在回路中**工作流时，这尤为重要。LangGraph为每个任务/入口点保留一个恢复值列表。当遇到中断时，它将与相应的恢复值匹配。这种匹配严格基于**索引**，因此恢复值的顺序应与中断的顺序匹配。

如果恢复时没有保持执行顺序，一个[`interrupt`](https://reference.langchain.com/python/langgraph/types/#langgraph.types.interrupt)调用可能与错误的`resume`值匹配，导致不正确的结果。

有关详细信息，请阅读[确定性](#determinism)部分。

<Tabs>
  <Tab title="不正确">
    在此示例中，工作流使用当前时间来确定要执行哪个任务。这是非确定性的，因为工作流的结果取决于执行它的时间。

    ```python  theme={null}
    from langgraph.func import entrypoint

    @entrypoint(checkpointer=checkpointer)
    def my_workflow(inputs: dict) -> int:
        t0 = inputs["t0"]
        t1 = time.time()  # [!code highlight]

        delta_t = t1 - t0

        if delta_t > 1:
            result = slow_task(1).result()
            value = interrupt("question")
        else:
            result = slow_task(2).result()
            value = interrupt("question")

        return {
            "result": result,
            "value": value
        }
    ```
  </Tab>

  <Tab title="正确">
    在此示例中，工作流使用输入`t0`来确定要执行哪个任务。这是确定性的，因为工作流的结果仅取决于输入。

    ```python  theme={null}
    import time

    from langgraph.func import task

    @task  # [!code highlight]
    def get_time() -> float:  # [!code highlight]
        return time.time()

    @entrypoint(checkpointer=checkpointer)
    def my_workflow(inputs: dict) -> int:
        t0 = inputs["t0"]
        t1 = get_time().result()  # [!code highlight]

        delta_t = t1 - t0

        if delta_t > 1:
            result = slow_task(1).result()
            value = interrupt("question")
        else:
            result = slow_task(2).result()
            value = interrupt("question")

        return {
            "result": result,
            "value": value
        }
    ```
  </Tab>
</Tabs>

***

<Callout icon="pen-to-square" iconType="regular">
  [在GitHub上编辑此页面的源代码。](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/functional-api.mdx)
</Callout>

<Tip icon="terminal" iconType="regular">
  [通过MCP以编程方式连接这些文档](/use-these-docs)到Claude、VSCode等，以获得实时答案。
</Tip>