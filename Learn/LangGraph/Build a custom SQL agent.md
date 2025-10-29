# 构建自定义SQL代理

在本教程中，我们将构建一个自定义代理，使用LangGraph来回答有关SQL数据库的问题。

LangChain提供了内置的[代理](/oss/python/langchain/agents)实现，这些实现使用[LangGraph](/oss/python/langgraph/overview)原语构建。如果需要更深度的自定义，可以直接在LangGraph中实现代理。本指南演示了一个SQL代理的示例实现。您可以找到使用更高级的LangChain抽象构建SQL代理的[教程](/oss/python/langchain/sql-agent)。

<警告>
  构建SQL数据库的问答系统需要执行模型生成的SQL查询。这样做存在固有风险。请确保您的数据库连接权限始终根据代理需求尽可能严格限制。这将减轻（但不能消除）构建模型驱动系统的风险。
</警告>

[预构建代理](/oss/python/langchain/sql-agent)让我们能够快速开始，但我们依赖系统提示来约束其行为——例如，我们指示代理总是以"列出表"工具开始，并在执行查询之前总是运行查询检查器工具。

我们可以在LangGraph中通过自定义代理来执行更高程度的控制。在这里，我们实现了一个简单的ReAct代理设置，有专用的节点用于特定的工具调用。我们将使用与预构建代理相同的[状态]。

### 概念

我们将涵盖以下概念：

* 用于从SQL数据库读取的[工具](/oss/python/langchain/tools)
* LangGraph[图API](/oss/python/langgraph/graph-api)，包括状态、节点、边和条件边。
* [人在回路中](/oss/python/langgraph/interrupts)的流程

## 设置

### 安装

<代码组>
  ```bash pip theme={null}
  pip install langchain  langgraph  langchain-community
  ```
</代码组>

### LangSmith

