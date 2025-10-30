# 中断 (Interrupts)

中断允许您在特定点暂停图执行，并在继续之前等待外部输入。这支持了需要外部输入才能继续进行的人机交互模式。当触发中断时，LangGraph使用其[持久化](/oss/python/langgraph/persistence)层保存图状态，并无限期等待，直到您恢复执行。

中断通过在图节点中的任何位置调用`interrupt()`函数来工作。该函数接受任何可JSON序列化的值，该值将呈现给调用者。当您准备好继续时，通过使用`Command`重新调用图来恢复执行，该命令随后成为节点内部`interrupt()`调用的返回值。

与静态断点（在特定节点之前或之后暂停）不同，中断是**动态的**——它们可以放在代码中的任何位置，并且可以根据应用程序逻辑进行条件设置。

* **检查点保持您的位置**：检查点会写入确切的图状态，以便您可以稍后恢复，即使在错误状态中也是如此。
* **`thread_id`是您的指针**：设置`config={"configurable": {"thread_id": ...}}`来告诉检查点要加载哪个状态。
* **中断负载作为`__interrupt__`呈现**：您传递给`interrupt()`的值在`__interrupt__`字段中返回给调用者，以便您知道图在等待什么。

您选择的`thread_id`实际上是您的持久游标。重用它会恢复相同的检查点；使用新值会启动一个具有空状态的全新线程。

## 使用`interrupt`暂停

