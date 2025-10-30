# 工作流程与代理

本指南介绍了常见的工作流程和代理模式。

* 工作流程具有预定义的代码路径，并且被设计为按特定顺序运行。
* 代理是动态的，它们定义自己的过程和工具使用方式。

<img src="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/agent_workflow.png?fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=c217c9ef517ee556cae3fc928a21dc55" alt="代理工作流程" data-og-width="4572" width="4572" data-og-height="2047" height="2047" data-path="oss/images/agent_workflow.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/agent_workflow.png?w=280&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=290e50cff2f72d524a107421ec8e3ff0 280w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/agent_workflow.png?w=560&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=a2bfc87080aee7dd4844f7f24035825e 560w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/agent_workflow.png?w=840&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=ae1fa9087b33b9ff8bc3446ccaa23e3d 840w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/agent_workflow.png?w=1100&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=06003ee1fe07d7a1ea8cf9200e7d0a10 1100w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/agent_workflow.png?w=1650&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=bc98b459a9b1fb226c2887de1696bde0 1650w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/agent_workflow.png?w=2500&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=1933bcdfd5c5b69b98ce96aafa456848 2500w" />

构建代理和工作流程时，LangGraph 提供了多项优势，包括[持久化](/oss/python/langgraph/persistence)、[流式传输](/oss/python/langgraph/streaming)，以及对调试和[部署](/oss/python/langgraph/deploy)的支持。

## 设置

要构建工作流程或代理，可以使用任何支持结构化输出和工具调用的[聊天模型](/oss/python/integrations/chat)。以下示例使用 Anthropic：

1. 安装依赖项：

```bash
pip install langchain_core langchain-anthropic langgraph
```

2. 初始化 LLM：

```python
import os
import getpass

from langchain_anthropic import ChatAnthropic

def _set_env(var: str):
    if not os.environ.get(var):
        os.environ[var] = getpass.getpass(f"{var}: ")


_set_env("ANTHROPIC_API_KEY")

llm = ChatAnthropic(model="claude-sonnet-4-5")
```

## LLM 与增强功能

工作流程和代理系统基于 LLM 以及您添加到它们的各种增强功能。[工具调用](/oss/python/langchain/tools)、[结构化输出](/oss/python/langchain/structured-output)和[短期记忆](/oss/python/langchain/short-term-memory)是一些可以根据您的需求定制 LLM 的选项。

<img src="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/augmented_llm.png?fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=7ea9656f46649b3ebac19e8309ae9006" alt="LLM 增强功能" data-og-width="1152" width="1152" data-og-height="778" height="778" data-path="oss/images/augmented_llm.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/augmented_llm.png?w=280&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=53613048c1b8bd3241bd27900a872ead 280w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/augmented_llm.png?w=560&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=7ba1f4427fd847bd410541ae38d66d40 560w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/augmented_llm.png?w=840&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=503822cf29a28500deb56f463b4244e4 840w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/augmented_llm.png?w=1100&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=279e0440278d3a26b73c72695636272e 1100w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/augmented_llm.png?w=1650&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=d936838b98bc9dce25168e2b2cfd23d0 1650w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/augmented_llm.png?w=2500&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=fa2115f972bc1152b5e03ae590600fa3 2500w" />

```python
# 结构化输出的模式
from pydantic import BaseModel, Field


class SearchQuery(BaseModel):
    search_query: str = Field(None, description="为网络搜索优化的查询。")
    justification: str = Field(
        None, description="为什么此查询与用户的请求相关。"
    )


# 使用结构化输出的模式增强 LLM
structured_llm = llm.with_structured_output(SearchQuery)

# 调用增强后的 LLM
output = structured_llm.invoke("钙 CT 评分与高胆固醇有什么关系？")

# 定义一个工具
def multiply(a: int, b: int) -> int:
    return a * b

# 使用工具增强 LLM
llm_with_tools = llm.bind_tools([multiply])

# 调用触发工具调用的 LLM
msg = llm_with_tools.invoke("2乘以3是多少？")

# 获取工具调用
msg.tool_calls
```

## 提示链式传递

提示链式传递是指每个 LLM 调用处理前一个调用的输出。它通常用于执行可以分解为更小、可验证步骤的明确定义的任务。一些示例包括：

* 将文档翻译成不同语言
* 验证生成的内容是否一致