设置[LangSmith](https://smith.langchain.com)来检查您链或代理内部发生的情况。然后设置以下环境变量：

```shell  theme={null}
export LANGSMITH_TRACING="true"
export LANGSMITH_API_KEY="..."
```

## 1. 选择LLM

选择支持[工具调用](/oss/python/integrations/providers/overview)的模型：

<标签页>
  <标签页标题="OpenAI">
    👉 阅读[OpenAI聊天模型集成文档](/oss/python/integrations/chat/openai/)

    ```shell  theme={null}
    pip install -U "langchain[openai]"
    ```

    <代码组>
      ```python init_chat_model theme={null}
      import os
      from langchain.chat_models import init_chat_model

      os.environ["OPENAI_API_KEY"] = "sk-..."

      model = init_chat_model("openai:gpt-4.1")
      ```

      ```python Model Class theme={null}
      import os
      from langchain_openai import ChatOpenAI

      os.environ["OPENAI_API_KEY"] = "sk-..."

      model = ChatOpenAI(model="gpt-4.1")
      ```
    </代码组>
  </标签页>

  <标签页标题="Anthropic">
    👉 阅读[Anthropic聊天模型集成文档](/oss/python/integrations/chat/anthropic/)

    ```shell  theme={null}
    pip install -U "langchain[anthropic]"
    ```

    <代码组>
      ```python init_chat_model theme={null}
      import os
      from langchain.chat_models import init_chat_model

      os.environ["ANTHROPIC_API_KEY"] = "sk-..."

      model = init_chat_model("anthropic:claude-sonnet-4-5")
      ```

      ```python Model Class theme={null}
      import os
      from langchain_anthropic import ChatAnthropic

      os.environ["ANTHROPIC_API_KEY"] = "sk-..."

      model = ChatAnthropic(model="claude-sonnet-4-5")
      ```
    </代码组>
  </标签页>

  <标签页标题="Azure">
    👉 阅读[Azure聊天模型集成文档](/oss/python/integrations/chat/azure_chat_openai/)

    ```shell  theme={null}
    pip install -U "langchain[openai]"
    ```

    <代码组>
      ```python init_chat_model theme={null}
      import os
      from langchain.chat_models import init_chat_model

      os.environ["AZURE_OPENAI_API_KEY"] = "..."
      os.environ["AZURE_OPENAI_ENDPOINT"] = "..."
      os.environ["OPENAI_API_VERSION"] = "2025-03-01-preview"

      model = init_chat_model(
          "azure_openai:gpt-4.1",
          azure_deployment=os.environ["AZURE_OPENAI_DEPLOYMENT_NAME"],
      )
      ```

      ```python Model Class theme={null}
      import os
      from langchain_openai import AzureChatOpenAI

      os.environ["AZURE_OPENAI_API_KEY"] = "..."
      os.environ["AZURE_OPENAI_ENDPOINT"] = "..."
      os.environ["OPENAI_API_VERSION"] = "2025-03-01-preview"

      model = AzureChatOpenAI(
          model="gpt-4.1",
          azure_deployment=os.environ["AZURE_OPENAI_DEPLOYMENT_NAME"]
      )
      ```
    </代码组>
  </标签页>

  <标签页标题="Google Gemini">
    👉 阅读[Google GenAI聊天模型集成文档](/oss/python/integrations/chat/google_generative_ai/)

    ```shell  theme={null}
    pip install -U "langchain[google-genai]"
    ```

    <代码组>
      ```python init_chat_model theme={null}
      import os
      from langchain.chat_models import init_chat_model

      os.environ["GOOGLE_API_KEY"] = "..."

      model = init_chat_model("google_genai:gemini-2.5-flash-lite")
      ```

      ```python Model Class theme={null}
      import os
      from langchain_google_genai import ChatGoogleGenerativeAI

      os.environ["GOOGLE_API_KEY"] = "..."

      model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite")
      ```
    </代码组>
  </标签页>

  <标签页标题="AWS Bedrock">
    👉 阅读[AWS Bedrock聊天模型集成文档](/oss/python/integrations/chat/bedrock/)

    ```shell  theme={null}
    pip install -U "langchain[aws]"
    ```

    <代码组>
      ```python init_chat_model theme={null}
      from langchain.chat_models import init_chat_model

      # Follow the steps here to configure your credentials:
      # https://docs.aws.amazon.com/bedrock/latest/userguide/getting-started.html

      model = init_chat_model(
          "anthropic.claude-3-5-sonnet-20240620-v1:0",
          model_provider="bedrock_converse",
      )
      ```

      ```python Model Class theme={null}
      from langchain_aws import ChatBedrock

      model = ChatBedrock(model="anthropic.claude-3-5-sonnet-20240620-v1:0")
      ```
    </代码组>
  </标签页>
</标签页>

下面示例中显示的输出使用的是OpenAI。

## 2. 配置数据库

在本教程中，您将创建一个[SQLite数据库](https://www.sqlitetutorial.net/sqlite-sample-database/)。SQLite是一个轻量级数据库，易于设置和使用。我们将加载`chinook`数据库，这是一个表示数字媒体商店的示例数据库。

为了方便起见，我们已将数据库(`Chinook.db`)托管在公共GCS存储桶上。

```python  theme={null}
import requests, pathlib

url = "https://storage.googleapis.com/benchmarks-artifacts/chinook/Chinook.db"
local_path = pathlib.Path("Chinook.db")

if local_path.exists():
    print(f"{local_path} 已存在，跳过下载。")
else:
    response = requests.get(url)
    if response.status_code == 200:
        local_path.write_bytes(response.content)
        print(f"文件已下载并保存为 {local_path}")
    else:
        print(f"下载文件失败。状态码: {response.status_code}")
```

我们将使用`langchain_community`包中可用的便捷SQL数据库包装器来与数据库交互。该包装器提供了一个简单界面来执行SQL查询和获取结果：

```python  theme={null}
from langchain_community.utilities import SQLDatabase

db = SQLDatabase.from_uri("sqlite:///Chinook.db")

print(f"方言: {db.dialect}")
print(f"可用表: {db.get_usable_table_names()}")
print(f"示例输出: {db.run("SELECT * FROM Artist LIMIT 5;")}")
```

```
方言: sqlite
可用表: ['Album', 'Artist', 'Customer', 'Employee', 'Genre', 'Invoice', 'InvoiceLine', 'MediaType', 'Playlist', 'PlaylistTrack', 'Track']
示例输出: [(1, 'AC/DC'), (2, 'Accept'), (3, 'Aerosmith'), (4, 'Alanis Morissette'), (5, 'Alice In Chains')]
```

## 3. 添加数据库交互工具

使用`langchain_community`包中可用的`SQLDatabase`包装器来与数据库交互。该包装器提供了一个简单界面来执行SQL查询和获取结果：

```python  theme={null}
from langchain_community.agent_toolkits import SQLDatabaseToolkit

toolkit = SQLDatabaseToolkit(db=db, llm=llm)

tools = toolkit.get_tools()

for tool in tools:
    print(f"{tool.name}: {tool.description}\n")
```

```
sql_db_query: 此工具的输入是一个详细且正确的SQL查询，输出是数据库中的结果。如果查询不正确，将返回错误消息。如果返回错误，请重写查询、检查查询并重试。如果您遇到"Unknown column 'xxxx' in 'field list'"问题，请使用sql_db_schema查询正确的表字段。

sql_db_schema: 此工具的输入是逗号分隔的表列表，输出是这些表的架构和样本行。务必先调用sql_db_list_tables确保表确实存在！示例输入: table1, table2, table3

sql_db_list_tables: 输入为空字符串，输出是数据库中的表列表，以逗号分隔。

sql_db_query_checker: 使用此工具在执行查询前双重检查您的查询是否正确。始终在使用sql_db_query执行查询前使用此工具！
```

## 4. 定义应用程序步骤

我们为以下步骤构建专用节点：

* 列出数据库表
* 调用"获取架构"工具
* 生成查询
* 检查查询

将这些步骤放入专用节点让我们能够：(1)在需要时强制工具调用，以及(2)自定义与每个步骤关联的提示。

```python  theme={null}
from typing import Literal

from langchain.agents import ToolNode
from langchain.messages import AIMessage
from langchain_core.runnables import RunnableConfig
from langgraph.graph import END, START, MessagesState, StateGraph


get_schema_tool = next(tool for tool in tools if tool.name == "sql_db_schema")
get_schema_node = ToolNode([get_schema_tool], name="get_schema")

run_query_tool = next(tool for tool in tools if tool.name == "sql_db_query")
run_query_node = ToolNode([run_query_tool], name="run_query")


# 示例：创建预定的工具调用
def list_tables(state: MessagesState):
    tool_call = {
        "name": "sql_db_list_tables",
        "args": {},
        "id": "abc123",
        "type": "tool_call",
    }
    tool_call_message = AIMessage(content="", tool_calls=[tool_call])

    list_tables_tool = next(tool for tool in tools if tool.name == "sql_db_list_tables")
    tool_message = list_tables_tool.invoke(tool_call)
    response = AIMessage(f"可用表: {tool_message.content}")

    return {"messages": [tool_call_message, tool_message, response]}


# 示例：强制模型创建工具调用
def call_get_schema(state: MessagesState):
    # 注意LangChain强制所有模型接受 `tool_choice="any"`
    # 以及 `tool_choice=<工具名称字符串>`。
    llm_with_tools = llm.bind_tools([get_schema_tool], tool_choice="any")
    response = llm_with_tools.invoke(state["messages"])

    return {"messages": [response]}


generate_query_system_prompt = """
您是一个旨在与SQL数据库交互的代理。
给定一个输入问题，创建一个语法正确的{dialect}查询来运行，
然后查看查询结果并返回答案。除非用户指定了他们希望获得的特定示例数量，
否则始终将查询限制为最多{top_k}个结果。

您可以通过相关列对结果排序，以返回数据库中最有趣的示例。
永远不要查询特定表的所有列，只根据问题询问相关列。

不要对数据库执行任何DML语句(INSERT, UPDATE, DELETE, DROP等)。
""".format(
    dialect=db.dialect,
    top_k=5,
)


def generate_query(state: MessagesState):
    system_message = {
        "role": "system",
        "content": generate_query_system_prompt,
    }
    # 我们在这里不强制工具调用，允许模型
    # 在获得解决方案时自然响应。
    llm_with_tools = llm.bind_tools([run_query_tool])
    response = llm_with_tools.invoke([system_message] + state["messages"])

    return {"messages": [response]}


check_query_system_prompt = """
您是一个具有高度细致注意力的SQL专家。
仔细检查{dialect}查询中的常见错误，包括：
- 使用NOT IN与NULL值
- 应该使用UNION ALL时却使用了UNION
- 使用BRAIN表示排除性范围
- 谓词中的数据类型不匹配
- 正确引用标识符
- 为函数使用正确数量的参数
- 转换为正确的数据类型
- 为连接使用正确的列

如果存在上述任何错误，请重写查询。如果没有错误，
只需重现原始查询。

您将在运行此检查后调用适当的工具来执行查询。
""".format(dialect=db.dialect)


def check_query(state: MessagesState):
    system_message = {
        "role": "system",
        "content": check_query_system_prompt,
    }

    # 生成人工用户消息进行检查
    tool_call = state["messages"][-1].tool_calls[0]
    user_message = {"role": "user", "content": tool_call["args"]["query"]}
    llm_with_tools = llm.bind_tools([run_query_tool], tool_choice="any")
    response = llm_with_tools.invoke([system_message, user_message])
    response.id = state["messages"][-1].id

    return {"messages": [response]}
```

## 5. 实现代理

我们现在可以使用[图API](/oss/python/langgraph/graph-api)将这些步骤组装成工作流。我们在查询生成步骤定义了一个[条件边](/oss/python/langgraph/graph-api#conditional-edges)，它将路由到查询检查器（如果生成了查询），如果没有工具调用存在（即LLM已提供查询的响应）则结束。

```python  theme={null}
def should_continue(state: MessagesState) -> Literal[END, "check_query"]:
    messages = state["messages"]
    last_message = messages[-1]
    if not last_message.tool_calls:
        return END
    else:
        return "check_query"


builder = StateGraph(MessagesState)
builder.add_node(list_tables)
builder.add_node(call_get_schema)
builder.add_node(get_schema_node, "get_schema")
builder.add_node(generate_query)
builder.add_node(check_query)
builder.add_node(run_query_node, "run_query")

builder.add_edge(START, "list_tables")
builder.add_edge("list_tables", "call_get_schema")
builder.add_edge("call_get_schema", "get_schema")
builder.add_edge("get_schema", "generate_query")
builder.add_conditional_edges(
    "generate_query",
    should_continue,
)
builder.add_edge("check_query", "run_query")
builder.add_edge("run_query", "generate_query")

agent = builder.compile()
```

我们在下面可视化应用程序：

```python  theme={null}
from IPython.display import Image, display
from langchain_core.runnables.graph import CurveStyle, MermaidDrawMethod, NodeStyles

display(Image(agent.get_graph().draw_mermaid_png()))
```

<img src="https://mintcdn.com/langchain-5e9cc07a/aAi4RLdXQAh8fThS/oss/images/sql-agent-langgraph.png?fit=max&auto=format&n=aAi4RLdXQAh8fThS&q=85&s=1ddd4aae369fb8c143edaccb0a09c81f" alt="SQL agent graph" style={{ height: "800px" }} data-og-width="308" width="308" data-og-height="645" height="645" data-path="oss/images/sql-agent-langgraph.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/aAi4RLdXQAh8fThS/oss/images/sql-agent-langgraph.png?w=280&fit=max&auto=format&n=aAi4RLdXQAh8fThS&q=85&s=e5d3e67f17d65e438370f7d771e3ba7d 280w, https://mintcdn.com/langchain-5e9cc07a/aAi4RLdXQAh8fThS/oss/images/sql-agent-langgraph.png?w=560&fit=max&auto=format&n=aAi4RLdXQAh8fThS&q=85&s=dbcb80fdb2d00a6dc33dc90f05d100b5 560w, https://mintcdn.com/langchain-5e9cc07a/aAi4RLdXQAh8fThS/oss/images/sql-agent-langgraph.png?w=840&fit=max&auto=format&n=aAi4RLdXQAh8fThS&q=85&s=72be69a1e7ac39afad3d0aa03ecffffa 840w, https://mintcdn.com/langchain-5e9cc07a/aAi4RLdXQAh8fThS/oss/images/sql-agent-langgraph.png?w=1100&fit=max&auto=format&n=aAi4RLdXQAh8fThS&q=85&s=5ad351b8b6641defe17882f5e102cab0 1100w, https://mintcdn.com/langchain-5e9cc07a/aAi4RLdXQAh8fThS/oss/images/sql-agent-langgraph.png?w=1650&fit=max&auto=format&n=aAi4RLdXQAh8fThS&q=85&s=8a5cefc8ac6938d0b4b0946e0522ffaa 1650w, https://mintcdn.com/langchain-5e9cc07a/aAi4RLdXQAh8fThS/oss/images/sql-agent-langgraph.png?w=2500&fit=max&auto=format&n=aAi4RLdXQAh8fThS&q=85&s=0b5b7711b4b2ece3a3ccb10a2b012166 2500w" />

我们现在可以调用图：

```python  theme={null}
question = "哪个类型的曲目平均最长？"

for step in agent.stream(
    {"messages": [{"role": "user", "content": question}]},
    stream_mode="values",
):
    step["messages"][-1].pretty_print()
```

```
================================ 人类消息 =================================

哪个类型的曲目平均最长？
================================== AI消息 ==================================

可用表: Album, Artist, Customer, Employee, Genre, Invoice, InvoiceLine, MediaType, Playlist, PlaylistTrack, Track
================================== AI消息 ==================================
工具调用:
  sql_db_schema (call_yzje0tj7JK3TEzDx4QnRR3lL)
 调用ID: call_yzje0tj7JK3TEzDx4QnRR3lL
  参数:
    table_names: Genre, Track
================================= 工具消息 =================================
名称: sql_db_schema


CREATE TABLE "Genre" (
	"GenreId" INTEGER NOT NULL,
	"Name" NVARCHAR(120),
	PRIMARY KEY ("GenreId")
)

/*
Genre表中的3行:
GenreId	Name
1	Rock
2	Jazz
3	Metal
*/


CREATE TABLE "Track" (
	"TrackId" INTEGER NOT NULL,
	"Name" NVARCHAR(200) NOT NULL,
	"AlbumId" INTEGER,
	"MediaTypeId" INTEGER NOT NULL,
	"GenreId" INTEGER,
	"Composer" NVARCHAR(220),
	"Milliseconds" INTEGER NOT NULL,
	"Bytes" INTEGER,
	"UnitPrice" NUMERIC(10, 2) NOT NULL,
	PRIMARY KEY ("TrackId"),
	FOREIGN KEY("MediaTypeId") REFERENCES "MediaType" ("MediaTypeId"),
	FOREIGN KEY("GenreId") REFERENCES "Genre" ("GenreId"),
	FOREIGN KEY("AlbumId") REFERENCES "Album" ("AlbumId")
)

/*
Track表中的3行:
TrackId	Name	AlbumId	MediaTypeId	GenreId	Composer	Milliseconds	Bytes	UnitPrice
1	For Those About To Rock (We Salute You)	1	1	1	Angus Young, Malcolm Young, Brian Johnson	343719	11170334	0.99
2	Balls to the Wall	2	2	1	U. Dirkschneider, W. Hoffmann, H. Frank, P. Baltes, S. Kaufmann, G. Hoffmann	342562	5510424	0.99
3	Fast As a Shark	3	2	1	F. Baltes, S. Kaufman, U. Dirkscneider & W. Hoffman	230619	3990994	0.99
*/
================================== AI消息 ==================================
工具调用:
  sql_db_query (call_cb9ApLfZLSq7CWg6jd0im90b)
 调用ID: call_cb9ApLfZLSq7CWg6jd0im90b
  参数:
    query: SELECT Genre.Name, AVG(Track.Milliseconds) AS AvgMilliseconds FROM Track JOIN Genre ON Track.GenreId = Genre.GenreId GROUP BY Genre.GenreId ORDER BY AvgMilliseconds DESC LIMIT 5;
================================== AI消息 ==================================
工具调用:
  sql_db_query (call_DMVALfnQ4kJsuF3Yl6jxbeAU)
 调用ID: call_DMVALfnQ4kJsuF3Yl6jxbeAU
  参数:
    query: SELECT Genre.Name, AVG(Track.Milliseconds) AS AvgMilliseconds FROM Track JOIN Genre ON Track.GenreId = Genre.GenreId GROUP BY Genre.GenreId ORDER BY AvgMilliseconds DESC LIMIT 5;
================================= 工具消息 =================================
名称: sql_db_query

[('Sci Fi & Fantasy', 2911783.0384615385), ('Science Fiction', 2625549.076923077), ('Drama', 2575283.78125), ('TV Shows', 2145041.0215053763), ('Comedy', 1585263.705882353)]
================================== AI消息 ==================================

平均曲目长度最长的类型是"Sci Fi & Fantasy"，平均曲目长度约为2,911,783毫秒。其他相对曲目长度较长的类型包括"Science Fiction"、"Drama"、"TV Shows"和"Comedy"。
```

<提示>
  查看上述运行的[LangSmith追踪记录](https://smith.langchain.com/public/94b8c9ac-12f7-4692-8706-836a1f30f1ea/r)。
</提示>

## 6. 实现人在回路审查

在执行SQL查询之前，检查代理的SQL查询以防止任何意外操作或低效做法是谨慎的做法。

在这里，我们利用LangGraph的[人在回路中](/oss/python/langgraph/interrupts)功能，在执行SQL查询前暂停运行并等待人工审查。使用LangGraph的[持久化层](/oss/python/langgraph/persistence)，我们可以无限期地暂停运行（或者至少在持久化层存活期间如此）。

让我们将`sql_db_query`工具包装在一个接收人工输入的节点中。我们可以使用[中断](/oss/python/langgraph/interrupts)函数来实现这一点。下面，我们允许输入以批准工具调用、编辑其参数或提供用户反馈。

```python  theme={null}
from langchain_core.runnables import RunnableConfig
from langchain.tools import tool
from langgraph.types import interrupt

@tool(
    run_query_tool.name,
    description=run_query_tool.description,
    args_schema=run_query_tool.args_schema
)
def run_query_tool_with_interrupt(config: RunnableConfig, **tool_input):
    request = {
        "action": run_query_tool.name,
        "args": tool_input,
        "description": "请审查工具调用"
    }
    response = interrupt([request]) # [!code highlight]
    # 批准工具调用
    if response["type"] == "accept":
        tool_response = run_query_tool.invoke(tool_input, config)
    # 更新工具调用参数
    elif response["type"] == "edit":
        tool_input = response["args"]["args"]
        tool_response = run_query_tool.invoke(tool_input, config)
    # 用用户反馈响应LLM
    elif response["type"] == "response":
        user_feedback = response["args"]
        tool_response = user_feedback
    else:
        raise ValueError(f"不支持的中断响应类型: {response['type']}")

    return tool_response
```

<注意>
  上述实现遵循了更广泛的[人在回路中](/oss/python/langgraph/interrupts)指南中的[工具中断示例](/oss/python/langgraph/interrupts#configuring-interrupts)。请参考该指南获取详细信息和替代方案。
</注意>

现在让我们重新组装我们的图。我们将用人工审查替换程序化检查。注意，我们现在包含了[检查器](/oss/python/langgraph/persistence)；这是暂停和恢复运行所必需的。

```python  theme={null}
from langgraph.checkpoint.memory import InMemorySaver

def should_continue(state: MessagesState) -> Literal[END, "run_query"]:
    messages = state["messages"]
    last_message = messages[-1]
    if not last_message.tool_calls:
        return END
    else:
        return "run_query"

builder = StateGraph(MessagesState)
builder.add_node(list_tables)
builder.add_node(call_get_schema)
builder.add_node(get_schema_node, "get_schema")
builder.add_node(generate_query)
builder.add_node(run_query_node, "run_query")

builder.add_edge(START, "list_tables")
builder.add_edge("list_tables", "call_get_schema")
builder.add_edge("call_get_schema", "get_schema")
builder.add_edge("get_schema", "generate_query")
builder.add_conditional_edges(
    "generate_query",
    should_continue,
)
builder.add_edge("run_query", "generate_query")

checkpointer = InMemorySaver() # [!code highlight]
agent = builder.compile(checkpointer=checkpointer) # [!code highlight]
```

我们可以像以前一样调用图。这次执行被中断了：

```python  theme={null}
import json

config = {"configurable": {"thread_id": "1"}}

question = "哪个类型的曲目平均最长？"

for step in agent.stream(
    {"messages": [{"role": "user", "content": question}]},
    config,
    stream_mode="values",
):
    if "messages" in step:
        step["messages"][-1].pretty_print()
    elif "__interrupt__" in step:
        action = step["__interrupt__"][0]
        print("中断:")
        for request in action.value:
            print(json.dumps(request, indent=2))
    else:
        pass
```

```
...

中断:
{
  "action": "sql_db_query",
  "args": {
    "query": "SELECT Genre.Name, AVG(Track.Milliseconds) AS AvgLength FROM Track JOIN Genre ON Track.GenreId = Genre.GenreId GROUP BY Genre.Name ORDER BY AvgLength DESC LIMIT 5;"
  },
  "description": "请审查工具调用"
}
```

我们可以使用[命令](/oss/python/langgraph/use-graph-api#combine-control-flow-and-state-updates-with-command)接受或编辑工具调用：

```python  theme={null}
from langgraph.types import Command


for step in agent.stream(
    Command(resume={"type": "accept"}),
    # Command(resume={"type": "edit", "args": {"query": "..."}}),
    config,
    stream_mode="values",
):
    if "messages" in step:
        step["messages"][-1].pretty_print()
    elif "__interrupt__" in step:
        action = step["__interrupt__"][0]
        print("中断:")
        for request in action.value:
            print(json.dumps(request, indent=2))
    else:
        pass
```

```
================================== AI消息 ==================================
工具调用:
  sql_db_query (call_t4yXkD6shwdTPuelXEmY3sAY)
 调用ID: call_t4yXkD6shwdTPuelXEmY3sAY
  参数:
    query: SELECT Genre.Name, AVG(Track.Milliseconds) AS AvgLength FROM Track JOIN Genre ON Track.GenreId = Genre.GenreId GROUP BY Genre.Name ORDER BY AvgLength DESC LIMIT 5;
================================= 工具消息 =================================
名称: sql_db_query

[('Sci Fi & Fantasy', 2911783.0384615385), ('Science Fiction', 2625549.076923077), ('Drama', 2575283.78125), ('TV Shows', 2145041.0215053763), ('Comedy', 1585263.705882353)]
================================== AI消息 ==================================

平均曲目长度最长的类型是"Sci Fi & Fantasy"，平均长度约为2,911,783毫秒。其他平均曲目长度较长的类型包括"Science Fiction"、"Drama"、"TV Shows"和"Comedy"。
```

有关详细信息，请参阅[人在回路中指南](/oss/python/langgraph/interrupts)。

## 下一步

查看[评估图](/langsmith/evaluate-graph)指南，了解如何使用LangSmith评估LangGraph应用程序，包括本教程中的SQL代理。

***

<引用图标="pen-to-square" 图标类型="常规">
  [在GitHub上编辑此页面的源代码。](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/sql-agent.mdx)
</引用>

<提示图标="terminal" 图标类型="常规">
  [通过MCP以编程方式连接这些文档](/use-these-docs)，与Claude、VSCode等连接，获得实时答案。
</提示>