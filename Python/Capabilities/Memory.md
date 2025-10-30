# 记忆

AI应用程序需要[记忆](/oss/python/concepts/memory)来在多次交互间共享上下文。在LangGraph中，你可以添加两种类型的记忆：

* [添加短期记忆](#add-short-term-memory)作为代理[状态](/oss/python/langgraph/graph-api#state)的一部分，以启用多轮对话。
* [添加长期记忆](#add-long-term-memory)用于跨会话存储用户特定或应用程序级别的数据。

## 添加短期记忆

**短期**记忆（线程级别[持久化](/oss/python/langgraph/persistence)）使代理能够跟踪多轮对话。要添加短期记忆：

```python  theme={null}
from langgraph.checkpoint.memory import InMemorySaver  # [!code highlight]
from langgraph.graph import StateGraph

checkpointer = InMemorySaver()  # [!code highlight]

builder = StateGraph(...)
graph = builder.compile(checkpointer=checkpointer)  # [!code highlight]

graph.invoke(
    {"messages": [{"role": "user", "content": "hi! i am Bob"}]},
    {"configurable": {"thread_id": "1"}},  # [!code highlight]
)
```

### 在生产环境中使用

在生产环境中，使用由数据库支持的检查点器：

```python  theme={null}
from langgraph.checkpoint.postgres import PostgresSaver

DB_URI = "postgresql://postgres:postgres@localhost:5442/postgres?sslmode=disable"
with PostgresSaver.from_conn_string(DB_URI) as checkpointer:  # [!code highlight]
    builder = StateGraph(...)
    graph = builder.compile(checkpointer=checkpointer)  # [!code highlight]
```

<Accordion title="示例：使用Postgres检查点器">
  ```
  pip install -U "psycopg[binary,pool]" langgraph langgraph-checkpoint-postgres
  ```

  <Tip>
    第一次使用Postgres检查点器时，你需要调用`checkpointer.setup()`
  </Tip>

  <Tabs>
    <Tab title="同步">
      ```python  theme={null}
      from langchain.chat_models import init_chat_model
      from langgraph.graph import StateGraph, MessagesState, START
      from langgraph.checkpoint.postgres import PostgresSaver  # [!code highlight]

      model = init_chat_model(model="anthropic:claude-3-5-haiku-latest")

      DB_URI = "postgresql://postgres:postgres@localhost:5442/postgres?sslmode=disable"
      with PostgresSaver.from_conn_string(DB_URI) as checkpointer:  # [!code highlight]
          # checkpointer.setup()

          def call_model(state: MessagesState):
              response = model.invoke(state["messages"])
              return {"messages": response}

          builder = StateGraph(MessagesState)
          builder.add_node(call_model)
          builder.add_edge(START, "call_model")

          graph = builder.compile(checkpointer=checkpointer)  # [!code highlight]

          config = {
              "configurable": {
                  "thread_id": "1"  # [!code highlight]
              }
          }

          for chunk in graph.stream(
              {"messages": [{"role": "user", "content": "hi! I'm bob"}]},
              config,  # [!code highlight]
              stream_mode="values"
          ):
              chunk["messages"][-1].pretty_print()

          for chunk in graph.stream(
              {"messages": [{"role": "user", "content": "what's my name?"}]},
              config,  # [!code highlight]
              stream_mode="values"
          ):
              chunk["messages"][-1].pretty_print()
      ```
    </Tab>

    <Tab title="异步">
      ```python  theme={null}
      from langchain.chat_models import init_chat_model
      from langgraph.graph import StateGraph, MessagesState, START
      from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver  # [!code highlight]

      model = init_chat_model(model="anthropic:claude-3-5-haiku-latest")

      DB_URI = "postgresql://postgres:postgres@localhost:5442/postgres?sslmode=disable"
      async with AsyncPostgresSaver.from_conn_string(DB_URI) as checkpointer:  # [!code highlight]
          # await checkpointer.setup()

          async def call_model(state: MessagesState):
              response = await model.ainvoke(state["messages"])
              return {"messages": response}

          builder = StateGraph(MessagesState)
          builder.add_node(call_model)
          builder.add_edge(START, "call_model")

          graph = builder.compile(checkpointer=checkpointer)  # [!code highlight]

          config = {
              "configurable": {
                  "thread_id": "1"  # [!code highlight]
              }
          }

          async for chunk in graph.astream(
              {"messages": [{"role": "user", "content": "hi! I'm bob"}]},
              config,  # [!code highlight]
              stream_mode="values"
          ):
              chunk["messages"][-1].pretty_print()

          async for chunk in graph.astream(
              {"messages": [{"role": "user", "content": "what's my name?"}]},
              config,  # [!code highlight]
              stream_mode="values"
          ):
              chunk["messages"][-1].pretty_print()
      ```
    </Tab>
  </Tabs>
</Accordion>

<Accordion title="示例：使用[MongoDB](https://pypi.org/project/langgraph-checkpoint-mongodb/)检查点器">
  ```
  pip install -U pymongo langgraph langgraph-checkpoint-mongodb
  ```

  <Note>
    **设置**
    要使用MongoDB检查点器，你需要一个MongoDB集群。如果你还没有，请遵循[此指南](https://www.mongodb.com/docs/guides/atlas/cluster/)创建一个集群。
  </Note>

  <Tabs>
    <Tab title="同步">
      ```python  theme={null}
      from langchain.chat_models import init_chat_model
      from langgraph.graph import StateGraph, MessagesState, START
      from langgraph.checkpoint.mongodb import MongoDBSaver  # [!code highlight]

      model = init_chat_model(model="anthropic:claude-3-5-haiku-latest")

      DB_URI = "localhost:27017"
      with MongoDBSaver.from_conn_string(DB_URI) as checkpointer:  # [!code highlight]

          def call_model(state: MessagesState):
              response = model.invoke(state["messages"])
              return {"messages": response}

          builder = StateGraph(MessagesState)
          builder.add_node(call_model)
          builder.add_edge(START, "call_model")

          graph = builder.compile(checkpointer=checkpointer)  # [!code highlight]

          config = {
              "configurable": {
                  "thread_id": "1"  # [!code highlight]
              }
          }

          for chunk in graph.stream(
              {"messages": [{"role": "user", "content": "hi! I'm bob"}]},
              config,  # [!code highlight]
              stream_mode="values"
          ):
              chunk["messages"][-1].pretty_print()

          for chunk in graph.stream(
              {"messages": [{"role": "user", "content": "what's my name?"}]},
              config,  # [!code highlight]
              stream_mode="values"
          ):
              chunk["messages"][-1].pretty_print()
      ```
    </Tab>

    <Tab title="异步">
      ```python  theme={null}
      from langchain.chat_models import init_chat_model
      from langgraph.graph import StateGraph, MessagesState, START
      from langgraph.checkpoint.mongodb.aio import AsyncMongoDBSaver  # [!code highlight]

      model = init_chat_model(model="anthropic:claude-3-5-haiku-latest")

      DB_URI = "localhost:27017"
      async with AsyncMongoDBSaver.from_conn_string(DB_URI) as checkpointer:  # [!code highlight]

          async def call_model(state: MessagesState):
              response = await model.ainvoke(state["messages"])
              return {"messages": response}

          builder = StateGraph(MessagesState)
          builder.add_node(call_model)
          builder.add_edge(START, "call_model")

          graph = builder.compile(checkpointer=checkpointer)  # [!code highlight]

          config = {
              "configurable": {
                  "thread_id": "1"  # [!code highlight]
              }
          }

          async for chunk in graph.astream(
              {"messages": [{"role": "user", "content": "hi! I'm bob"}]},
              config,  # [!code highlight]
              stream_mode="values"
          ):
              chunk["messages"][-1].pretty_print()

          async for chunk in graph.astream(
              {"messages": [{"role": "user", "content": "what's my name?"}]},
              config,  # [!code highlight]
              stream_mode="values"
          ):
              chunk["messages"][-1].pretty_print()
      ```
    </Tab>
  </Tabs>
</Accordion>

<Accordion title="示例：使用[Redis](https://pypi.org/project/langgraph-checkpoint-redis/)检查点器">
  ```
  pip install -U langgraph langgraph-checkpoint-redis
  ```

  <Tip>
    第一次使用Redis检查点器时，你需要调用`checkpointer.setup()`
  </Tip>

  <Tabs>
    <Tab title="同步">
      ```python  theme={null}
      from langchain.chat_models import init_chat_model
      from langgraph.graph import StateGraph, MessagesState, START
      from langgraph.checkpoint.redis import RedisSaver  # [!code highlight]

      model = init_chat_model(model="anthropic:claude-3-5-haiku-latest")

      DB_URI = "redis://localhost:6379"
      with RedisSaver.from_conn_string(DB_URI) as checkpointer:  # [!code highlight]
          # checkpointer.setup()

          def call_model(state: MessagesState):
              response = model.invoke(state["messages"])
              return {"messages": response}

          builder = StateGraph(MessagesState)
          builder.add_node(call_model)
          builder.add_edge(START, "call_model")

          graph = builder.compile(checkpointer=checkpointer)  # [!code highlight]

          config = {
              "configurable": {
                  "thread_id": "1"  # [!code highlight]
              }
          }

          for chunk in graph.stream(
              {"messages": [{"role": "user", "content": "hi! I'm bob"}]},
              config,  # [!code highlight]
              stream_mode="values"
          ):
              chunk["messages"][-1].pretty_print()

          for chunk in graph.stream(
              {"messages": [{"role": "user", "content": "what's my name?"}]},
              config,  # [!code highlight]
              stream_mode="values"
          ):
              chunk["messages"][-1].pretty_print()
      ```
    </Tab>

    <Tab title="异步">
      ```python  theme={null}
      from langchain.chat_models import init_chat_model
      from langgraph.graph import StateGraph, MessagesState, START
      from langgraph.checkpoint.redis.aio import AsyncRedisSaver  # [!code highlight]

      model = init_chat_model(model="anthropic:claude-3-5-haiku-latest")

      DB_URI = "redis://localhost:6379"
      async with AsyncRedisSaver.from_conn_string(DB_URI) as checkpointer:  # [!code highlight]
          # await checkpointer.asetup()

          async def call_model(state: MessagesState):
              response = await model.ainvoke(state["messages"])
              return {"messages": response}

          builder = StateGraph(MessagesState)
          builder.add_node(call_model)
          builder.add_edge(START, "call_model")

          graph = builder.compile(checkpointer=checkpointer)  # [!code highlight]

          config = {
              "configurable": {
                  "thread_id": "1"  # [!code highlight]
              }
          }

          async for chunk in graph.astream(
              {"messages": [{"role": "user", "content": "hi! I'm bob"}]},
              config,  # [!code highlight]
              stream_mode="values"
          ):
              chunk["messages"][-1].pretty_print()

          async for chunk in graph.astream(
              {"messages": [{"role": "user", "content": "what's my name?"}]},
              config,  # [!code highlight]
              stream_mode="values"
          ):
              chunk["messages"][-1].pretty_print()
      ```
    </Tab>
  </Tabs>
</Accordion>

### 在子图中使用

如果你的图包含[子图](/oss/python/langgraph/use-subgraphs)，你只需要在编译父图时提供检查点器。LangGraph会自动将检查点器传播到子图。

```python  theme={null}
from langgraph.graph import START, StateGraph
from langgraph.checkpoint.memory import InMemorySaver
from typing import TypedDict

class State(TypedDict):
    foo: str

# 子图

def subgraph_node_1(state: State):
    return {"foo": state["foo"] + "bar"}

subgraph_builder = StateGraph(State)
subgraph_builder.add_node(subgraph_node_1)
subgraph_builder.add_edge(START, "subgraph_node_1")
subgraph = subgraph_builder.compile()  # [!code highlight]

# 父图

builder = StateGraph(State)
builder.add_node("node_1", subgraph)  # [!code highlight]
builder.add_edge(START, "node_1")

checkpointer = InMemorySaver()
graph = builder.compile(checkpointer=checkpointer)  # [!code highlight]
```

如果你希望子图有自己的记忆，你可以使用适当的检查点器选项编译它。这在[多代理](/oss/python/langchain/multi-agent)系统中很有用，如果你希望代理跟踪它们的内部消息历史。

```python  theme={null}
subgraph_builder = StateGraph(...)
subgraph = subgraph_builder.compile(checkpointer=True)  # [!code highlight]
```

## 添加长期记忆

使用长期记忆在对话间存储用户特定或应用程序特定的数据。

```python  theme={null}
from langgraph.store.memory import InMemoryStore  # [!code highlight]
from langgraph.graph import StateGraph

store = InMemoryStore()  # [!code highlight]

builder = StateGraph(...)
graph = builder.compile(store=store)  # [!code highlight]
```

### 在生产环境中使用

在生产环境中，使用由数据库支持的存储：

```python  theme={null}
from langgraph.store.postgres import PostgresStore

DB_URI = "postgresql://postgres:postgres@localhost:5442/postgres?sslmode=disable"
with PostgresStore.from_conn_string(DB_URI) as store:  # [!code highlight]
    builder = StateGraph(...)
    graph = builder.compile(store=store)  # [!code highlight]
```

<Accordion title="示例：使用Postgres存储">
  ```
  pip install -U "psycopg[binary,pool]" langgraph langgraph-checkpoint-postgres
  ```

  <Tip>
    第一次使用Postgres存储时，你需要调用`store.setup()`
  </Tip>

  <Tabs>
    <Tab title="同步">
      ```python  theme={null}
      from langchain_core.runnables import RunnableConfig
      from langchain.chat_models import init_chat_model
      from langgraph.graph import StateGraph, MessagesState, START
      from langgraph.checkpoint.postgres import PostgresSaver
      from langgraph.store.postgres import PostgresStore  # [!code highlight]
      from langgraph.store.base import BaseStore

      model = init_chat_model(model="anthropic:claude-3-5-haiku-latest")

      DB_URI = "postgresql://postgres:postgres@localhost:5442/postgres?sslmode=disable"

      with (
          PostgresStore.from_conn_string(DB_URI) as store,  # [!code highlight]
          PostgresSaver.from_conn_string(DB_URI) as checkpointer,
      ):
          # store.setup()
          # checkpointer.setup()

          def call_model(
              state: MessagesState,
              config: RunnableConfig,
              *,
              store: BaseStore,  # [!code highlight]
          ):
              user_id = config["configurable"]["user_id"]
              namespace = ("memories", user_id)
              memories = store.search(namespace, query=str(state["messages"][-1].content))  # [!code highlight]
              info = "\n".join([d.value["data"] for d in memories])
              system_msg = f"You are a helpful assistant talking to the user. User info: {info}"

              # 如果用户要求模型记住，则存储新记忆
              last_message = state["messages"][-1]
              if "remember" in last_message.content.lower():
                  memory = "User name is Bob"
                  store.put(namespace, str(uuid.uuid4()), {"data": memory})  # [!code highlight]

              response = model.invoke(
                  [{"role": "system", "content": system_msg}] + state["messages"]
              )
              return {"messages": response}

          builder = StateGraph(MessagesState)
          builder.add_node(call_model)
          builder.add_edge(START, "call_model")

          graph = builder.compile(
              checkpointer=checkpointer,
              store=store,  # [!code highlight]
          )

          config = {
              "configurable": {
                  "thread_id": "1",  # [!code highlight]
                  "user_id": "1",  # [!code highlight]
              }
          }
          for chunk in graph.stream(
              {"messages": [{"role": "user", "content": "Hi! Remember: my name is Bob"}]},
              config,  # [!code highlight]
              stream_mode="values",
          ):
              chunk["messages"][-1].pretty_print()

          config = {
              "configurable": {
                  "thread_id": "2",  # [!code highlight]
                  "user_id": "1",
              }
          }

          for chunk in graph.stream(
              {"messages": [{"role": "user", "content": "what is my name?"}]},
              config,  # [!code highlight]
              stream_mode="values",
          ):
              chunk["messages"][-1].pretty_print()
      ```
    </Tab>

    <Tab title="异步">
      ```python  theme={null}
      from langchain_core.runnables import RunnableConfig
      from langchain.chat_models import init_chat_model
      from langgraph.graph import StateGraph, MessagesState, START
      from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver
      from langgraph.store.postgres.aio import AsyncPostgresStore  # [!code highlight]
      from langgraph.store.base import BaseStore

      model = init_chat_model(model="anthropic:claude-3-5-haiku-latest")

      DB_URI = "postgresql://postgres:postgres@localhost:5442/postgres?sslmode=disable"

      async with (
          AsyncPostgresStore.from_conn_string(DB_URI) as store,  # [!code highlight]
          AsyncPostgresSaver.from_conn_string(DB_URI) as checkpointer,
      ):
          # await store.setup()
          # await checkpointer.setup()

          async def call_model(
              state: MessagesState,
              config: RunnableConfig,
              *,
              store: BaseStore,  # [!code highlight]
          ):
              user_id = config["configurable"]["user_id"]
              namespace = ("memories", user_id)
              memories = await store.asearch(namespace, query=str(state["messages"][-1].content))  # [!code highlight]
              info = "\n".join([d.value["data"] for d in memories])
              system_msg = f"You are a helpful assistant talking to the user. User info: {info}"

              # 如果用户要求模型记住，则存储新记忆
              last_message = state["messages"][-1]
              if "remember" in last_message.content.lower():
                  memory = "User name is Bob"
                  await store.aput(namespace, str(uuid.uuid4()), {"data": memory})  # [!code highlight]

              response = await model.ainvoke(
                  [{"role": "system", "content": system_msg}] + state["messages"]
              )
              return {"messages": response}

          builder = StateGraph(MessagesState)
          builder.add_node(call_model)
          builder.add_edge(START, "call_model")

          graph = builder.compile(
              checkpointer=checkpointer,
              store=store,  # [!code highlight]
          )

          config = {
              "configurable": {
                  "thread_id": "1",  # [!code highlight]
                  "user_id": "1",  # [!code highlight]
              }
          }
          async for chunk in graph.astream(
              {"messages": [{"role": "user", "content": "Hi! Remember: my name is Bob"}]},
              config,  # [!code highlight]
              stream_mode="values",
          ):
              chunk["messages"][-1].pretty_print()

          config = {
              "configurable": {
                  "thread_id": "2",  # [!code highlight]
                  "user_id": "1",
              }
          }

          async for chunk in graph.astream(
              {"messages": [{"role": "user", "content": "what is my name?"}]},
              config,  # [!code highlight]
              stream_mode="values",
          ):
              chunk["messages"][-1].pretty_print()
      ```
    </Tab>
  </Tabs>
</Accordion>

<Accordion title="示例：使用[Redis](https://pypi.org/project/langgraph-checkpoint-redis/)存储">
  ```
  pip install -U langgraph langgraph-checkpoint-redis
  ```

  <Tip>
    第一次使用Redis存储时，你需要调用`store.setup()`
  </Tip>

  <Tabs>
    <Tab title="同步">
      ```python  theme={null}
      from langchain_core.runnables import RunnableConfig
      from langchain.chat_models import init_chat_model
      from langgraph.graph import StateGraph, MessagesState, START
      from langgraph.checkpoint.redis import RedisSaver
      from langgraph.store.redis import RedisStore  # [!code highlight]
      from langgraph.store.base import BaseStore

      model = init_chat_model(model="anthropic:claude-3-5-haiku-latest")

      DB_URI = "redis://localhost:6379"

      with (
          RedisStore.from_conn_string(DB_URI) as store,  # [!code highlight]
          RedisSaver.from_conn_string(DB_URI) as checkpointer,
      ):
          store.setup()
          checkpointer.setup()

          def call_model(
              state: MessagesState,
              config: RunnableConfig,
              *,
              store: BaseStore,  # [!code highlight]
          ):
              user_id = config["configurable"]["user_id"]
              namespace = ("memories", user_id)
              memories = store.search(namespace, query=str(state["messages"][-1].content))  # [!code highlight]
              info = "\n".join([d.value["data"] for d in memories])
              system_msg = f"You are a helpful assistant talking to the user. User info: {info}"

              # 如果用户要求模型记住，则存储新记忆
              last_message = state["messages"][-1]
              if "remember" in last_message.content.lower():
                  memory = "User name is Bob"
                  store.put(namespace, str(uuid.uuid4()), {"data": memory})  # [!code highlight]

              response = model.invoke(
                  [{"role": "system", "content": system_msg}] + state["messages"]
              )
              return {"messages": response}

          builder = StateGraph(MessagesState)
          builder.add_node(call_model)
          builder.add_edge(START, "call_model")

          graph = builder.compile(
              checkpointer=checkpointer,
              store=store,  # [!code highlight]
          )

          config = {
              "configurable": {
                  "thread_id": "1",  # [!code highlight]
                  "user_id": "1",  # [!code highlight]
              }
          }
          for chunk in graph.stream(
              {"messages": [{"role": "user", "content": "Hi! Remember: my name is Bob"}]},
              config,  # [!code highlight]
              stream_mode="values",
          ):
              chunk["messages"][-1].pretty_print()

          config = {
              "configurable": {
                  "thread_id": "2",  # [!code highlight]
                  "user_id": "1",
              }
          }

          for chunk in graph.stream(
              {"messages": [{"role": "user", "content": "what is my name?"}]},
              config,  # [!code highlight]
              stream_mode="values",
          ):
              chunk["messages"][-1].pretty_print()
      ```
    </Tab>

    <Tab title="异步">
      ```python  theme={null}
      from langchain_core.runnables import RunnableConfig
      from langchain.chat_models import init_chat_model
      from langgraph.graph import StateGraph, MessagesState, START
      from langgraph.checkpoint.redis.aio import AsyncRedisSaver
      from langgraph.store.redis.aio import AsyncRedisStore  # [!code highlight]
      from langgraph.store.base import BaseStore

      model = init_chat_model(model="anthropic:claude-3-5-haiku-latest")

      DB_URI = "redis://localhost:6379"

      async with (
          AsyncRedisStore.from_conn_string(DB_URI) as store,  # [!code highlight]
          AsyncRedisSaver.from_conn_string(DB_URI) as checkpointer,
      ):
          # await store.setup()
          # await checkpointer.asetup()

          async def call_model(
              state: MessagesState,
              config: RunnableConfig,
              *,
              store: BaseStore,  # [!code highlight]
          ):
              user_id = config["configurable"]["user_id"]
              namespace = ("memories", user_id)
              memories = await store.asearch(namespace, query=str(state["messages"][-1].content))  # [!code highlight]
              info = "\n".join([d.value["data"] for d in memories])
              system_msg = f"You are a helpful assistant talking to the user. User info: {info}"

              # 如果用户要求模型记住，则存储新记忆
              last_message = state["messages"][-1]
              if "remember" in last_message.content.lower():
                  memory = "User name is Bob"
                  await store.aput(namespace, str(uuid.uuid4()), {"data": memory})  # [!code highlight]

              response = await model.ainvoke(
                  [{"role": "system", "content": system_msg}] + state["messages"]
              )
              return {"messages": response}

          builder = StateGraph(MessagesState)
          builder.add_node(call_model)
          builder.add_edge(START, "call_model")

          graph = builder.compile(
              checkpointer=checkpointer,
              store=store,  # [!code highlight]
          )

          config = {
              "configurable": {
                  "thread_id": "1",  # [!code highlight]
                  "user_id": "1",  # [!code highlight]
              }
          }
          async for chunk in graph.astream(
              {"messages": [{"role": "user", "content": "Hi! Remember: my name is Bob"}]},
              config,  # [!code highlight]
              stream_mode="values",
          ):
              chunk["messages"][-1].pretty_print()

          config = {
              "configurable": {
                  "thread_id": "2",  # [!code highlight]
                  "user_id": "1",
              }
          }

          async for chunk in graph.astream(
              {"messages": [{"role": "user", "content": "what is my name?"}]},
              config,  # [!code highlight]
              stream_mode="values",
          ):
              chunk["messages"][-1].pretty_print()
      ```
    </Tab>
  </Tabs>
</Accordion>

### 使用语义搜索

在图的记忆存储中启用语义搜索，让图的代理能够通过语义相似度搜索存储中的项目。

```python  theme={null}
from langchain.embeddings import init_embeddings
from langgraph.store.memory import InMemoryStore

# 创建启用语义搜索的存储
embeddings = init_embeddings("openai:text-embedding-3-small")
store = InMemoryStore(
    index={
        "embed": embeddings,
        "dims": 1536,
    }
)

store.put(("user_123", "memories"), "1", {"text": "I love pizza"})
store.put(("user_123", "memories"), "2", {"text": "I am a plumber"})

items = store.search(
    ("user_123", "memories"), query="I'm hungry", limit=1
)
```

<Accordion title="具有语义搜索的长期记忆">
  ```python  theme={null}

  from langchain.embeddings import init_embeddings
  from langchain.chat_models import init_chat_model
  from langgraph.store.base import BaseStore
  from langgraph.store.memory import InMemoryStore
  from langgraph.graph import START, MessagesState, StateGraph

  model = init_chat_model("openai:gpt-4o-mini")

  # 创建启用语义搜索的存储
  embeddings = init_embeddings("openai:text-embedding-3-small")
  store = InMemoryStore(
      index={
          "embed": embeddings,
          "dims": 1536,
      }
  )

  store.put(("user_123", "memories"), "1", {"text": "I love pizza"})
  store.put(("user_123", "memories"), "2", {"text": "I am a plumber"})

  def chat(state, *, store: BaseStore):
      # 根据用户的最后一条消息进行搜索
      items = store.search(
          ("user_123", "memories"), query=state["messages"][-1].content, limit=2
      )
      memories = "\n".join(item.value["text"] for item in items)
      memories = f"## Memories of user\n{memories}" if memories else ""
      response = model.invoke(
          [
              {"role": "system", "content": f"You are a helpful assistant.\n{memories}"},
              *state["messages"],
          ]
      )
      return {"messages": [response]}


  builder = StateGraph(MessagesState)
  builder.add_node(chat)
  builder.add_edge(START, "chat")
  graph = builder.compile(store=store)

  for message, metadata in graph.stream(
      input={"messages": [{"role": "user", "content": "I'm hungry"}]},
      stream_mode="messages",
  ):
      print(message.content, end="")
  ```
</Accordion>

## 管理短期记忆

启用[短期记忆](#add-short-term-memory)后，长对话可能会超过LLM的上下文窗口。常见的解决方案包括：

* [修剪消息](#trim-messages)：删除前N条或后N条消息（在调用LLM之前）
* [删除消息](#delete-messages)：从LangGraph状态中永久删除
* [总结消息](#summarize-messages)：总结历史记录中的早期消息并用总结替换它们
* [管理检查点](#manage-checkpoints)来存储和检索消息历史
* 自定义策略（例如消息过滤等）

这使代理能够在不超出LLM上下文窗口的情况下跟踪对话。

### 修剪消息

大多数LLM都有支持的最大上下文窗口（以令牌为单位）。决定何时截断消息的一种方法是计算消息历史中的令牌数，并在接近该限制时进行截断。如果你使用LangChain，可以使用修剪消息实用程序并指定要保留的令牌数量以及处理边界的`策略`（例如保留最后的`max_tokens`）。

要修剪消息历史，请使用[`trim_messages`](https://python.langchain.com/api_reference/core/messages/langchain_core.messages.utils.trim_messages.html)函数：

```python  theme={null}
from langchain_core.messages.utils import (  # [!code highlight]
    trim_messages,  # [!code highlight]
    count_tokens_approximately  # [!code highlight]
)  # [!code highlight]

def call_model(state: MessagesState):
    messages = trim_messages(  # [!code highlight]
        state["messages"],
        strategy="last",
        token_counter=count_tokens_approximately,
        max_tokens=128,
        start_on="human",
        end_on=("human", "tool"),
    )
    response = model.invoke(messages)
    return {"messages": [response]}

builder = StateGraph(MessagesState)
builder.add_node(call_model)
...
```

<Accordion title="完整示例：修剪消息">
  ```python  theme={null}
  from langchain_core.messages.utils import (
      trim_messages,  # [!code highlight]
      count_tokens_approximately  # [!code highlight]
  )
  from langchain.chat_models import init_chat_model
  from langgraph.graph import StateGraph, START, MessagesState

  model = init_chat_model("anthropic:claude-sonnet-4-5")
  summarization_model = model.bind(max_tokens=128)

  def call_model(state: MessagesState):
      messages = trim_messages(  # [!code highlight]
          state["messages"],
          strategy="last",
          token_counter=count_tokens_approximately,
          max_tokens=128,
          start_on="human",
          end_on=("human", "tool"),
      )
      response = model.invoke(messages)
      return {"messages": [response]}

  checkpointer = InMemorySaver()
  builder = StateGraph(MessagesState)
  builder.add_node(call_model)
  builder.add_edge(START, "call_model")
  graph = builder.compile(checkpointer=checkpointer)

  config = {"configurable": {"thread_id": "1"}}
  graph.invoke({"messages": "hi, my name is bob"}, config)
  graph.invoke({"messages": "write a short poem about cats"}, config)
  graph.invoke({"messages": "now do the same but for dogs"}, config)
  final_response = graph.invoke({"messages": "what's my name?"}, config)

  final_response["messages"][-1].pretty_print()
  ```

  ```
  ================================== Ai Message ==================================

  Your name is Bob, as you mentioned when you first introduced yourself.
  ```
</Accordion>

### 删除消息

你可以从图状态中删除消息来管理消息历史。当你想删除特定消息或清除整个消息历史时，这很有用。

要从图状态中删除消息，你可以使用`RemoveMessage`。为了使`RemoveMessage`工作，你需要使用带有[`add_messages`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.message.add_messages)[reducer](/oss/python/langgraph/graph-api#reducers)的状态键，如[`MessagesState`](/oss/python/langgraph/graph-api#messagesstate)。

要删除特定消息：

```python  theme={null}
from langchain.messages import RemoveMessage  # [!code highlight]

def delete_messages(state):
    messages = state["messages"]
    if len(messages) > 2:
        # 删除最早的两条消息
        return {"messages": [RemoveMessage(id=m.id) for m in messages[:2]]}  # [!code highlight]
```

要删除**所有**消息：

```python  theme={null}
from langgraph.graph.message import REMOVE_ALL_MESSAGES  # [!code highlight]

def delete_messages(state):
    return {"messages": [RemoveMessage(id=REMOVE_ALL_MESSAGES)]}  # [!code highlight]
```

<Warning>
  删除消息时，**请确保**结果消息历史是有效的。检查你使用的LLM提供商的限制。例如：

  * 一些提供商期望消息历史以`user`消息开头
  * 大多数提供商要求带有工具调用的`assistant`消息后跟相应的`tool`结果消息。
</Warning>

<Accordion title="完整示例：删除消息">
  ```python  theme={null}
  from langchain.messages import RemoveMessage  # [!code highlight]

  def delete_messages(state):
      messages = state["messages"]
      if len(messages) > 2:
          # 删除最早的两条消息
          return {"messages": [RemoveMessage(id=m.id) for m in messages[:2]]}  # [!code highlight]

  def call_model(state: MessagesState):
      response = model.invoke(state["messages"])
      return {"messages": response}

  builder = StateGraph(MessagesState)
  builder.add_sequence([call_model, delete_messages])
  builder.add_edge(START, "call_model")

  checkpointer = InMemorySaver()
  app = builder.compile(checkpointer=checkpointer)

  for event in app.stream(
      {"messages": [{"role": "user", "content": "hi! I'm bob"}]},
      config,
      stream_mode="values"
  ):
      print([(message.type, message.content) for message in event["messages"]])

  for event in app.stream(
      {"messages": [{"role": "user", "content": "what's my name?"}]},
      config,
      stream_mode="values"
  ):
      print([(message.type, message.content) for message in event["messages"]])
  ```

  ```
  [('human', "hi! I'm bob")]
  [('human', "hi! I'm bob"), ('ai', 'Hi Bob! How are you doing today? Is there anything I can help you with?')]
  [('human', "hi! I'm bob"), ('ai', 'Hi Bob! How are you doing today? Is there anything I can help you with?'), ('human', "what's my name?")]
  [('human', "hi! I'm bob"), ('ai', 'Hi Bob! How are you doing today? Is there anything I can help you with?'), ('human', "what's my name?"), ('ai', 'Your name is Bob.')]
  [('human', "what's my name?"), ('ai', 'Your name is Bob.')]
  ```
</Accordion>

### 总结消息

如上所示的修剪或删除消息的问题是你可能会因为消息队列的剔除而丢失信息。因此，一些应用程序受益于更复杂的方法，即使用聊天模型总结消息历史。

<img src="https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/summary.png?fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=c8ed3facdccd4ef5c7e52902c72ba938" alt="" data-og-width="609" width="609" data-og-height="242" height="242" data-path="oss/images/summary.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/summary.png?w=280&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=4208b9b0cc9f459f3dc4e5219918471b 280w, https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/summary.png?w=560&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=7acb77c081545f57042368f4e9d0c8cb 560w, https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/summary.png?w=840&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=2fcfdb0c481d2e1d361e76db763a41e5 840w, https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/summary.png?w=1100&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=4abdac693a562788aa0db8681bef8ea7 1100w, https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/summary.png?w=1650&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=40acfefa91dcb11b247a6e4a7705f22b 1650w, https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/summary.png?w=2500&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=8d765aaf7551e8b0fc2720de7d2ac2a8 2500w" />

可以使用提示和编排逻辑来总结消息历史。例如，在LangGraph中，你可以扩展[`MessagesState`](/oss/python/langgraph/graph-api#working-with-messages-in-graph-state)以包含`summary`键：

```python  theme={null}
from langgraph.graph import MessagesState
class State(MessagesState):
    summary: str
```

然后，你可以生成聊天历史的总结，使用任何现有的总结作为下一个总结的上下文。这个`summarize_conversation`节点可以在`messages`状态键中积累了一定数量的消息后调用。

```python  theme={null}
def summarize_conversation(state: State):

    # 首先，我们获取任何现有的总结
    summary = state.get("summary", "")

    # 创建我们的总结提示
    if summary:

        # 总结已经存在
        summary_message = (
            f"This is a summary of the conversation to date: {summary}\n\n"
            "Extend the summary by taking into account the new messages above:"
        )

    else:
        summary_message = "Create a summary of the conversation above:"

    # 将提示添加到我们的历史中
    messages = state["messages"] + [HumanMessage(content=summary_message)]
    response = model.invoke(messages)

    # 删除除最近2条消息外的所有消息
    delete_messages = [RemoveMessage(id=m.id) for m in state["messages"][:-2]]
    return {"summary": response.content, "messages": delete_messages}
```

<Accordion title="完整示例：总结消息">
  ```python  theme={null}
  from typing import Any, TypedDict

  from langchain.chat_models import init_chat_model
  from langchain.messages import AnyMessage
  from langchain_core.messages.utils import count_tokens_approximately
  from langgraph.graph import StateGraph, START, MessagesState
  from langgraph.checkpoint.memory import InMemorySaver
  from langmem.short_term import SummarizationNode, RunningSummary  # [!code highlight]

  model = init_chat_model("anthropic:claude-sonnet-4-5")
  summarization_model = model.bind(max_tokens=128)

  class State(MessagesState):
      context: dict[str, RunningSummary]  # [!code highlight]

  class LLMInputState(TypedDict):  # [!code highlight]
      summarized_messages: list[AnyMessage]
      context: dict[str, RunningSummary]

  summarization_node = SummarizationNode(  # [!code highlight]
      token_counter=count_tokens_approximately,
      model=summarization_model,
      max_tokens=256,
      max_tokens_before_summary=256,
      max_summary_tokens=128,
  )

  def call_model(state: LLMInputState):  # [!code highlight]
      response = model.invoke(state["summarized_messages"])
      return {"messages": [response]}

  checkpointer = InMemorySaver()
  builder = StateGraph(State)
  builder.add_node(call_model)
  builder.add_node("summarize", summarization_node)  # [!code highlight]
  builder.add_edge(START, "summarize")
  builder.add_edge("summarize", "call_model")
  graph = builder.compile(checkpointer=checkpointer)

  # 调用图
  config = {"configurable": {"thread_id": "1"}}
  graph.invoke({"messages": "hi, my name is bob"}, config)
  graph.invoke({"messages": "write a short poem about cats"}, config)
  graph.invoke({"messages": "now do the same but for dogs"}, config)
  final_response = graph.invoke({"messages": "what's my name?"}, config)

  final_response["messages"][-1].pretty_print()
  print("\nSummary:", final_response["context"]["running_summary"].summary)
  ```

  1. 我们将在`context`字段中跟踪我们的运行总结

  （`SummarizationNode`所期望的）。

  1. 定义仅用于过滤的私有状态

  `call_model`节点的输入。

  1. 我们在这里传递私有输入状态以隔离总结节点返回的消息

  ```
  ================================== Ai Message ==================================

  From our conversation, I can see that you introduced yourself as Bob. That's the name you shared with me when we began talking.

  Summary: In this conversation, I was introduced to Bob, who then asked me to write a poem about cats. I composed a poem titled "The Mystery of Cats" that captured cats' graceful movements, independent nature, and their special relationship with humans. Bob then requested a similar poem about dogs, so I wrote "The Joy of Dogs," which highlighted dogs' loyalty, enthusiasm, and loving companionship. Both poems were written in a similar style but emphasized the distinct characteristics that make each pet special.
  ```
</Accordion>

### 管理检查点

你可以查看和删除检查点器存储的信息。

<a id="checkpoint" />

#### 查看线程状态

<Tabs>
  <Tab title="图/功能API">
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
  </Tab>
  <Tab title="检查点器API">
    ```python  theme={null}
    config = {
        "configurable": {
            "thread_id": "1",  # [!code highlight]
            # 可选地为特定检查点提供ID，
            # 否则显示最新的检查点
            # "checkpoint_id": "1f029ca3-1f5b-6704-8004-820c16b69a5a"  # [!code highlight]

        }
    }
    checkpointer.get_tuple(config)  # [!code highlight]
    ```

    ```
    CheckpointTuple(
        config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1f5b-6704-8004-820c16b69a5a'}},
        checkpoint={
            'v': 3,
            'ts': '2025-05-05T16:01:24.680462+00:00',
            'id': '1f029ca3-1f5b-6704-8004-820c16b69a5a',
            'channel_versions': {'__start__': '00000000000000000000000000000005.0.5290678567601859', 'messages': '00000000000000000000000000000006.0.3205149138784782', 'branch:to:call_model': '00000000000000000000000000000006.0.14611156755133758'}, 'versions_seen': {'__input__': {}, '__start__': {'__start__': '00000000000000000000000000000004.0.5736472536395331'}, 'call_model': {'branch:to:call_model': '00000000000000000000000000000005.0.1410174088651449'}},
            'channel_values': {'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today?), HumanMessage(content="what's my name?"), AIMessage(content='Your name is Bob.')]},
        },
        metadata={
            'source': 'loop',
            'writes': {'call_model': {'messages': AIMessage(content='Your name is Bob.')}},
            'step': 4,
            'parents': {},
            'thread_id': '1'
        },
        parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1790-6b0a-8003-baf965b6a38f'}},
        pending_writes=[]
    )
    ```
  </Tab>
</Tabs>

<a id="checkpoints" />

#### 查看线程的历史

<Tabs>
  <Tab title="图/功能API">
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
  </Tab>
  <Tab title="检查点器API">
    ```python  theme={null}
    config = {
        "configurable": {
            "thread_id": "1"  # [!code highlight]
        }
    }
    list(checkpointer.list(config))  # [!code highlight]
    ```

    ```
    [
        CheckpointTuple(
            config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1f5b-6704-8004-820c16b69a5a'}},
            checkpoint={
                'v': 3,
                'ts': '2025-05-05T16:01:24.680462+00:00',
                'id': '1f029ca3-1f5b-6704-8004-820c16b69a5a',
                'channel_versions': {'__start__': '00000000000000000000000000000005.0.5290678567601859', 'messages': '00000000000000000000000000000006.0.3205149138784782', 'branch:to:call_model': '00000000000000000000000000000006.0.14611156755133758'},
                'versions_seen': {'__input__': {}, '__start__': {'__start__': '00000000000000000000000000000004.0.5736472536395331'}, 'call_model': {'branch:to:call_model': '00000000000000000000000000000005.0.1410174088651449'}},
                'channel_values': {'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?'), HumanMessage(content="what's my name?"), AIMessage(content='Your name is Bob.')]},
            },
            metadata={'source': 'loop', 'writes': {'call_model': {'messages': AIMessage(content='Your name is Bob.')}}, 'step': 4, 'parents': {}, 'thread_id': '1'},
            parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1790-6b0a-8003-baf965b6a38f'}},
            pending_writes=[]
        ),
        CheckpointTuple(
            config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1790-6b0a-8003-baf965b6a38f'}},
            checkpoint={
                'v': 3,
                'ts': '2025-05-05T16:01:23.863421+00:00',
                'id': '1f029ca3-1790-6b0a-8003-baf965b6a38f',
                'channel_versions': {'__start__': '00000000000000000000000000000005.0.5290678567601859', 'messages': '00000000000000000000000000000006.0.3205149138784782', 'branch:to:call_model': '00000000000000000000000000000006.0.14611156755133758'},
                'versions_seen': {'__input__': {}, '__start__': {'__start__': '00000000000000000000000000000004.0.5736472536395331'}, 'call_model': {'branch:to:call_model': '00000000000000000000000000000005.0.1410174088651449'}},
                'channel_values': {'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?'), HumanMessage(content="what's my name?")], 'branch:to:call_model': None}
            },
            metadata={'source': 'loop', 'writes': None, 'step': 3, 'parents': {}, 'thread_id': '1'},
            parent_config={...},
            pending_writes=[('8ab4155e-6b15-b885-9ce5-bed69a2c305c', 'messages', AIMessage(content='Your name is Bob.'))]
        ),
        CheckpointTuple(
            config={...},
            checkpoint={
                'v': 3,
                'ts': '2025-05-05T16:01:23.863173+00:00',
                'id': '1f029ca3-1790-616e-8002-9e021694a0cd',
                'channel_versions': {'__start__': '00000000000000000000000000000004.0.5736472536395331', 'messages': '00000000000000000000000000000003.0.7056767754077798', 'branch:to:call_model': '00000000000000000000000000000003.0.22059023329132854'},
                'versions_seen': {'__input__': {}, '__start__': {'__start__': '00000000000000000000000000000001.0.7040775356287469'}, 'call_model': {'branch:to:call_model': '00000000000000000000000000000002.0.9300422176788571'}},
                'channel_values': {'__start__': {'messages': [{'role': 'user', 'content': "what's my name?"}]}, 'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?')]}
            },
            metadata={'source': 'input', 'writes': {'__start__': {'messages': [{'role': 'user', 'content': "what's my name?"}]}}, 'step': 2, 'parents': {}, 'thread_id': '1'},
            parent_config={...},
            pending_writes=[('24ba39d6-6db1-4c9b-f4c5-682aeaf38dcd', 'messages', [{'role': 'user', 'content': "what's my name?"}]), ('24ba39d6-6db1-4c9b-f4c5-682aeaf38dcd', 'branch:to:call_model', None)]
        ),
        CheckpointTuple(
            config={...},
            checkpoint={
                'v': 3,
                'ts': '2025-05-05T16:01:23.862295+00:00',
                'id': '1f029ca3-178d-6f54-8001-d7b180db0c89',
                'channel_versions': {'__start__': '00000000000000000000000000000002.0.18673090920108737', 'messages': '00000000000000000000000000000003.0.7056767754077798', 'branch:to:call_model': '00000000000000000000000000000003.0.22059023329132854'},
                'versions_seen': {'__input__': {}, '__start__': {'__start__': '00000000000000000000000000000001.0.7040775356287469'}, 'call_model': {'branch:to:call_model': '00000000000000000000000000000002.0.9300422176788571'}},
                'channel_values': {'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?')]}
            },
            metadata={'source': 'loop', 'writes': {'call_model': {'messages': AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?')}}, 'step': 1, 'parents': {}, 'thread_id': '1'},
            parent_config={...},
            pending_writes=[]
        ),
        CheckpointTuple(
            config={...},
            checkpoint={
                'v': 3,
                'ts': '2025-05-05T16:01:22.278960+00:00',
                'id': '1f029ca3-0874-6612-8000-339f2abc83b1',
                'channel_versions': {'__start__': '00000000000000000000000000000002.0.18673090920108737', 'messages': '00000000000000000000000000000002.0.30296526818059655', 'branch:to:call_model': '00000000000000000000000000000002.0.9300422176788571'},
                'versions_seen': {'__input__': {}, '__start__': {'__start__': '00000000000000000000000000000001.0.7040775356287469'}},
                'channel_values': {'messages': [HumanMessage(content="hi! I'm bob")], 'branch:to:call_model': None}
            },
            metadata={'source': 'loop', 'writes': None, 'step': 0, 'parents': {}, 'thread_id': '1'},
            parent_config={...},
            pending_writes=[('8cbd75e0-3720-b056-04f7-71ac805140a0', 'messages', AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?'))]
        ),
        CheckpointTuple(
            config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-0870-6ce2-bfff-1f3f14c3e565'}},
            checkpoint={
                'v': 3,
                'ts': '2025-05-05T16:01:22.277497+00:00',
                'id': '1f029ca3-0870-6ce2-bfff-1f3f14c3e565',
                'channel_versions': {'__start__': '00000000000000000000000000000001.0.7040775356287469'},
                'versions_seen': {'__input__': {}},
                'channel_values': {'__start__': {'messages': [{'role': 'user', 'content': "hi! I'm bob"}]}}
            },
            metadata={'source': 'input', 'writes': {'__start__': {'messages': [{'role': 'user', 'content': "hi! I'm bob"}]}}, 'step': -1, 'parents': {}, 'thread_id': '1'},
            parent_config=None,
            pending_writes=[('d458367b-8265-812c-18e2-33001d199ce6', 'messages', [{'role': 'user', 'content': "hi! I'm bob"}]), ('d458367b-8265-812c-18e2-33001d199ce6', 'branch:to:call_model', None)]
        )
    ]
    ```
  </Tab>
</Tabs>

#### 删除线程的所有检查点

```python  theme={null}
thread_id = "1"
checkpointer.delete_thread(thread_id)
```

## 预构建的记忆工具

**LangMem**是一个由LangChain维护的库，提供用于在代理中管理长期记忆的工具。有关使用示例，请参见[LangMem文档](https://langchain-ai.github.io/langmem/)。

***

<Callout icon="pen-to-square" iconType="regular">
  [在GitHub上编辑此页面的源代码。](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/add-memory.mdx)
</Callout>

<Tip icon="terminal" iconType="regular">
  [通过MCP以编程方式连接这些文档](/use-these-docs)到Claude、VSCode等，以获取实时答案。
</Tip>