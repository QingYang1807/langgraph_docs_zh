# 使用时间旅行功能

当处理基于模型做出决策的非确定性系统（例如由大语言模型驱动的智能体）时，详细检查其决策过程会非常有用：

1. <Icon icon="lightbulb" size={16} /> **理解推理过程**：分析导致成功结果的步骤。
2. <Icon icon="bug" size={16} /> **调试错误**：确定错误发生的位置和原因。
3. <Icon icon="magnifying-glass" size={16} /> **探索替代方案**：测试不同的路径以发现更好的解决方案。

LangGraph提供[时间旅行](/oss/python/langgraph/use-time-travel)功能来支持这些用例。具体来说，您可以从前面的检查点恢复执行——既可以重放相同的状态，也可以修改状态以探索替代方案。在所有情况下，恢复过去的执行都会在历史中创建一个新的分支。

要在LangGraph中使用[时间旅行](/oss/python/langgraph/use-time-travel)功能：

1. [运行图](#1-run-the-graph)：使用[`invoke`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.CompiledStateGraph.invoke)或[`stream`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.CompiledStateGraph.stream)方法，通过初始输入运行图。
2. [识别现有线程中的检查点](#2-identify-a-checkpoint)：使用[`get_state_history`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.CompiledGraph.get_state_history)方法检索特定`thread_id`的执行历史，并找到所需的`checkpoint_id`。或者，在希望执行暂停的节点之前设置[interrupt](/oss/python/langgraph/interrupts)。然后，您可以找到在该中断之前记录的最新检查点。
3. [更新图状态（可选）](#3-update-the-state-optional)：使用[`update_state`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.CompiledGraph.update_state)方法修改检查点处的图状态，并从替代状态恢复执行。
4. [从检查点恢复执行](#4-resume-execution-from-the-checkpoint)：使用`invoke`或`stream`方法，输入为`None`，并包含适当的`thread_id`和`checkpoint_id`的配置。

<Tip>
  有关时间旅行的概念概述，请参阅[时间旅行](/oss/python/langgraph/use-time-travel)。
</Tip>

## 在工作流中的应用

此示例构建了一个简单的LangGraph工作流，它使用大语言模型生成笑话主题并编写笑话。它演示了如何运行图、检索过去的执行检查点、可选地修改状态，以及从选定的检查点恢复执行以探索不同的结果。

### 设置

首先我们需要安装所需的包

```python  theme={null}
%%capture --no-stderr
pip install --quiet -U langgraph langchain_anthropic
```

接下来，我们需要设置我们将使用的Anthropic（大语言模型）的API密钥

```python  theme={null}
import getpass
import os


def _set_env(var: str):
    if not os.environ.get(var):
        os.environ[var] = getpass.getpass(f"{var}: ")


_set_env("ANTHROPIC_API_KEY")
```

<Tip>
  注册[LangSmith](https://smith.langchain.com)以快速识别问题并改进您使用LangGraph构建的项目性能。LangSmith让您可以使用跟踪数据来调试、测试和监控使用LangGraph构建的大语言模型应用程序。
</Tip>

```python  theme={null}
import uuid

from typing_extensions import TypedDict, NotRequired
from langgraph.graph import StateGraph, START, END
from langchain.chat_models import init_chat_model
from langgraph.checkpoint.memory import InMemorySaver


class State(TypedDict):
    topic: NotRequired[str]
    joke: NotRequired[str]


model = init_chat_model(
    "anthropic:claude-sonnet-4-5",
    temperature=0,
)


def generate_topic(state: State):
    """大语言模型调用，生成笑话主题"""
    msg = model.invoke("给我一个有趣的笑话主题")
    return {"topic": msg.content}


def write_joke(state: State):
    """基于主题编写笑话的大语言模型调用"""
    msg = model.invoke(f"关于{state['topic']}写一个简短的笑话")
    return {"joke": msg.content}


# 构建工作流
workflow = StateGraph(State)

# 添加节点
workflow.add_node("generate_topic", generate_topic)
workflow.add_node("write_joke", write_joke)

# 添加边来连接节点
workflow.add_edge(START, "generate_topic")
workflow.add_edge("generate_topic", "write_joke")
workflow.add_edge("write_joke", END)

# 编译
checkpointer = InMemorySaver()
graph = workflow.compile(checkpointer=checkpointer)
graph
```

### 1. 运行图

```python  theme={null}
config = {
    "configurable": {
        "thread_id": uuid.uuid4(),
    }
}
state = graph.invoke({}, config)

print(state["topic"])
print()
print(state["joke"])
```

**输出：**

```
不如试试"干衣机中的袜子秘密生活"？你知道，探索袜子以成对进入洗衣篮却以单只出现的神秘现象。它们去哪儿了？它们是否在其他地方开始了新生活？有没有我们不知道的袜子天堂？这个困扰我们所有人的日常之谜有很多喜剧潜力！

# 干衣机中的袜子秘密生活

我终于发现了所有失踪的袜子在干衣机后去了哪里。事实证明，它们根本没有失踪——它们只是带着来自自助洗衣店的其他私奔袜子开始新生活了。

我的蓝色菱形格子袜现在百慕大和红色圆点袜一起生活，在Sockstagram上度假照片，并给我寄送棉絮作为赡养费。
```

### 2. 识别检查点

```python  theme={null}
# 状态按逆时序顺序返回。
states = list(graph.get_state_history(config))

for state in states:
    print(state.next)
    print(state.config["configurable"]["checkpoint_id"])
    print()
```

**输出：**

```
()
1f02ac4a-ec9f-6524-8002-8f7b0bbeed0e

('write_joke',)
1f02ac4a-ce2a-6494-8001-cb2e2d651227

('generate_topic',)
1f02ac4a-a4e0-630d-8000-b73c254ba748

('__start__',)
1f02ac4a-a4dd-665e-bfff-e6c8c44315d9
```

```python  theme={null}
# 这是倒数第二个状态（状态按时间顺序列出）
selected_state = states[1]
print(selected_state.next)
print(selected_state.values)
```

**输出：**

```
('write_joke',)
{'topic': '不如试试"干衣机中的袜子秘密生活"？你知道，探索袜子以成对进入洗衣篮却以单只出现的神秘现象。它们去哪儿了？它们是否在其他地方开始了新生活？有没有我们不知道的袜子天堂？这个困扰我们所有人的日常之谜有很多喜剧潜力！'}
```

<a id="optional" />

### 3. 更新状态

[`update_state`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.CompiledGraph.update_state)将创建一个新的检查点。新检查点将与同一个线程关联，但具有新的检查点ID。

```python  theme={null}
new_config = graph.update_state(selected_state.config, values={"topic": "鸡"})
print(new_config)
```

**输出：**

```
{'configurable': {'thread_id': 'c62e2e03-c27b-4cb6-8cea-ea9bfedae006', 'checkpoint_ns': '', 'checkpoint_id': '1f02ac4a-ecee-600b-8002-a1d21df32e4c'}}
```

### 4. 从检查点恢复执行

```python  theme={null}
graph.invoke(None, new_config)
```

**输出：**

```python  theme={null}
{'topic': '鸡',
 'joke': '为什么鸡要加入乐队？\n\n因为它有出色的鼓槌！'}
```

***

<Callout icon="pen-to-square" iconType="regular">
  [在GitHub上编辑此页面源码。](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/use-time-travel.mdx)
</Callout>

<Tip icon="terminal" iconType="regular">
  [通过MCP以编程方式连接这些文档](/use-these-docs)到Claude、VSCode等，获取实时答案。
</Tip>