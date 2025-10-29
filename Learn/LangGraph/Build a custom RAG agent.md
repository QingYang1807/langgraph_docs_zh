# 构建自定义RAG代理

## 概述

在本教程中，我们将使用LangGraph构建一个[检索](/oss/python/langchain/retrieval)代理。

LangChain提供了内置的[代理](/oss/python/langchain/agents)实现，这些实现使用[LangGraph](/oss/python/langgraph/overview)原语构建。如果需要更深度的自定义，可以直接在LangGraph中实现代理。本指南演示了一个检索代理的示例实现。[检索](/oss/python/langchain/retrieval)代理在您希望LLM决定是从向量存储中检索上下文还是直接响应用户时非常有用。

在本教程结束时，我们将完成以下工作：

1. 获取并预处理将用于检索的文档。
2. 索引这些文档以进行语义搜索，并为代理创建一个检索器工具。
3. 构建一个能够决定何时使用检索器工具的代理式RAG系统。

<img src="https://mintcdn.com/langchain-5e9cc07a/I6RpA28iE233vhYX/images/langgraph-hybrid-rag-tutorial.png?fit=max&auto=format&n=I6RpA28iE233vhYX&q=85&s=855348219691485642b22a1419939ea7" alt="混合RAG" data-og-width="1615" width="1615" data-og-height="589" height="589" data-path="images/langgraph-hybrid-rag-tutorial.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/I6RpA28iE233vhYX/images/langgraph-hybrid-rag-tutorial.png?w=280&fit=max&auto=format&n=I6RpA28iE233vhYX&q=85&s=09097cb9a1dc57b16d33f084641ea93f 280w, https://mintcdn.com/langchain-5e9cc07a/I6RpA28iE233vhYX/images/langgraph-hybrid-rag-tutorial.png?w=560&fit=max&auto=format&n=I6RpA28iE233vhYX&q=85&s=d0bf85cfa36ac7e1a905593a4688f2d2 560w, https://mintcdn.com/langchain-5e9cc07a/I6RpA28iE233vhYX/images/langgraph-hybrid-rag-tutorial.png?w=840&fit=max&auto=format&n=I6RpA28iE233vhYX&q=85&s=b7626e6ae3cb94fb90a61e6fad69c8ba 840w, https://mintcdn.com/langchain-5e9cc07a/I6RpA28iE233vhYX/images/langgraph-hybrid-rag-tutorial.png?w=1100&fit=max&auto=format&n=I6RpA28iE233vhYX&q=85&s=2425baddda7209901bdde4425c23292c 1100w, https://mintcdn.com/langchain-5e9cc07a/I6RpA28iE233vhYX/images/langgraph-hybrid-rag-tutorial.png?w=1650&fit=max&auto=format&n=I6RpA28iE233vhYX&q=85&s=4e5f030034237589f651b704d0377a76 1650w, https://mintcdn.com/langchain-5e9cc07a/I6RpA28iE233vhYX/images/langgraph-hybrid-rag-tutorial.png?w=2500&fit=max&auto=format&n=I6RpA28iE233vhYX&q=85&s=3ec3c7c91fd2be4d749b1c267027ac1e 2500w" />

### 概念

我们将涵盖以下概念：

* 使用[文档加载器](/oss/python/integrations/document_loaders)、[文本分割器](/oss/python/integrations/splitters)、[嵌入](/oss/python/integrations/text_embedding)和[向量存储](/oss/python/integrations/vectorstores)进行[检索](/oss/python/langchain/retrieval)
* LangGraph[图API](/oss/python/langgraph/graph-api)，包括状态、节点、边和条件边。

## 设置

让我们下载所需的包并设置我们的API密钥：

```python  theme={null}
pip install -U langgraph "langchain[openai]" langchain-community langchain-text-splitters bs4
```

```python  theme={null}
import getpass
import os


def _set_env(key: str):
    if key not in os.environ:
        os.environ[key] = getpass.getpass(f"{key}:")


_set_env("OPENAI_API_KEY")
```