[`interrupt`](https://reference.langchain.com/python/langgraph/types/#langgraph.types.interrupt)函数暂停图执行并向调用者返回一个值。当您在节点内调用[`interrupt`](https://reference.langchain.com/python/langgraph/types/#langgraph.types.interrupt)时，LangGraph保存当前图状态并等待您使用输入恢复执行。

要使用[`interrupt`](https://reference.langchain.com/python/langgraph/types/#langgraph.types.interrupt)，您需要：

1. 一个**检查点**来持久化图状态（在生产环境中使用持久检查点）
2. 配置中的**线程ID**，以便运行时知道从哪个状态恢复
3. 在要暂停的位置调用`interrupt()`（负载必须是可JSON序列化的）

```python  theme={null}
from langgraph.types import interrupt

def approval_node(state: State):
    # 暂停并请求批准
    approved = interrupt("您是否批准此操作？")

    # 当您恢复时，Command(resume=...)会在这里返回该值
    return {"approved": approved}
```

当您调用[`interrupt`](https://reference.langchain.com/python/langgraph/types/#langgraph.types.interrupt)时，会发生以下情况：

1. **图执行被挂起**在调用[`interrupt`](https://reference.langchain.com/python/langgraph/types/#langgraph.types.interrupt)的确切位置
2. **状态被保存**使用检查点，以便稍后可以恢复执行，在生产环境中，这应该是一个持久检查点（例如由数据库支持）
3. **值被返回**给调用者的`__interrupt__`字段；它可以是任何可JSON序列化的值（字符串、对象、数组等）
4. **图无限期等待**，直到您使用响应恢复执行
5. **响应被传回**节点内，成为`interrupt()`调用的返回值

## 恢复中断

在中断暂停执行后，您通过使用包含恢复值的`Command`再次调用图来恢复它。恢复值被传回给`interrupt`调用，允许节点使用外部输入继续执行。

```python  theme={null}
from langgraph.types import Command

# 初始运行 - 触发中断并暂停
# thread_id是持久指针（在生产环境中存储稳定的ID）
config = {"configurable": {"thread_id": "thread-1"}}
result = graph.invoke({"input": "data"}, config=config)

# 检查中断了什么
# __interrupt__包含传递给interrupt()的负载
print(result["__interrupt__"])
# > [Interrupt(value='您是否批准此操作？')]

# 使用人类的响应恢复
# 恢复负载成为节点内interrupt()的返回值
graph.invoke(Command(resume=True), config=config)
```

**关于恢复的关键点：**

* 恢复时必须使用与中断发生时**相同的线程ID**
* 传递给`Command(resume=...)`的值成为[`interrupt`](https://reference.langchain.com/python/langgraph/types/#langgraph.types.interrupt)调用的返回值
* 恢复时，节点从调用[`interrupt`](https://reference.langchain.com/python/langgraph/types/#langgraph.types.interrupt)的节点的开头重新启动，因此[`interrupt`](https://reference.langchain.com/python/langgraph/types/#langgraph.types.interrupt)之前的任何代码都会再次运行
* 您可以传递任何可JSON序列化的值作为恢复值

## 常见模式

中断解锁的关键功能是暂停执行并等待外部输入的能力。这对于各种用例非常有用，包括：

* <Icon icon="check-circle" /> [批准工作流](#批准或拒绝)：在执行关键操作（API调用、数据库更改、金融交易）之前暂停
* <Icon icon="pencil" /> [审查和编辑](#审查和编辑状态)：让人类在继续之前审查和修改LLM输出或工具调用
* <Icon icon="wrench" /> [中断工具调用](#工具中的中断)：在执行工具调用之前暂停，以在执行前审查和编辑工具调用
* <Icon icon="shield-check" /> [验证人类输入](#验证人类输入)：在继续下一步之前暂停，以验证人类输入

### 批准或拒绝

中断最常见的用途之一是在关键操作之前暂停并请求批准。例如，您可能希望要求人类批准API调用、数据库更改或任何其他重要决策。

```python  theme={null}
from typing import Literal
from langgraph.types import interrupt, Command

def approval_node(state: State) -> Command[Literal["proceed", "cancel"]]:
    # 暂停执行；负载显示在result["__interrupt__"]中
    is_approved = interrupt({
        "question": "您是否要继续此操作？",
        "details": state["action_details"]
    })

    # 根据响应进行路由
    if is_approved:
        return Command(goto="proceed")  # 在提供恢复负载后运行
    else:
        return Command(goto="cancel")
```

当您恢复图时，传递`true`表示批准，`false`表示拒绝：

```python  theme={null}
# 批准
graph.invoke(Command(resume=True), config=config)

# 拒绝
graph.invoke(Command(resume=False), config=config)
```

<Accordion title="完整示例">
  ```python  theme={null}
  import sqlite3
  from typing import Literal, Optional, TypedDict

  from langgraph.checkpoint.memory import MemorySaver
  from langgraph.graph import StateGraph, START, END
  from langgraph.types import Command, interrupt


  class ApprovalState(TypedDict):
      action_details: str
      status: Optional[Literal["pending", "approved", "rejected"]]


  def approval_node(state: ApprovalState) -> Command[Literal["proceed", "cancel"]]:
      # 公开详细信息，以便调用者可以在UI中呈现它们
      decision = interrupt({
          "question": "批准此操作？",
          "details": state["action_details"],
      })

      # 恢复后路由到适当的节点
      return Command(goto="proceed" if decision else "cancel")


  def proceed_node(state: ApprovalState):
      return {"status": "approved"}


  def cancel_node(state: ApprovalState):
      return {"status": "rejected"}


  builder = StateGraph(ApprovalState)
  builder.add_node("approval", approval_node)
  builder.add_node("proceed", proceed_node)
  builder.add_node("cancel", cancel_node)
  builder.add_edge(START, "approval")
  builder.add_edge("approval", "proceed")
  builder.add_edge("approval", "cancel")
  builder.add_edge("proceed", END)
  builder.add_edge("cancel", END)

  # 在生产环境中使用更持久的检查点
  checkpointer = MemorySaver()
  graph = builder.compile(checkpointer=checkpointer)

  config = {"configurable": {"thread_id": "approval-123"}}
  initial = graph.invoke(
      {"action_details": "转账$500", "status": "pending"},
      config=config,
  )
  print(initial["__interrupt__"])  # -> [Interrupt(value={'question': ..., 'details': ...})]

  # 使用决策恢复；True路由到proceed，False路由到cancel
  resumed = graph.invoke(Command(resume=True), config=config)
  print(resumed["status"])  # -> "approved"
  ```
</Accordion>

### 审查和编辑状态

有时您希望让人类在继续之前审查和编辑图状态的一部分。这对于纠正LLM、添加缺失信息或进行调整很有用。

```python  theme={null}
from langgraph.types import interrupt

def review_node(state: State):
    # 暂停并显示当前内容以供审查（在result["__interrupt__"]中显示）
    edited_content = interrupt({
        "instruction": "审查并编辑此内容",
        "content": state["generated_text"]
    })

    # 使用编辑版本更新状态
    return {"generated_text": edited_content}
```

恢复时，提供编辑后的内容：

```python  theme={null}
graph.invoke(
    Command(resume="经过编辑和改进的文本"),  # 值成为interrupt()的返回值
    config=config
)
```

<Accordion title="完整示例">
  ```python  theme={null}
  import sqlite3
  from typing import TypedDict

  from langgraph.checkpoint.memory import MemorySaver
  from langgraph.graph import StateGraph, START, END
  from langgraph.types import Command, interrupt


  class ReviewState(TypedDict):
      generated_text: str


  def review_node(state: ReviewState):
      # 请求审查者编辑生成的内容
      updated = interrupt({
          "instruction": "审查并编辑此内容",
          "content": state["generated_text"],
      })
      return {"generated_text": updated}


  builder = StateGraph(ReviewState)
  builder.add_node("review", review_node)
  builder.add_edge(START, "review")
  builder.add_edge("review", END)

  checkpointer = MemorySaver()
  graph = builder.compile(checkpointer=checkpointer)

  config = {"configurable": {"thread_id": "review-42"}}
  initial = graph.invoke({"generated_text": "初始草稿"}, config=config)
  print(initial["__interrupt__"])  # -> [Interrupt(value={'instruction': ..., 'content': ...})]

  # 使用审查者编辑的文本恢复
  final_state = graph.invoke(
      Command(resume="经过审查后改进的草稿"),
      config=config,
  )
  print(final_state["generated_text"])  # -> "经过审查后改进的草稿"
  ```
</Accordion>

### 工具中的中断

您还可以将中断直接放在工具函数中。这使得工具本身在调用时暂停等待批准，并允许在执行工具调用之前进行人工审查和编辑。

首先，定义一个使用[`interrupt`](https://reference.langchain.com/python/langgraph/types/#langgraph.types.interrupt)的工具：

```python  theme={null}
from langchain.tools import tool
from langgraph.types import interrupt

@tool
def send_email(to: str, subject: str, body: str):
    """向收件人发送电子邮件。"""

    # 发送前暂停；负载在result["__interrupt__"]中显示
    response = interrupt({
        "action": "send_email",
        "to": to,
        "subject": subject,
        "body": body,
        "message": "批准发送此电子邮件？"
    })

    if response.get("action") == "approve":
        # 恢复值可以在执行前覆盖输入
        final_to = response.get("to", to)
        final_subject = response.get("subject", subject)
        final_body = response.get("body", body)
        return f"电子邮件已发送至{final_to}，主题为'{final_subject}'"
    return "用户取消了电子邮件"
```

当您希望批准逻辑与工具本身一起存在时，这种方法很有用，使它在图的不同部分可重用。LLM可以自然地调用工具，并且每当调用工具时，中断都会暂停执行，允许您批准、编辑或取消操作。

<Accordion title="完整示例">
  ```python  theme={null}
  import sqlite3
  from typing import TypedDict

  from langchain.tools import tool
  from langchain_anthropic import ChatAnthropic
  from langgraph.checkpoint.sqlite import SqliteSaver
  from langgraph.graph import StateGraph, START, END
  from langgraph.types import Command, interrupt


  class AgentState(TypedDict):
      messages: list[dict]


  @tool
  def send_email(to: str, subject: str, body: str):
      """向收件人发送电子邮件。"""

      # 发送前暂停；负载在result["__interrupt__"]中显示
      response = interrupt({
          "action": "send_email",
          "to": to,
          "subject": subject,
          "body": body,
          "message": "批准发送此电子邮件？",
      })

      if response.get("action") == "approve":
          final_to = response.get("to", to)
          final_subject = response.get("subject", subject)
          final_body = response.get("body", body)

          # 实际发送电子邮件（您的实现在这里）
          print(f"[send_email] to={final_to} subject={final_subject} body={final_body}")
          return f"电子邮件已发送至{final_to}"

      return "用户取消了电子邮件"


  model = ChatAnthropic(model="claude-sonnet-4-5").bind_tools([send_email])


  def agent_node(state: AgentState):
      # LLM可能决定调用工具；中断在发送前暂停
      result = model.invoke(state["messages"])
      return {"messages": state["messages"] + [result]}


  builder = StateGraph(AgentState)
  builder.add_node("agent", agent_node)
  builder.add_edge(START, "agent")
  builder.add_edge("agent", END)

  checkpointer = SqliteSaver(sqlite3.connect("tool-approval.db"))
  graph = builder.compile(checkpointer=checkpointer)

  config = {"configurable": {"thread_id": "email-workflow"}}
  initial = graph.invoke(
      {
          "messages": [
              {"role": "user", "content": "发送一封关于会议的电子邮件给alice@example.com"}
          ]
      },
      config=config,
  )
  print(initial["__interrupt__"])  # -> [Interrupt(value={'action': 'send_email', ...})]

  # 使用批准和可选编辑的参数恢复
  resumed = graph.invoke(
      Command(resume={"action": "approve", "subject": "更新的主题"}),
      config=config,
  )
  print(resumed["messages"][-1])  # -> send_email返回的工具结果
  ```
</Accordion>

### 验证人类输入

有时您需要验证来自人类的输入，如果无效则再次询问。您可以使用循环中的多个[`interrupt`](https://reference.langchain.com/python/langgraph/types/#langgraph.types.interrupt)调用来做到这一点。

```python  theme={null}
from langgraph.types import interrupt

def get_age_node(state: State):
    prompt = "您的年龄是多少？"

    while True:
        answer = interrupt(prompt)  # 负载在result["__interrupt__"]中显示

        # 验证输入
        if isinstance(answer, int) and answer > 0:
            # 有效输入 - 继续
            break
        else:
            # 无效输入 - 使用更具体的提示再次询问
            prompt = f"'{answer}'不是有效的年龄。请输入一个正数。"

    return {"age": answer}
```

每次您使用无效输入恢复图时，它会以更清晰的消息再次询问。一旦提供了有效输入，节点完成并继续图执行。

<Accordion title="完整示例">
  ```python  theme={null}
  import sqlite3
  from typing import TypedDict

  from langgraph.checkpoint.sqlite import SqliteSaver
  from langgraph.graph import StateGraph, START, END
  from langgraph.types import Command, interrupt


  class FormState(TypedDict):
      age: int | None


  def get_age_node(state: FormState):
      prompt = "您的年龄是多少？"

      while True:
          answer = interrupt(prompt)  # 负载在result["__interrupt__"]中显示

          if isinstance(answer, int) and answer > 0:
              return {"age": answer}

          prompt = f"'{answer}'不是有效的年龄。请输入一个正数。"


  builder = StateGraph(FormState)
  builder.add_node("collect_age", get_age_node)
  builder.add_edge(START, "collect_age")
  builder.add_edge("collect_age", END)

  checkpointer = SqliteSaver(sqlite3.connect("forms.db"))
  graph = builder.compile(checkpointer=checkpointer)

  config = {"configurable": {"thread_id": "form-1"}}
  first = graph.invoke({"age": None}, config=config)
  print(first["__interrupt__"])  # -> [Interrupt(value='您的年龄是多少？', ...)]

  # 提供无效数据；节点会重新提示
  retry = graph.invoke(Command(resume="thirty"), config=config)
  print(retry["__interrupt__"])  # -> [Interrupt(value="'thirty'不是有效的年龄...", ...)]

  # 提供有效数据；循环退出并更新状态
  final = graph.invoke(Command(resume=30), config=config)
  print(final["age"])  # -> 30
  ```
</Accordion>

## 中断的规则

当您在节点内调用[`interrupt`](https://reference.langchain.com/python/langgraph/types/#langgraph.types.interrupt)时，LangGraph通过引发一个特殊异常来暂停执行，该异常通知运行时暂停。此异常通过调用栈向上传播并被运行时捕获，运行时通知图保存当前状态并等待外部输入。

当执行恢复（在您提供所请求的输入之后），运行时从节点的开头重新启动整个节点——它不会从调用[`interrupt`](https://reference.langchain.com/python/langgraph/types/#langgraph.types.interrupt)的确切行恢复。这意味着在[`interrupt`](https://reference.langchain.com/python/langgraph/types/#langgraph.types.interrupt)之前运行的任何代码都将再次执行。因此，在使用中断时有一些重要的规则需要遵循，以确保它们按预期运行。

### 不要将`interrupt`调用包装在try/except中

[`interrupt`](https://reference.langchain.com/python/langgraph/types/#langgraph.types.interrupt)在调用点暂停执行的方式是抛出一个特殊异常。如果您将[`interrupt`](https://reference.langchain.com/python/langgraph/types/#langgraph.types.interrupt)调用包装在try/except块中，您将捕获此异常，中断将不会传递回图。

* ✅ 将[`interrupt`](https://reference.langchain.com/python/langgraph/types/#langgraph.types.interrupt)调用与容易出错的代码分开
* ✅ 在try/except块中使用特定的异常类型

<CodeGroup>
  ```python 分离逻辑 theme={null}
  def node_a(state: State):
      # ✅ 好：先中断，然后单独处理错误情况
      interrupt("您的名字是什么？")
      try:
          fetch_data()  # 这可能会失败
      except Exception as e:
          print(e)
      return state
  ```

  ```python 显式异常处理 theme={null}
  def node_a(state: State):
      # ✅ 好：捕获特定的异常类型不会捕获中断异常
      try:
          name = interrupt("您的名字是什么？")
          fetch_data()  # 这可能会失败
      except NetworkException as e:
          print(e)
      return state
  ```
</CodeGroup>

* 🔴 不要将[`interrupt`](https://reference.langchain.com/python/langgraph/types/#langgraph.types.interrupt)调用包装在裸try/except块中

```python  theme={null}
def node_a(state: State):
    # ❌ 不好：将中断包装在裸try/except中会捕获中断异常
    try:
        interrupt("您的名字是什么？")
    except Exception as e:
        print(e)
    return state
```

### 不要在节点内重新排序`interrupt`调用

在单个节点中使用多个中断是常见的，但是如果处理不当，这可能导致意外行为。

当一个节点包含多个中断调用时，LangGraph会保持一个特定于执行节点的任务的恢复值列表。每当执行恢复时，它从节点的开头开始。对于遇到的每个中断，LangGraph检查任务的恢复列表中是否存在匹配的值。匹配是**严格基于索引的**，因此节点内中断调用的顺序很重要。

* ✅ 在节点执行过程中保持[`interrupt`](https://reference.langchain.com/python/langgraph/types/#langgraph.types.interrupt)调用的一致性

```python  theme={null}
def node_a(state: State):
    # ✅ 好：中断调用每次都以相同的顺序发生
    name = interrupt("您的名字是什么？")
    age = interrupt("您的年龄是多少？")
    city = interrupt("您的城市是什么？")

    return {
        "name": name,
        "age": age,
        "city": city
    }
```

* 🔴 不要在节点内有条件地跳过[`interrupt`](https://reference.langchain.com/python/langgraph/types/#langgraph.types.interrupt)调用
* 🔴 不要使用在执行之间不是确定性的逻辑来循环[`interrupt`](https://reference.langchain.com/python/langgraph/types/#langgraph.types.interrupt)调用

<CodeGroup>
  ```python 跳过中断 theme={null}
  def node_a(state: State):
      # ❌ 不好：有条件地跳过中断会改变顺序
      name = interrupt("您的名字是什么？")

      # 在第一次运行时，这可能会跳过中断
      # 在恢复时，它可能不会跳过 - 导致索引不匹配
      if state.get("needs_age"):
          age = interrupt("您的年龄是多少？")

      city = interrupt("您的城市是什么？")

      return {"name": name, "city": city}
  ```

  ```python 循环中断 theme={null}
  def node_a(state: State):
      # ❌ 不好：基于非确定性数据进行循环
      # 中断数量在执行之间会变化
      results = []
      for item in state.get("dynamic_list", []):  # 列表可能在运行之间更改
          result = interrupt(f"批准{item}？")
          results.append(result)

      return {"results": results}
  ```
</CodeGroup>

### 不要在`interrupt`调用中返回复杂值

根据使用的检查点类型，复杂值可能不可序列化（例如，您不能序列化函数）。为了使您的图适应任何部署，最佳实践是只使用可以合理序列化的值。

* ✅ 将简单的、可JSON序列化的类型传递给[`interrupt`](https://reference.langchain.com/python/langgraph/types/#langgraph.types.interrupt)
* ✅ 传递带有简单值的字典/对象

<CodeGroup>
  ```python 简单值 theme={null}
  def node_a(state: State):
      # ✅ 好：传递可序列化的简单类型
      name = interrupt("您的名字是什么？")
      count = interrupt(42)
      approved = interrupt(True)

      return {"name": name, "count": count, "approved": approved}
  ```

  ```python 结构化数据 theme={null}
  def node_a(state: State):
      # ✅ 好：传递带有简单值的字典
      response = interrupt({
          "question": "输入用户详细信息",
          "fields": ["name", "email", "age"],
          "current_values": state.get("user", {})
      })

      return {"user": response}
  ```
</CodeGroup>

* 🔴 不要将函数、类实例或其他复杂对象传递给[`interrupt`](https://reference.langchain.com/python/langgraph/types/#langgraph.types.interrupt)

<CodeGroup>
  ```python 函数 theme={null}
  def validate_input(value):
      return len(value) > 0

  def node_a(state: State):
      # ❌ 不好：将函数传递给中断
      # 该函数无法被序列化
      response = interrupt({
          "question": "您的名字是什么？",
          "validator": validate_input  # 这将失败
      })
      return {"name": response}
  ```

  ```python 类实例 theme={null}
  class DataProcessor:
      def __init__(self, config):
          self.config = config

  def node_a(state: State):
      processor = DataProcessor({"mode": "strict"})

      # ❌ 不好：将类实例传递给中断
      # 该实例无法被序列化
      response = interrupt({
          "question": "输入要处理的数据",
          "processor": processor  # 这将失败
      })
      return {"result": response}
  ```
</CodeGroup>

### 在`interrupt`之前调用的副作用必须是幂等的

因为中断通过重新运行它们被调用的节点来工作，所以在[`interrupt`](https://reference.langchain.com/python/langgraph/types/#langgraph.types.interrupt)之前调用的副作用应该（理想情况下）是幂等的。就上下文而言，幂等性意味着相同的操作可以应用多次，而不会改变初始执行之外的成果。

例如，您可能有一个API调用来更新节点内的记录。如果在调用之后调用[`interrupt`](https://reference.langchain.com/python/langgraph/types/#langgraph.types.interrupt)，当节点恢复时它将被多次运行，可能会覆盖初始更新或创建重复记录。

* ✅ 在[`interrupt`](https://reference.langchain.com/python/langgraph/types/#langgraph.types.interrupt)之前使用幂等操作
* ✅ 将副作用放在[`interrupt`](https://reference.langchain.com/python/langgraph/types/#langgraph.types.interrupt)调用之后
* ✅ 尽可能将副作用分离到单独的节点中

<CodeGroup>
  ```python 幂等操作 theme={null}
  def node_a(state: State):
      # ✅ 好：使用upsert操作，这是幂等的
      # 多次运行此操作将产生相同的结果
      db.upsert_user(
          user_id=state["user_id"],
          status="pending_approval"
      )

      approved = interrupt("批准此更改？")

      return {"approved": approved}
  ```

  ```python 中断后的副作用 theme={null}
  def node_a(state: State):
      # ✅ 好：将副作用放在中断之后
      # 这确保它只在收到批准后运行一次
      approved = interrupt("批准此更改？")

      if approved:
          db.create_audit_log(
              user_id=state["user_id"],
              action="approved"
          )

      return {"approved": approved}
  ```

  ```python 分离到不同节点 theme={null}
  def approval_node(state: State):
      # ✅ 好：仅在此节点中处理中断
      approved = interrupt("批准此更改？")

      return {"approved": approved}

  def notification_node(state: State):
      # ✅ 好：副作用发生在单独的节点中
      # 这在批准后运行，所以它只执行一次
      if (state.approved):
          send_notification(
              user_id=state["user_id"],
              status="approved"
          )

      return state
  ```
</CodeGroup>

* 🔴 不要在[`interrupt`](https://reference.langchain.com/python/langgraph/types/#langgraph.types.interrupt)之前执行非幂等操作
* 🔴 不要在不检查它们是否存在的情况下创建新记录

<CodeGroup>
  ```python 创建记录 theme={null}
  def node_a(state: State):
      # ❌ 不好：在中断之前创建新记录
      # 这会在每次恢复时创建重复记录
      audit_id = db.create_audit_log({
          "user_id": state["user_id"],
          "action": "pending_approval",
          "timestamp": datetime.now()
      })

      approved = interrupt("批准此更改？")

      return {"approved": approved, "audit_id": audit_id}
  ```

  ```python 追加到列表 theme={null}
  def node_a(state: State):
      # ❌ 不好：在中断之前追加到列表
      # 这会在每次恢复时添加重复条目
      db.append_to_history(state["user_id"], "approval_requested")

      approved = interrupt("批准此更改？")

      return {"approved": approved}
  ```
</CodeGroup>

## 在作为函数调用的子图中使用

在节点内调用子图时，父图将从调用子图和触发[`interrupt`](https://reference.langchain.com/python/langgraph/types/#langgraph.types.interrupt)的**节点的开头**恢复执行。类似地，**子图**也将从调用[`interrupt`](https://reference.langchain.com/python/langgraph/types/#langgraph.types.interrupt)的节点的开头恢复。

```python  theme={null}
def node_in_parent_graph(state: State):
    some_code()  # <-- 恢复时这将重新执行
    # 作为函数调用子图
    # 子图包含`interrupt`调用
    subgraph_result = subgraph.invoke(some_input)

async function node_in_subgraph(state: State) {
    someOtherCode(); # <-- 恢复时这也将重新执行
    result = interrupt("您的名字是什么？")
    ...
}
```

## 使用中断进行调试

要调试和测试图，您可以使用静态中断作为断点，一次一个节点地逐步执行图执行。静态中断在定义的点触发，可以在节点执行之前或之后。您可以通过在编译图时指定`interrupt_before`和`interrupt_after`来设置这些。

<Note>
  不建议将静态中断用于人机交互工作流。请改用[`interrupt`](https://reference.langchain.com/python/langgraph/types/#langgraph.types.interrupt)方法。
</Note>

<Tabs>
  <Tab title="编译时">
    ```python  theme={null}
    graph = builder.compile(
        interrupt_before=["node_a"],  # [!code highlight]
        interrupt_after=["node_b", "node_c"],  # [!code highlight]
        checkpointer=checkpointer,
    )

    # 将线程ID传递给图
    config = {
        "configurable": {
            "thread_id": "some_thread"
        }
    }

    # 运行图直到断点
    graph.invoke(inputs, config=config)  # [!code highlight]

    # 恢复图
    graph.invoke(None, config=config)  # [!code highlight]
    ```

    1. 断点在`compile`时设置。
    2. `interrupt_before`指定应在执行节点之前暂停执行的节点。
    3. `interrupt_after`指定应在执行节点之后暂停执行的节点。
    4. 需要检查点才能启用断点。
    5. 图运行直到遇到第一个断点。
    6. 通过为输入传递`None`来恢复图。这将运行图直到遇到下一个断点。
  </Tab>

  <Tab title="运行时">
    ```python  theme={null}
    config = {
        "configurable": {
            "thread_id": "some_thread"
        }
    }

    # 运行图直到断点
    graph.invoke(
        inputs,
        interrupt_before=["node_a"],  # [!code highlight]
        interrupt_after=["node_b", "node_c"],  # [!code highlight]
        config=config,
    )

    # 恢复图
    graph.invoke(None, config=config)  # [!code highlight]
    ```

    1. `graph.invoke`使用`interrupt_before`和`interrupt_after`参数调用。这是一个运行时配置，可以为每次调用更改。
    2. `interrupt_before`指定应在执行节点之前暂停执行的节点。
    3. `interrupt_after`指定应在执行节点之后暂停执行的节点。
    4. 图运行直到遇到第一个断点。
    5. 通过为输入传递`None`来恢复图。这将运行图直到遇到下一个断点。
  </Tab>
</Tabs>

### 使用LangGraph Studio

您可以使用[LangGraph Studio](/langsmith/studio)在运行图之前在UI中为图设置静态中断。您还可以使用UI检查执行过程中任何点的图状态。

<img src="https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/static-interrupt.png?fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=5aa4e7cea2ab147cef5b4e210dd6c4a1" alt="image" data-og-width="1252" width="1252" data-og-height="1040" height="1040" data-path="oss/images/static-interrupt.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/static-interrupt.png?w=280&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=52d02b507d0a6a879f7fb88d9c6767d0 280w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/static-interrupt.png?w=560&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=e363cd4980edff9bab422f4f1c0ee3c8 560w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/static-interrupt.png?w=840&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=49d26a3641953c23ef3fbc51e828c305 840w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/static-interrupt.png?w=1100&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=2dba15683b3baa1a61bc3bcada35ae1e 1100w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/static-interrupt.png?w=1650&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=9f9a2c0f2631c0e69cd248f6319933fe 1650w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/static-interrupt.png?w=2500&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=5a46b765b436ab5d0dc2f41c01ffad80 2500w" />

***

<Callout icon="pen-to-square" iconType="regular">
  [在GitHub上编辑此页面的源代码。](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/interrupts.mdx)
</Callout>

<Tip icon="terminal" iconType="regular">
  [通过MCP将这些文档以编程方式连接](/use-these-docs)到Claude、VSCode等，以获取实时答案。
</Tip>