<img src="https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/prompt_chain.png?fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=762dec147c31b8dc6ebb0857e236fc1f" alt="提示链式传递" data-og-width="1412" width="1412" data-og-height="444" height="444" data-path="oss/images/prompt_chain.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/prompt_chain.png?w=280&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=fda27cf4f997e350d4ce48be16049c47 280w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/prompt_chain.png?w=560&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=1374b6de11900d394fc73722a3a6040e 560w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/prompt_chain.png?w=840&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=25246c7111a87b5df5a2af24a0181efe 840w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/prompt_chain.png?w=1100&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=0c57da86a49cf966cc090497ade347f1 1100w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/prompt_chain.png?w=1650&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=a1b5c8fc644d7a80c0792b71769c97da 1650w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/prompt_chain.png?w=2500&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=8a3f66f0e365e503a85b30be48bc1a76 2500w" />

```python
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END
from IPython.display import Image, display

# 图状态
class State(TypedDict):
    topic: str
    joke: str
    improved_joke: str
    final_joke: str

# 节点
def generate_joke(state: State):
    """第一次 LLM 调用以生成初始笑话"""
    msg = llm.invoke(f"写一个关于{state['topic']}的简短笑话")
    return {"joke": msg.content}

def check_punchline(state: State):
    """门控函数，检查笑话是否有笑点"""
    # 简单检查 - 笑话是否包含"？"或"！"
    if "?" in state["joke"] or "!" in state["joke"]:
        return "Pass"
    return "Fail"

def improve_joke(state: State):
    """第二次 LLM 调用以改进笑话"""
    msg = llm.invoke(f"通过添加文字游戏让这个笑话更有趣：{state['joke']}")
    return {"improved_joke": msg.content}

def polish_joke(state: State):
    """第三次 LLM 调用进行最终润色"""
    msg = llm.invoke(f"给这个笑话添加一个令人惊讶的转折：{state['improved_joke']}")
    return {"final_joke": msg.content}

# 构建工作流程
workflow = StateGraph(State)

# 添加节点
workflow.add_node("generate_joke", generate_joke)
workflow.add_node("improve_joke", improve_joke)
workflow.add_node("polish_joke", polish_joke)

# 添加边以连接节点
workflow.add_edge(START, "generate_joke")
workflow.add_conditional_edges(
    "generate_joke", check_punchline, {"Fail": "improve_joke", "Pass": END}
)
workflow.add_edge("improve_joke", "polish_joke")
workflow.add_edge("polish_joke", END)

# 编译
chain = workflow.compile()

# 显示工作流程
display(Image(chain.get_graph().draw_mermaid_png()))

# 调用
state = chain.invoke({"topic": "猫"})
print("初始笑话：")
print(state["joke"])
print("\n--- --- ---\n")

if "improved_joke" in state:
    print("改进后的笑话：")
    print(state["improved_joke"])
    print("\n--- --- ---\n")
    print("最终笑话：")
    print(state["final_joke"])
else:
    print("笑话未通过质量检测 - 未检测到笑点！")

```

```python
from langgraph.func import entrypoint, task

# 任务
@task
def generate_joke(topic: str):
    """第一次 LLM 调用以生成初始笑话"""
    msg = llm.invoke(f"写一个关于{topic}的简短笑话")
    return msg.content

def check_punchline(joke: str):
    """门控函数，检查笑话是否有笑点"""
    # 简单检查 - 笑话是否包含"？"或"！"
    if "?" in joke or "!" in joke:
        return "Fail"
    return "Pass"

@task
def improve_joke(joke: str):
    """第二次 LLM 调用以改进笑话"""
    msg = llm.invoke(f"通过添加文字游戏让这个笑话更有趣：{joke}")
    return msg.content

@task
def polish_joke(joke: str):
    """第三次 LLM 调用进行最终润色"""
    msg = llm.invoke(f"给这个笑话添加一个令人惊讶的转折：{joke}")
    return msg.content

@entrypoint()
def prompt_chaining_workflow(topic: str):
    original_joke = generate_joke(topic).result()
    if check_punchline(original_joke) == "Pass":
        return original_joke
    improved_joke = improve_joke(original_joke).result()
    return polish_joke(improved_joke).result()

# 调用
for step in prompt_chaining_workflow.stream("猫", stream_mode="updates"):
    print(step)
    print("\n")
```

## 并行化

通过并行化，多个 LLM 同时处理一个任务。这可以通过同时运行多个独立的子任务，或者多次运行同一任务以检查不同的输出来实现。并行化通常用于：

* 分割子任务并行运行，提高速度
* 多次运行任务以检查不同输出，提高置信度

一些示例包括：

* 运行一个处理文档关键词的子任务，同时运行第二个子任务检查格式错误
* 多次运行任务，根据不同标准（如引用数量、使用的来源数量和来源质量）对文档准确性进行评分

