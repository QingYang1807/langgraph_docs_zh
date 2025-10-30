# 人机交互循环

人机交互循环（Human-in-the-Loop，HITL）中间件允许您在代理工具调用中添加人工监督功能。
当模型提出的操作可能需要审查时——例如写入文件或执行SQL——中间件可以暂停执行并等待决策。

它是通过将每个工具调用与可配置策略进行检查来实现的。如果需要干预，中间件会发出一个[interrupt](https://reference.langchain.com/python/langgraph/types/#langgraph.types.interrupt)来中止执行。图状态使用LangGraph的[persistence layer](/oss/python/langgraph/persistence)保存，因此执行可以安全暂停并在稍后恢复。

然后，人工决定确定接下来发生什么：操作可以按原样批准（`approve`），在运行前修改（`edit`），或被拒绝并提供反馈（`reject`）。

## 中断决策类型

中间件定义了三种内置方式，让人可以响应中断：

| 决策类型 | 描述 | 示例用例 |
| --- | --- | --- |
| ✅ `approve` | 操作按原样批准并执行，不做任何更改。 | 确切按照撰写的内容发送邮件草稿 |
| ✏️ `edit` | 工具调用以修改后的形式执行。 | 在发送邮件前更改收件人 |
| ❌ `reject` | 工具调用被拒绝，并添加解释到对话中。 | 拒绝邮件草稿并说明如何重写 |

每个工具可用的决策类型取决于您在`interrupt_on`中配置的策略。当多个工具调用同时暂停时，每个操作需要单独的决策。决策必须按照操作在中断请求中出现的顺序提供。

<Tip>
  在**编辑**工具参数时，请谨慎进行更改。对原始参数的重大修改可能导致模型重新评估其方法，并可能多次执行工具或采取意外操作。
</Tip>

## 配置中断

要使用HITL，在创建代理时将中间件添加到代理的`middleware`列表中。

您可以通过将工具操作映射到每个操作允许的决策类型来配置它。当中间件的工具调用匹配映射中的操作时，它将中断执行。

```python  theme={null}
from langchain.agents import create_agent
from langchain.agents.middleware import HumanInTheLoopMiddleware # [!code highlight]
from langgraph.checkpoint.memory import InMemorySaver # [!code highlight]


agent = create_agent(
    model="openai:gpt-4o",
    tools=[write_file_tool, execute_sql_tool, read_data_tool],
    middleware=[
        HumanInTheLoopMiddleware( # [!code highlight]
            interrupt_on={
                "write_file": True,  # 允许所有决策（批准、编辑、拒绝）
                "execute_sql": {"allowed_decisions": ["approve", "reject"]},  # 不允许编辑
                # 安全操作，无需批准
                "read_data": False,
            },
            # 中断消息的前缀 - 与工具名称和参数组合形成完整消息
            # 例如，"工具执行等待批准：execute_sql，查询='DELETE FROM...'"
            # 单个工具可以通过在其中断配置中指定"description"来覆盖此设置
            description_prefix="工具执行等待批准",
        ),
    ],
    # 人机交互循环需要检查点来处理中断。
    # 在生产环境中，使用持久化的检查点，如AsyncPostgresSaver。
    checkpointer=InMemorySaver(),  # [!code highlight]
)
```

<Info>
  您必须配置一个检查点器来持久化跨中断的图状态。
  在生产环境中，使用持久化的检查点器如[`AsyncPostgresSaver`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.postgres.aio.AsyncPostgresSaver)。对于测试或原型设计，使用[`InMemorySaver`](https://reference.langchain.com/python/langgraph/checkpoints/#langgraph.checkpoint.memory.InMemorySaver)。

  当调用代理时，传递一个包含**线程ID**的`config`，将执行与对话线程关联。
  有关详细信息，请参阅[LangGraph中断文档](/oss/python/langgraph/interrupts)。
</Info>

## 响应中断

当您调用代理时，它会一直运行直到完成或引发中断。当中间件的工具调用匹配您在`interrupt_on`中配置的策略时，会触发中断。在这种情况下，调用结果将包含一个`__interrupt__`字段，其中包含需要审查的操作。然后，您可以将这些操作呈现给审查人员，并在提供决策后恢复执行。

```python  theme={null}
from langgraph.types import Command

# 人机交互循环利用LangGraph的持久化层。
# 您必须提供线程ID，将执行与对话线程关联，
# 这样对话可以被暂停和恢复（正如人工审查所需）。
config = {"configurable": {"thread_id": "some_id"}} # [!code highlight]
# 运行图直到遇到中断。
result = agent.invoke(
    {
        "messages": [
            {
                "role": "user",
                "content": "从数据库中删除旧记录",
            }
        ]
    },
    config=config # [!code highlight]
)

# 中断包含完整的HITL请求，包括action_requests和review_configs
print(result['__interrupt__'])
# > [
# >    Interrupt(
# >       value={
# >          'action_requests': [
# >             {
# >                'name': 'execute_sql',
# >                'arguments': {'query': 'DELETE FROM records WHERE created_at < NOW() - INTERVAL \'30 days\';'},
# >                'description': '工具执行等待批准\n\n工具：execute_sql\n参数：{...}'
# >             }
# >          ],
# >          'review_configs': [
# >             {
# >                'action_name': 'execute_sql',
# >                'allowed_decisions': ['approve', 'reject']
# >             }
# >          ]
# >       }
# >    )
# > ]


# 使用批准决策恢复执行
agent.invoke(
    Command( # [!code highlight]
        resume={"decisions": [{"type": "approve"}]}  # 或 "edit", "reject" [!code highlight]
    ), # [!code highlight]
    config=config # 相同的线程ID以恢复暂停的对话
)
```

### 决策类型

<Tabs>
  <Tab title="✅ approve">
    使用`approve`按原样批准工具调用并执行它，不做任何更改。

    ```python  theme={null}
    agent.invoke(
        Command(
            # 决策作为列表提供，每个审查操作一个。
            # 决策的顺序必须与`__interrupt__`请求中列出的操作顺序匹配。
            resume={
                "decisions": [
                    {
                        "type": "approve",
                    }
                ]
            }
        ),
        config=config  # 相同的线程ID以恢复暂停的对话
    )
    ```
  </Tab>

  <Tab title="✏️ edit">
    使用`edit`在执行前修改工具调用。
    提供带有新工具名称和参数的编辑操作。

    ```python  theme={null}
    agent.invoke(
        Command(
            # 决策作为列表提供，每个审查操作一个。
            # 决策的顺序必须与`__interrupt__`请求中列出的操作顺序匹配。
            resume={
                "decisions": [
                    {
                        "type": "edit",
                        # 带有工具名称和参数的编辑操作
                        "edited_action": {
                            # 要调用的工具名称。
                            # 通常与原始操作相同。
                            "name": "new_tool_name",
                            # 传递给工具的参数。
                            "args": {"key1": "new_value", "key2": "original_value"},
                        }
                    }
                ]
            }
        ),
        config=config  # 相同的线程ID以恢复暂停的对话
    )
    ```

    <Tip>
      在**编辑**工具参数时，请谨慎进行更改。对原始参数的重大修改可能导致模型重新评估其方法，并可能多次执行工具或采取意外行动。
    </Tip>
  </Tab>

  <Tab title="❌ reject">
    使用`reject`拒绝工具调用并提供反馈而不是执行。

    ```python  theme={null}
    agent.invoke(
        Command(
            # 决策作为列表提供，每个审查操作一个。
            # 决策的顺序必须与`__interrupt__`请求中列出的操作顺序匹配。
            resume={
                "decisions": [
                    {
                        "type": "reject",
                        # 关于为什么拒绝操作的解释
                        "message": "不，这是错误的，因为...，而是应该这样做...",
                    }
                ]
            }
        ),
        config=config  # 相同的线程ID以恢复暂停的对话
    )
    ```

    `message`被添加到对话中作为反馈，帮助代理理解为什么操作被拒绝以及它应该怎么做。

    ***

    ### 多个决策

    当多个操作正在审查中时，按照它们在中断中出现的顺序为每个操作提供决策：

    ```python  theme={null}
    {
        "decisions": [
            {"type": "approve"},
            {
                "type": "edit",
                "edited_action": {
                    "name": "tool_name",
                    "args": {"param": "new_value"}
                }
            },
            {
                "type": "reject",
                "message": "此操作不允许"
            }
        ]
    }
    ```
  </Tab>
</Tabs>

## 执行生命周期

中间件定义了一个`after_model`钩子，它在模型生成响应之后但在任何工具调用执行之前运行：

1. 代理调用模型生成响应。
2. 中间件检查响应中的工具调用。
3. 如果任何调用需要人工输入，中间件会构建一个包含`action_requests`和`review_configs`的`HITLRequest`，并调用[interrupt](https://reference.langchain.com/python/langgraph/types/#langgraph.types.interrupt)。
4. 代理等待人工决策。
5. 基于`HITLResponse`决策，中间件执行批准或编辑过的调用，为拒绝的调用合成[ToolMessage](https://reference.langchain.com/python/langchain/messages/#langchain.messages.ToolMessage)，并恢复执行。

## 自定义HITL逻辑

对于更专业的工作流程，您可以直接使用[interrupt](https://reference.langchain.com/python/langgraph/types/#langgraph.types.interrupt)原语和[middleware](/oss/python/langchain/middleware)抽象构建自定义HITL逻辑。

回顾上面的[执行生命周期](#execution-lifecycle)，了解如何将中断集成到代理的操作中。

***

<Callout icon="pen-to-square" iconType="regular">
  [在GitHub上编辑此页面的源代码。](https://github.com/langchain-ai/docs/edit/main/src/oss/langchain/human-in-the-loop.mdx)
</Callout>

<Tip icon="terminal" iconType="regular">
  [通过MCP将这些文档编程连接到Claude、VSCode等](/use-these-docs)，以获得实时答案。
</Tip>