<Tip>
  注册LangSmith以快速发现并改进您的LangGraph项目性能。[LangSmith](https://docs.smith.langchain.com)让您可以使用跟踪数据来调试、测试和监控使用LangGraph构建的LLM应用。
</Tip>

## 1. 预处理文档

1. 获取要在我们的RAG系统中使用的文档。我们将使用[Lilian Weng的博客](https://lilianweng.github.io/)中最新的三篇文章。我们将首先使用`WebBaseLoader`工具获取页面内容：

```python  theme={null}
from langchain_community.document_loaders import WebBaseLoader

urls = [
    "https://lilianweng.github.io/posts/2024-11-28-reward-hacking/",
    "https://lilianweng.github.io/posts/2024-07-07-hallucination/",
    "https://lilianweng.github.io/posts/2024-04-12-diffusion-video/",
]

docs = [WebBaseLoader(url).load() for url in urls]
```

```python  theme={null}
docs[0][0].page_content.strip()[:1000]
```

2. 将获取的文档分割成较小的块，以便索引到我们的向量存储中：

```python  theme={null}
from langchain_text_splitters import RecursiveCharacterTextSplitter

docs_list = [item for sublist in docs for item in sublist]

text_splitter = RecursiveCharacterTextSplitter.from_tiktoken_encoder(
    chunk_size=100, chunk_overlap=50
)
doc_splits = text_splitter.split_documents(docs_list)
```

```python  theme={null}
doc_splits[0].page_content.strip()
```

## 2. 创建检索器工具

既然我们已经分割了文档，就可以将它们索引到用于语义搜索的向量存储中。

1. 使用内存向量存储和OpenAI嵌入：

```python  theme={null}
from langchain_core.vectorstores import InMemoryVectorStore
from langchain_openai import OpenAIEmbeddings

vectorstore = InMemoryVectorStore.from_documents(
    documents=doc_splits, embedding=OpenAIEmbeddings()
)
retriever = vectorstore.as_retriever()
```

2. 使用LangChain预构建的`create_retriever_tool`创建检索器工具：

```python  theme={null}
from langchain_classic.tools.retriever import create_retriever_tool

retriever_tool = create_retriever_tool(
    retriever,
    "retrieve_blog_posts",
    "搜索并返回关于Lilian Weng博客文章的信息。",
)
```

3. 测试该工具：

```python  theme={null}
retriever_tool.invoke({"query": "types of reward hacking"})
```

## 3. 生成查询

现在我们将开始为我们的代理式RAG图构建组件([节点](/oss/python/langgraph/graph-api#nodes)和[边](/oss/python/langgraph/graph-api#edges))。

请注意，这些组件将在[`MessagesState`](/oss/python/langgraph/graph-api#messagesstate)上运行 - 包含包含[聊天消息](https://python.langchain.com/docs/concepts/messages/)列表的`messages`键的图状态。

1. 构建一个`generate_query_or_respond`节点。它将调用LLM根据当前的图状态（消息列表）生成响应。给定输入消息，它将决定使用检索器工具检索，还是直接响应用户。请注意，我们通过`.bind_tools`将之前创建的`retriever_tool`提供给聊天模型：

```python  theme={null}
from langgraph.graph import MessagesState
from langchain.chat_models import init_chat_model

response_model = init_chat_model("openai:gpt-4o", temperature=0)


def generate_query_or_respond(state: MessagesState):
    """调用模型根据当前状态生成响应。根据问题，它将决定使用检索器工具检索，或者直接响应用户。"""
    response = (
        response_model
        .bind_tools([retriever_tool]).invoke(state["messages"])  # [!code highlight]
    )
    return {"messages": [response]}
```

2. 在随机输入上尝试：

```python  theme={null}
input = {"messages": [{"role": "user", "content": "你好！"}]}
generate_query_or_respond(input)["messages"][-1].pretty_print()
```

**输出：**

```
================================== Ai Message ==================================

你好！今天有什么可以帮您的吗？
```

3. 提出一个需要语义搜索的问题：

```python  theme={null}
input = {
    "messages": [
        {
            "role": "user",
            "content": "Lilian Weng对奖励攻击的类型说了什么？",
        }
    ]
}
generate_query_or_respond(input)["messages"][-1].pretty_print()
```

**输出：**

```
================================== Ai Message ==================================
Tool Calls:
retrieve_blog_posts (call_tYQxgfIlnQUDMdtAhdbXNwIM)
Call ID: call_tYQxgfIlnQUDMdtAhdbXNwIM
Args:
    query: types of reward hacking
```

## 4. 评估文档

1. 添加一个[条件边](/oss/python/langgraph/graph-api#conditional-edges) - `grade_documents` - 来确定检索到的文档是否与问题相关。我们将使用具有结构化输出模式`GradeDocuments`的模型进行文档评估。`grade_documents`函数将根据评估决策（`generate_answer`或`rewrite_question`）返回要前往的节点名称：

```python  theme={null}
from pydantic import BaseModel, Field
from typing import Literal

GRADE_PROMPT = (
    "您是一个评估检索到的文档与用户问题相关性的评估者。\n "
    "这是检索到的文档：\n\n {context} \n\n"
    "这是用户问题：{question} \n"
    "如果文档包含与用户问题相关的关键词或语义含义，则将其评为相关。\n"
    "给出二进制评分'yes'或'no'来表示文档是否与问题相关。"
)


class GradeDocuments(BaseModel):  # [!code highlight]
    """使用二进制分数评估文档的相关性。"""

    binary_score: str = Field(
        description="相关性评分：如果相关则为'yes'，不相关则为'no'"
    )


grader_model = init_chat_model("openai:gpt-4o", temperature=0)


def grade_documents(
    state: MessagesState,
) -> Literal["generate_answer", "rewrite_question"]:
    """确定检索到的文档是否与问题相关。"""
    question = state["messages"][0].content
    context = state["messages"][-1].content

    prompt = GRADE_PROMPT.format(question=question, context=context)
    response = (
        grader_model
        .with_structured_output(GradeDocuments).invoke(  # [!code highlight]
            [{"role": "user", "content": prompt}]
        )
    )
    score = response.binary_score

    if score == "yes":
        return "generate_answer"
    else:
        return "rewrite_question"
```

2. 使用工具响应中的不相关文档运行此函数：

```python  theme={null}
from langchain_core.messages import convert_to_messages

input = {
    "messages": convert_to_messages(
        [
            {
                "role": "user",
                "content": "Lilian Weng对奖励攻击的类型说了什么？",
            },
            {
                "role": "assistant",
                "content": "",
                "tool_calls": [
                    {
                        "id": "1",
                        "name": "retrieve_blog_posts",
                        "args": {"query": "types of reward hacking"},
                    }
                ],
            },
            {"role": "tool", "content": "meow", "tool_call_id": "1"},
        ]
    )
}
grade_documents(input)
```

3. 确认相关文档被正确分类：

```python  theme={null}
input = {
    "messages": convert_to_messages(
        [
            {
                "role": "user",
                "content": "Lilian Weng对奖励攻击的类型说了什么？",
            },
            {
                "role": "assistant",
                "content": "",
                "tool_calls": [
                    {
                        "id": "1",
                        "name": "retrieve_blog_posts",
                        "args": {"query": "types of reward hacking"},
                    }
                ],
            },
            {
                "role": "tool",
                "content": "奖励攻击可以分为两种类型：环境或目标规范错误，和奖励篡改",
                "tool_call_id": "1",
            },
        ]
    )
}
grade_documents(input)
```

## 5. 重写问题

1. 构建`rewrite_question`节点。检索器工具可能返回不相关的文档，这表明需要改进原始的用户问题。为此，我们将调用`rewrite_question`节点：

```python  theme={null}
REWRITE_PROMPT = (
    "查看输入并尝试推理底层的语义意图/含义。\n"
    "这是初始问题："
    "\n ------- \n"
    "{question}"
    "\n ------- \n"
    "制定一个改进的问题："
)


def rewrite_question(state: MessagesState):
    """重写原始用户问题。"""
    messages = state["messages"]
    question = messages[0].content
    prompt = REWRITE_PROMPT.format(question=question)
    response = response_model.invoke([{"role": "user", "content": prompt}])
    return {"messages": [{"role": "user", "content": response.content}]}
```

2. 尝试使用：

```python  theme={null}
input = {
    "messages": convert_to_messages(
        [
            {
                "role": "user",
                "content": "Lilian Weng对奖励攻击的类型说了什么？",
            },
            {
                "role": "assistant",
                "content": "",
                "tool_calls": [
                    {
                        "id": "1",
                        "name": "retrieve_blog_posts",
                        "args": {"query": "types of reward hacking"},
                    }
                ],
            },
            {"role": "tool", "content": "meow", "tool_call_id": "1"},
        ]
    )
}

response = rewrite_question(input)
print(response["messages"][-1]["content"])
```

**输出：**

```
Lilian Weng描述了哪些不同类型的奖励攻击，她如何解释它们？
```

## 6. 生成答案

1. 构建`generate_answer`节点：如果我们通过了评估检查，就可以基于原始问题和检索到的上下文生成最终答案：

```python  theme={null}
GENERATE_PROMPT = (
    "您是一个用于问答任务的助手。 "
    "使用以下检索到的上下文片段来回答问题。 "
    "如果您不知道答案，就说不知道。 "
    "最多使用三句话，保持答案简洁。\n"
    "问题：{question} \n"
    "上下文：{context}"
)


def generate_answer(state: MessagesState):
    """生成答案。"""
    question = state["messages"][0].content
    context = state["messages"][-1].content
    prompt = GENERATE_PROMPT.format(question=question, context=context)
    response = response_model.invoke([{"role": "user", "content": prompt}])
    return {"messages": [response]}
```

2. 尝试使用：

```python  theme={null}
input = {
    "messages": convert_to_messages(
        [
            {
                "role": "user",
                "content": "Lilian Weng对奖励攻击的类型说了什么？",
            },
            {
                "role": "assistant",
                "content": "",
                "tool_calls": [
                    {
                        "id": "1",
                        "name": "retrieve_blog_posts",
                        "args": {"query": "types of reward hacking"},
                    }
                ],
            },
            {
                "role": "tool",
                "content": "奖励攻击可以分为两种类型：环境或目标规范错误，和奖励篡改",
                "tool_call_id": "1",
            },
        ]
    )
}

response = generate_answer(input)
response["messages"][-1].pretty_print()
```

**输出：**

```
================================== Ai Message ==================================

Lilian Weng将奖励攻击分为两类：环境或目标规范错误，和奖励篡改。她将奖励攻击视为一个广泛的概念，包括这两个类别。奖励攻击发生在代理利用奖励函数中的缺陷或模糊性以实现高奖励而没有执行预期行为时。
```

## 7. 组装图

现在我们将把所有节点和边组装成一个完整的图：

* 从`generate_query_or_respond`开始，并确定我们是否需要调用`retriever_tool`
* 使用`tools_condition`路由到下一步：
  * 如果`generate_query_or_respond`返回`tool_calls`，调用`retriever_tool`检索上下文
  * 否则，直接响应用户
* 评估检索到的文档内容与问题的相关性(`grade_documents`)并路由到下一步：
  * 如果不相关，使用`rewrite_question`重写问题，然后再次调用`generate_query_or_respond`
  * 如果相关，继续进行`generate_answer`，并使用检索到的文档上下文生成最终响应，使用[`ToolMessage`](https://reference.langchain.com/python/langchain/messages/#langchain.messages.ToolMessage)

```python  theme={null}
from langgraph.graph import StateGraph, START, END
from langgraph.prebuilt import ToolNode, tools_condition

workflow = StateGraph(MessagesState)

# 定义我们将在其间循环的节点
workflow.add_node(generate_query_or_respond)
workflow.add_node("retrieve", ToolNode([retriever_tool]))
workflow.add_node(rewrite_question)
workflow.add_node(generate_answer)

workflow.add_edge(START, "generate_query_or_respond")

# 决定是否检索
workflow.add_conditional_edges(
    "generate_query_or_respond",
    # 评估LLM决策（调用`retriever_tool`工具或响应用户）
    tools_condition,
    {
        # 将条件输出转换为图中的节点
        "tools": "retrieve",
        END: END,
    },
)

# 在调用`action`节点后采取的边。
workflow.add_conditional_edges(
    "retrieve",
    # 评估代理决策
    grade_documents,
)
workflow.add_edge("generate_answer", END)
workflow.add_edge("rewrite_question", "generate_query_or_respond")

# 编译
graph = workflow.compile()
```

可视化该图：

```python  theme={null}
from IPython.display import Image, display

display(Image(graph.get_graph().draw_mermaid_png()))
```

<img src="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/agentic-rag-output.png?fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=ddedbd57514888e614ece260092201df" alt="SQL代理图" style={{ height: "800px" }} data-og-width="1245" width="1245" data-og-height="1395" height="1395" data-path="oss/images/agentic-rag-output.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/agentic-rag-output.png?w=280&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=e8ade9698046fa97bd4600ffc0ee2ffd 280w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/agentic-rag-output.png?w=560&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=67cd8edf5fac7f5a2d23cc4aadaecd20 560w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/agentic-rag-output.png?w=840&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=7c415b76149654aeec54f321e199e5b2 840w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/agentic-rag-output.png?w=1100&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=0c7527bf22c7378c2001fba2bbc3ebad 1100w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/agentic-rag-output.png?w=1650&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=194746e6bf4e46aaadcf32b8f941a736 1650w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/agentic-rag-output.png?w=2500&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=952697cbb31285db207d11a075a2167f 2500w" />

## 8. 运行代理式RAG

现在让我们通过运行一个问题来测试完整的图：

```python  theme={null}
for chunk in graph.stream(
    {
        "messages": [
            {
                "role": "user",
                "content": "Lilian Weng对奖励攻击的类型说了什么？",
            }
        ]
    }
):
    for node, update in chunk.items():
        print("来自节点的更新", node)
        update["messages"][-1].pretty_print()
        print("\n\n")
```

**输出：**

```
来自节点的更新 generate_query_or_respond
================================== Ai Message ==================================
Tool Calls:
  retrieve_blog_posts (call_NYu2vq4km9nNNEFqJwefWKu1)
 Call ID: call_NYu2vq4km9nNNEFqJwefWKu1
  Args:
    query: types of reward hacking



来自节点的更新 retrieve
================================= Tool Message ==================================
Name: retrieve_blog_posts

(注意：一些工作将奖励篡定义为一种不同于奖励攻击的对齐行为独立类别。但在这里我认为奖励攻击是一个更广泛的概念。)
从高层次上看，奖励攻击可以分为两种类型：环境或目标规范错误，和奖励篡改。

为什么存在奖励攻击？#

Pan等人(2022)研究了奖励攻击作为代理能力函数，包括(1)模型大小，(2)动作空间分辨率，(3)观测空间噪声，和(4)训练时间。他们还提出了三种规范代理奖励的分类法：

让我们定义奖励攻击#
RL中的奖励塑造具有挑战性。当RL代理利用奖励函数中的缺陷或模糊性以获得高奖励而没有真正学习预期行为或按设计完成任务时，就会发生奖励攻击。近年来，已经提出了几个相关概念，都指某种形式的奖励攻击：



来自节点的更新 generate_answer
================================== Ai Message ==================================

Lilian Weng将奖励攻击分为两类：环境或目标规范错误，和奖励篡改。她将奖励攻击视为一个广泛的概念，包括这两个类别。奖励攻击发生在代理利用奖励函数中的缺陷或模糊性以获得高奖励而没有执行预期行为时。
```

***

<Callout icon="pen-to-square" iconType="regular">
  [在GitHub上编辑此页面的源代码。](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/agentic-rag.mdx)
</Callout>

<Tip icon="terminal" iconType="regular">
  [通过MCP以编程方式连接这些文档](/use-these-docs)到Claude、VSCode等，以获得实时答案。
</Tip>