<img src="https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/parallelization.png?fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=8afe3c427d8cede6fed1e4b2a5107b71" alt="parallelization.png" data-og-width="1020" width="1020" data-og-height="684" height="684" data-path="oss/images/parallelization.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/parallelization.png?w=280&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=88e51062b14d9186a6f0ea246bc48635 280w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/parallelization.png?w=560&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=934941ca52019b7cbce7fbdd31d00f0f 560w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/parallelization.png?w=840&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=30b5c86c545d0e34878ff0a2c367dd0a 840w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/parallelization.png?w=1100&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=6227d2c39f332eaeda23f7db66871dd7 1100w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/parallelization.png?w=1650&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=283f3ee2924a385ab88f2cbfd9c9c48c 1650w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/parallelization.png?w=2500&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=69f6a97716b38998b7b399c3d8ac7d9c 2500w" />

```python
# 图状态
class State(TypedDict):
    topic: str
    joke: str
    story: str
    poem: str
    combined_output: str

# 节点
def call_llm_1(state: State):
    """第一次 LLM 调用以生成初始笑话"""
    msg = llm.invoke(f"写一个关于{state['topic']}的笑话")
    return {"joke": msg.content}

def call_llm_2(state: State):
    """第二次 LLM 调用以生成故事"""
    msg = llm.invoke(f"写一个关于{state['topic']}的故事")
    return {"story": msg.content}

def call_llm_3(state: State):
    """第三次 LLM 调用以生成诗歌"""
    msg = llm.invoke(f"写一首关于{state['topic']}的诗")
    return {"poem": msg.content}

def aggregator(state: State):
    """将笑话和故事合并为单个输出"""
    combined = f"这是关于{state['topic']}的故事、笑话和诗歌！\n\n"
    combined += f"故事：\n{state['story']}\n\n"
    combined += f"笑话：\n{state['joke']}\n\n"
    combined += f"诗歌：\n{state['poem']}"
    return {"combined_output": combined}

# 构建工作流程
parallel_builder = StateGraph(State)

# 添加节点
parallel_builder.add_node("call_llm_1", call_llm_1)
parallel_builder.add_node("call_llm_2", call_llm_2)
parallel_builder.add_node("call_llm_3", call_llm_3)
parallel_builder.add_node("aggregator", aggregator)

# 添加边以连接节点
parallel_builder.add_edge(START, "call_llm_1")
parallel_builder.add_edge(START, "call_llm_2")
parallel_builder.add_edge(START, "call_llm_3")
parallel_builder.add_edge("call_llm_1", "aggregator")
parallel_builder.add_edge("call_llm_2", "aggregator")
parallel_builder.add_edge("call_llm_3", "aggregator")
parallel_builder.add_edge("aggregator", END)
parallel_workflow = parallel_builder.compile()

# 显示工作流程
display(Image(parallel_workflow.get_graph().draw_mermaid_png()))

# 调用
state = parallel_workflow.invoke({"topic": "猫"})
print(state["combined_output"])
```

```python
@task
def call_llm_1(topic: str):
    """第一次 LLM 调用以生成初始笑话"""
    msg = llm.invoke(f"写一个关于{topic}的笑话")
    return msg.content

@task
def call_llm_2(topic: str):
    """第二次 LLM 调用以生成故事"""
    msg = llm.invoke(f"写一个关于{topic}的故事")
    return msg.content

@task
def call_llm_3(topic):
    """第三次 LLM 调用以生成诗歌"""
    msg = llm.invoke(f"写一首关于{topic}的诗")
    return msg.content

@task
def aggregator(topic, joke, story, poem):
    """将笑话和故事合并为单个输出"""
    combined = f"这是关于{topic}的故事、笑话和诗歌！\n\n"
    combined += f"故事：\n{story}\n\n"
    combined += f"笑话：\n{joke}\n\n"
    combined += f"诗歌：\n{poem}"
    return combined

# 构建工作流程
@entrypoint()
def parallel_workflow(topic: str):
    joke_fut = call_llm_1(topic)
    story_fut = call_llm_2(topic)
    poem_fut = call_llm_3(topic)
    return aggregator(
        topic, joke_fut.result(), story_fut.result(), poem_fut.result()
    ).result()

# 调用
for step in parallel_workflow.stream("猫", stream_mode="updates"):
    print(step)
    print("\n")
```

## 路由

路由工作流程处理输入，然后将它们定向到上下文特定的任务。这使您能够为复杂任务定义专门的流程。例如，构建用于回答产品相关问题的工作流程可能会首先处理问题类型，然后将请求路由到定价、退款、退货等特定流程。

<img src="https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/routing.png?fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=272e0e9b681b89cd7d35d5c812c50ee6" alt="routing.png" data-og-width="1214" width="1214" data-og-height="678" height="678" data-path="oss/images/routing.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/routing.png?w=280&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=ab85efe91d20c816f9a4e491e92a61f7 280w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/routing.png?w=560&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=769e29f9be058a47ee85e0c9228e6e44 560w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/routing.png?w=840&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=3711ee40746670731a0ce3e96b7cfeb1 840w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/routing.png?w=1100&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=9aaa28410da7643f4a2587f7bfae0f21 1100w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/routing.png?w=1650&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=6706326c7fef0511805c684d1e4f7082 1650w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/routing.png?w=2500&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=f6d603145ca33791b18c8c8afec0bb4d 2500w" />

```python
from typing_extensions import Literal
from langchain.messages import HumanMessage, SystemMessage


# 用于路由逻辑的结构化输出模式
class Route(BaseModel):
    step: Literal["诗歌", "故事", "笑话"] = Field(
        None, description="路由过程中的下一步"
    )


# 使用结构化输出增强 LLM
router = llm.with_structured_output(Route)


# 状态
class State(TypedDict):
    input: str
    decision: str
    output: str


# 节点
def llm_call_1(state: State):
    """写一个故事"""

    result = llm.invoke(state["input"])
    return {"output": result.content}


def llm_call_2(state: State):
    """写一个笑话"""

    result = llm.invoke(state["input"])
    return {"output": result.content}


def llm_call_3(state: State):
    """写一首诗"""

    result = llm.invoke(state["input"])
    return {"output": result.content}


def llm_call_router(state: State):
    """将输入路由到适当的节点"""

    # 运行增强的 LLM 并使用结构化输出作为路由逻辑
    decision = router.invoke(
        [
            SystemMessage(
                content="根据用户的请求将输入路由到故事、笑话或诗歌。"
            ),
            HumanMessage(content=state["input"]),
        ]
    )

    return {"decision": decision.step}


# 条件边函数，将路由到适当的节点
def route_decision(state: State):
    # 返回您想要访问的下一个节点名称
    if state["decision"] == "故事":
        return "llm_call_1"
    elif state["decision"] == "笑话":
        return "llm_call_2"
    elif state["decision"] == "诗歌":
        return "llm_call_3"


# 构建工作流程
router_builder = StateGraph(State)

# 添加节点
router_builder.add_node("llm_call_1", llm_call_1)
router_builder.add_node("llm_call_2", llm_call_2)
router_builder.add_node("llm_call_3", llm_call_3)
router_builder.add_node("llm_call_router", llm_call_router)

# 添加边以连接节点
router_builder.add_edge(START, "llm_call_router")
router_builder.add_conditional_edges(
    "llm_call_router",
    route_decision,
    {  # route_decision 返回的名称 : 要访问的下一个节点名称
        "llm_call_1": "llm_call_1",
        "llm_call_2": "llm_call_2",
        "llm_call_3": "llm_call_3",
    },
)
router_builder.add_edge("llm_call_1", END)
router_builder.add_edge("llm_call_2", END)
router_builder.add_edge("llm_call_3", END)

# 编译工作流程
router_workflow = router_builder.compile()

# 显示工作流程
display(Image(router_workflow.get_graph().draw_mermaid_png()))

# 调用
state = router_workflow.invoke({"input": "给我写一个关于猫的笑话"})
print(state["output"])
```

```python
from typing_extensions import Literal
from pydantic import BaseModel
from langchain.messages import HumanMessage, SystemMessage


# 用于路由逻辑的结构化输出模式
class Route(BaseModel):
    step: Literal["诗歌", "故事", "笑话"] = Field(
        None, description="路由过程中的下一步"
    )


# 使用结构化输出增强 LLM
router = llm.with_structured_output(Route)


@task
def llm_call_1(input_: str):
    """写一个故事"""
    result = llm.invoke(input_)
    return result.content


@task
def llm_call_2(input_: str):
    """写一个笑话"""
    result = llm.invoke(input_)
    return result.content


@task
def llm_call_3(input_: str):
    """写一首诗"""
    result = llm.invoke(input_)
    return result.content


def llm_call_router(input_: str):
    """将输入路由到适当的节点"""
    # 运行增强的 LLM 并使用结构化输出作为路由逻辑
    decision = router.invoke(
        [
            SystemMessage(
                content="根据用户的请求将输入路由到故事、笑话或诗歌。"
            ),
            HumanMessage(content=input_),
        ]
    )
    return decision.step


# 创建工作流程
@entrypoint()
def router_workflow(input_: str):
    next_step = llm_call_router(input_)
    if next_step == "故事":
        llm_call = llm_call_1
    elif next_step == "笑话":
        llm_call = llm_call_2
    elif next_step == "诗歌":
        llm_call = llm_call_3

    return llm_call(input_).result()

# 调用
for step in router_workflow.stream("给我写一个关于猫的笑话", stream_mode="updates"):
    print(step)
    print("\n")
```

## 指挥者-工作者

在指挥者-工作者配置中，指挥者：

* 将任务分解为子任务
* 将子任务委派给工作者
* 将工作者的输出合并为最终结果

<img src="https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/worker.png?fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=2e423c67cd4f12e049cea9c169ff0676" alt="worker.png" data-og-width="1486" width="1486" data-og-height="548" height="548" data-path="oss/images/worker.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/worker.png?w=280&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=037222991ea08f889306be035c4730b6 280w, https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/worker.png?w=560&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=081f3ff05cc1fe50770c864d74084b5b 560w, https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/worker.png?w=840&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=0ef6c1b9ceb5159030aa34d0f05f1ada 840w, https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/worker.png?w=1100&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=92ec7353a89ae96e221a5a8f65c88adf 1100w, https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/worker.png?w=1650&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=71b201dd99fa234ebfb918915aac3295 1650w, https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/worker.png?w=2500&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=4f7b6e2064db575027932394a3658fbd 2500w" />

指挥者-工作者工作流程提供了更大的灵活性，通常用于子任务无法像[并行化](#parallelization)那样预先定义的情况。这在编写代码或需要跨多个文件更新内容的工作流程中很常见。例如，需要在未知数量的文档中更新多个 Python 库安装说明的工作流程可能会使用此模式。

```python
from typing import Annotated, List
import operator
# 用于规划的结构化输出模式
class Section(BaseModel):
    name: str = Field(
        description="此部分的名称。",
    )
    description: str = Field(
        description="此部分将涵盖的主要主题和概念的简要概述。",
    )
class Sections(BaseModel):
    sections: List[Section] = Field(
        description="报告的各个部分。",
    )
# 使用结构化输出增强 LLM
planner = llm.with_structured_output(Sections)
```


```python
from typing import List

# 用于规划的结构化输出模式
class Section(BaseModel):
    name: str = Field(
        description="此部分的名称。",
    )
    description: str = Field(
        description="此部分将涵盖的主要主题和概念的简要概述。",
    )

class Sections(BaseModel):
    sections: List[Section] = Field(
        description="报告的各个部分。",
    )

# 使用结构化输出增强 LLM
planner = llm.with_structured_output(Sections)

@task
def orchestrator(topic: str):
    """生成报告计划的指挥者"""
    # 生成查询
    report_sections = planner.invoke(
        [
            SystemMessage(content="生成报告计划。"),
            HumanMessage(content=f"这是报告主题：{topic}"),
        ]
    )
    return report_sections.sections

@task
def llm_call(section: Section):
    """工作者编写报告的一部分"""
    # 生成部分
    result = llm.invoke(
        [
            SystemMessage(content="编写报告部分。"),
            HumanMessage(
                content=f"这是部分名称：{section.name}和描述：{section.description}"
            ),
        ]
    )
    # 将更新后的部分写入已完成部分
    return result.content

@task
def synthesizer(completed_sections: list[str]):
    """从部分合成完整报告"""
    final_report = "\n\n---\n\n".join(completed_sections)
    return final_report

@entrypoint()
def orchestrator_worker(topic: str):
    sections = orchestrator(topic).result()
    section_futures = [llm_call(section) for section in sections]
    final_report = synthesizer(
        [section_fut.result() for section_fut in section_futures]
    ).result()
    return final_report

# 调用
report = orchestrator_worker.invoke("创建一份关于 LLM 缩放定律的报告")
from IPython.display import Markdown
Markdown(report)
```

### 在 LangGraph 中创建工作者

指挥者-工作者工作流程很常见，LangGraph 内置了对它们的支持。`Send` API 允许您动态创建工作者节点并将特定输入发送给它们。每个工作者都有自己的状态，所有工作者的输出都写入到指挥者图可以访问的共享状态键中。这使指挥者可以访问所有工作者的输出，并将它们合成为最终输出。下面的示例遍历部分列表，并使用 `Send` API 将每个部分发送给工作者。

```python
from langgraph.types import Send


# 图状态
class State(TypedDict):
    topic: str  # 报告主题
    sections: list[Section]  # 报告部分列表
    completed_sections: Annotated[
        list, operator.add
    ]  # 所有工作者并行写入此键
    final_report: str  # 最终报告


# 工作者状态
class WorkerState(TypedDict):
    section: Section
    completed_sections: Annotated[list, operator.add]


# 节点
def orchestrator(state: State):
    """生成报告计划的指挥者"""

    # 生成查询
    report_sections = planner.invoke(
        [
            SystemMessage(content="生成报告计划。"),
            HumanMessage(content=f"这是报告主题：{state['topic']}"),
        ]
    )

    return {"sections": report_sections.sections}


def llm_call(state: WorkerState):
    """工作者编写报告的一部分"""

    # 生成部分
    section = llm.invoke(
        [
            SystemMessage(
                content="根据提供的名称和描述编写报告部分。每个部分不要添加序言。使用 markdown 格式。"
            ),
            HumanMessage(
                content=f"这是部分名称：{state['section'].name}和描述：{state['section'].description}"
            ),
        ]
    )

    # 将更新后的部分写入已完成部分
    return {"completed_sections": [section.content]}


def synthesizer(state: State):
    """从部分合成完整报告"""

    # 已完成部分列表
    completed_sections = state["completed_sections"]

    # 将已完成部分格式化为字符串，用作最终部分的上下文
    completed_report_sections = "\n\n---\n\n".join(completed_sections)

    return {"final_report": completed_report_sections}


# 条件边函数，创建编写报告每个部分的 llm_call 工作者
def assign_workers(state: State):
    """为计划中的每个部分分配一个工作者"""

    # 通过 Send() API 并行启动部分编写
    return [Send("llm_call", {"section": s}) for s in state["sections"]]


# 构建工作流程
orchestrator_worker_builder = StateGraph(State)

# 添加节点
orchestrator_worker_builder.add_node("orchestrator", orchestrator)
orchestrator_worker_builder.add_node("llm_call", llm_call)
orchestrator_worker_builder.add_node("synthesizer", synthesizer)

# 添加边以连接节点
orchestrator_worker_builder.add_edge(START, "orchestrator")
orchestrator_worker_builder.add_conditional_edges(
    "orchestrator", assign_workers, ["llm_call"]
)
orchestrator_worker_builder.add_edge("llm_call", "synthesizer")
orchestrator_worker_builder.add_edge("synthesizer", END)

# 编译工作流程
orchestrator_worker = orchestrator_worker_builder.compile()

# 显示工作流程
display(Image(orchestrator_worker.get_graph().draw_mermaid_png()))

# 调用
state = orchestrator_worker.invoke({"topic": "创建一份关于 LLM 缩放定律的报告"})

from IPython.display import Markdown
Markdown(state["final_report"])
```

## 评估者-优化器

在评估者-优化器工作流程中，一个 LLM 调用创建响应，另一个评估该响应。如果评估者或[人工参与回路](/oss/python/langgraph/interrupts)确定响应需要改进，则会提供反馈并重新创建响应。此循环持续生成，直到生成可接受的响应为止。

评估者-优化器工作流程通常用于任务有特定成功标准但需要迭代才能达到这些标准的情况。例如，在两种语言之间翻译文本时，并不总是有完美的匹配。可能需要几次迭代才能生成在两种语言中含义相同的翻译。

<img src="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/evaluator_optimizer.png?fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=9bd0474f42b6040b14ed6968a9ab4e3c" alt="evaluator_optimizer.png" data-og-width="1004" width="1004" data-og-height="340" height="340" data-path="oss/images/evaluator_optimizer.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/evaluator_optimizer.png?w=280&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=ab36856e5f9a518b22e71278aa8b1711 280w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/evaluator_optimizer.png?w=560&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=3ec597c92270278c2bac203d36b611c2 560w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/evaluator_optimizer.png?w=840&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=3ad3bfb734a0e509d9b87fdb4e808bfd 840w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/evaluator_optimizer.png?w=1100&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=e82bd25a463d3cdf76036649c03358a9 1100w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/evaluator_optimizer.png?w=1650&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=d31717ae3e76243dd975a53f46e8c1f6 1650w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/evaluator_optimizer.png?w=2500&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=a9bb4fb1583f6ad06c0b13602cd14811 2500w" />

```python
# 图状态
class State(TypedDict):
    joke: str
    topic: str
    feedback: str
    funny_or_not: str

# 用于评估的结构化输出模式
class Feedback(BaseModel):
    grade: Literal["有趣", "无趣"] = Field(
        description="判断笑话是否有趣。",
    )
    feedback: str = Field(
        description="如果笑话无趣，提供如何改进它的反馈。",
    )

# 使用结构化输出增强 LLM
evaluator = llm.with_structured_output(Feedback)

# 节点
def llm_call_generator(state: State):
    """LLM 生成笑话"""
    if state.get("feedback"):
        msg = llm.invoke(
            f"写一个关于{state['topic']}的笑话，但要考虑反馈：{state['feedback']}"
        )
    else:
        msg = llm.invoke(f"写一个关于{state['topic']}的笑话")
    return {"joke": msg.content}
def llm_call_evaluator(state: State):
    """LLM 评估笑话"""
    grade = evaluator.invoke(f"为笑话评分 {state['joke']}")
    return {"funny_or_not": grade.grade, "feedback": grade.feedback}

# 条件边函数，根据评估者的反馈路由回笑话生成器或结束
def route_joke(state: State):
    """根据评估者的反馈路由回笑话生成器或结束"""
    if state["funny_or_not"] == "有趣":
        return "接受"
    elif state["funny_or_not"] == "无趣":
        return "拒绝 + 反馈"

# 构建工作流程
optimizer_builder = StateGraph(State)

# 添加节点
optimizer_builder.add_node("llm_call_generator", llm_call_generator)
optimizer_builder.add_node("llm_call_evaluator", llm_call_evaluator)

# 添加边以连接节点
optimizer_builder.add_edge(START, "llm_call_generator")
optimizer_builder.add_edge("llm_call_generator", "llm_call_evaluator")
optimizer_builder.add_conditional_edges(
    "llm_call_evaluator",
    route_joke,
    {  # route_joke 返回的名称 : 要访问的下一个节点名称
        "接受": END,
        "拒绝 + 反馈": "llm_call_generator",
    },
)

# 编译工作流程
optimizer_workflow = optimizer_builder.compile()

# 显示工作流程
display(Image(optimizer_workflow.get_graph().draw_mermaid_png()))

# 调用
state = optimizer_workflow.invoke({"topic": "猫"})
print(state["joke"])
```


```python
# 用于评估的结构化输出模式
class Feedback(BaseModel):
    grade: Literal["有趣", "无趣"] = Field(
        description="判断笑话是否有趣。",
    )
    feedback: str = Field(
        description="如果笑话无趣，提供如何改进它的反馈。",
    )

# 使用结构化输出增强 LLM
evaluator = llm.with_structured_output(Feedback)

# 节点
@task
def llm_call_generator(topic: str, feedback: Feedback):
    """LLM 生成笑话"""
    if feedback:
        msg = llm.invoke(
            f"写一个关于{topic}的笑话，但要考虑反馈：{feedback}"
        )
    else:
        msg = llm.invoke(f"写一个关于{topic}的笑话")
    return msg.content

@task
def llm_call_evaluator(joke: str):
    """LLM 评估笑话"""
    feedback = evaluator.invoke(f"为笑话评分 {joke}")
    return feedback

@entrypoint()
def optimizer_workflow(topic: str):
    feedback = None
    while True:
        joke = llm_call_generator(topic, feedback).result()
        feedback = llm_call_evaluator(joke).result()
        if feedback.grade == "有趣":
            break
    return joke

# 调用
for step in optimizer_workflow.stream("猫", stream_mode="updates"):
    print(step)
    print("\n")
```

## 代理

代理通常实现为 LLM 使用[工具](/oss/python/langchain/tools)执行操作。它们在持续反馈循环中运行，用于问题和解决方案不可预测的情况。代理比工作流程具有更多的自主权，可以决定使用哪些工具以及如何解决问题。您仍然可以定义可用的工具集和代理行为指南。

<img src="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/agent.png?fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=bd8da41dbf8b5e6fc9ea6bb10cb63e38" alt="agent.png" data-og-width="1732" width="1732" data-og-height="712" height="712" data-path="oss/images/agent.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/agent.png?w=280&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=f7a590604edc49cfa273b5856f3a3ee3 280w, https://mintcdn.com langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/agent.png?w=560&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=dff9b17d345fe0fea25616b3b0dc6ebf 560w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/agent.png?w=840&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=bd932835b919f5e58be77221b6d0f194 840w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/agent.png?w=1100&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=d53318b0c9c898a6146991691cbac058 1100w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/agent.png?w=1650&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=ea66fb96bc07c595d321b8b71e651ddb 1650w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/agent.png?w=2500&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=b02599a3c9ba2a5c830b9a346f9d26c9 2500w" />


> 要开始使用代理，请参阅[快速入门](/oss/python/langchain/quickstart)或阅读有关 LangChain 中[它们如何工作](/oss/python/langchain/agents)的更多信息。

```python
from langchain.tools import tool


# 定义工具
@tool
def multiply(a: int, b: int) -> int:
    """将 `a` 和 `b` 相乘。

    参数：
        a：第一个整数
        b：第二个整数
    """
    return a * b


@tool
def add(a: int, b: int) -> int:
    """将 `a` 和 `b` 相加。

    参数：
        a：第一个整数
        b：第二个整数
    """
    return a + b


@tool
def divide(a: int, b: int) -> float:
    """将 `a` 除以 `b`。

    参数：
        a：第一个整数
        b：第二个整数
    """
    return a / b


# 使用工具增强 LLM
tools = [add, multiply, divide]
tools_by_name = {tool.name: tool for tool in tools}
llm_with_tools = llm.bind_tools(tools)
```

```python
from langgraph.graph import MessagesState
from langchain.messages import SystemMessage, HumanMessage, ToolMessage

# 节点
def llm_call(state: MessagesState):
    """LLM 决定是否调用工具"""
    return {
        "messages": [
            llm_with_tools.invoke(
                [
                    SystemMessage(
                        content="您是一个执行一组输入上算术运算的有用助手。"
                    )
                ]
                + state["messages"]
            )
        ]
    }

def tool_node(state: dict):
    """执行工具调用"""
    result = []
    for tool_call in state["messages"][-1].tool_calls:
        tool = tools_by_name[tool_call["name"]]
        observation = tool.invoke(tool_call["args"])
        result.append(ToolMessage(content=observation, tool_call_id=tool_call["id"]))
    return {"messages": result}
# 条件边函数，根据 LLM 是否进行工具调用路由到工具节点或结束
def should_continue(state: MessagesState) -> Literal["tool_node", END]:
    """根据 LLM 是否进行工具调用决定我们是否应该继续循环或停止"""
    messages = state["messages"]
    last_message = messages[-1]
    # 如果 LLM 进行工具调用，则执行操作
    if last_message.tool_calls:
        return "tool_node"
    # 否则，我们停止（回复用户）
    return END

# 构建工作流程
agent_builder = StateGraph(MessagesState)

# 添加节点
agent_builder.add_node("llm_call", llm_call)
agent_builder.add_node("tool_node", tool_node)

# 添加边以连接节点
agent_builder.add_edge(START, "llm_call")
agent_builder.add_conditional_edges(
    "llm_call",
    should_continue,
    ["tool_node", END]
)
agent_builder.add_edge("tool_node", "llm_call")

# 编译代理
agent = agent_builder.compile()

# 显示代理
display(Image(agent.get_graph(xray=True).draw_mermaid_png()))

# 调用
messages = [HumanMessage(content="将 3 和 4 相加。")]
messages = agent.invoke({"messages": messages})
for m in messages["messages"]:
    m.pretty_print()
```

```python
from langgraph.graph import add_messages
from langchain.messages import (
    SystemMessage,
    HumanMessage,
    BaseMessage,
    ToolCall,
)

@task
def call_llm(messages: list[BaseMessage]):
    """LLM 决定是否调用工具"""
    return llm_with_tools.invoke(
        [
            SystemMessage(
                content="您是一个执行一组输入上算术运算的有用助手。"
            )
        ]
        + messages
    )

@task
def call_tool(tool_call: ToolCall):
    """执行工具调用"""
    tool = tools_by_name[tool_call["name"]]
    return tool.invoke(tool_call)

@entrypoint()
def agent(messages: list[BaseMessage]):
    llm_response = call_llm(messages).result()
    while True:
        if not llm_response.tool_calls:
            break
        # 执行工具
        tool_result_futures = [
            call_tool(tool_call) for tool_call in llm_response.tool_calls
        ]
        tool_results = [fut.result() for fut in tool_result_futures]
        messages = add_messages(messages, [llm_response, *tool_results])
        llm_response = call_llm(messages).result()
    messages = add_messages(messages, llm_response)
    return messages

# 调用
messages = [HumanMessage(content="将 3 和 4 相加。")]
for chunk in agent.stream(messages, stream_mode="updates"):
    print(chunk)
    print("\n")
```

***

[在 GitHub 上编辑此页面源代码。](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/workflows-agents.mdx)

[通过 MCP 以编程方式连接这些文档](/use-these-docs)，与 Claude、VSCode 等实时交互。