# 上下文概述

**上下文工程**是构建动态系统的实践，该系统以正确的格式提供正确的信息和工具，使AI应用程序能够完成任务。上下文可以沿着两个关键维度进行特征描述：

1. 按**可变性**分类：

* **静态上下文**：在执行过程中不会改变的不可变数据（例如，用户元数据、数据库连接、工具）
* **动态上下文**：随着应用程序运行而演变的可变数据（例如，对话历史、中间结果、工具调用观察结果）

2. 按**生命周期**分类：

* **运行时上下文**：限定于单次运行或调用的数据
* **跨对话上下文**：在多次对话或会话中持续存在的数据


> 运行时上下文指的是本地上下文：您的代码运行所需的数据和依赖项。它**不**指的是：> 
> 
> * LLM上下文，即传递到LLM提示中的数据。
> * "上下文窗口"，即可以传递给LLM的最大令牌数。> 
> 运行时上下文可用于优化LLM上下文。例如，您可以使用运行时上下文中的用户元数据来获取用户偏好并将其输入到上下文窗口中。


LangGraph提供了三种管理上下文的方式，它们结合了可变性和生命周期维度：

| 上下文类型                                                                 | 描述                                   | 可变性 | 生命周期           |
| -------------------------------------------------------------------------- | ------------------------------------- | ------ | ------------------ |
| [**配置**](#config-static-context)                                         | 在运行开始时传递的数据                | 静态   | 单次运行           |
| [**动态运行时上下文（状态）**](#dynamic-runtime-context-state)             | 在单次运行过程中演变的可变数据        | 动态   | 单次运行           |
| [**动态跨对话上下文（存储）**](#dynamic-cross-conversation-context-store) | 在多次对话间共享的持久性数据          | 动态   | 跨对话             |


## 配置

配置适用于不可变数据，如用户元数据或API密钥。当您拥有在运行过程中不会更改的值时，请使用此功能。

使用名为**"configurable"**的键来指定配置，该键专为此目的保留。

```typescript
await graph.invoke(
  { messages: [{ role: "user", content: "hi!" }] },
  { configurable: { user_id: "user_123" } } // [!code highlight]
);
```


## 动态运行时上下文

**动态运行时上下文**表示可以在单次运行过程中演变的可变数据，并通过LangGraph状态对象进行管理。这包括对话历史、中间结果以及从工具或LLM输出派生的值。在LangGraph中，状态对象在运行过程中充当[短期记忆](/oss/javascript/concepts/memory)。

<Tabs>
  <Tab title="在代理中">
    示例展示了如何将状态整合到代理**提示**中。

    状态也可以被代理的**工具**访问，这些工具可以根据需要读取或更新状态。详情请参见[工具调用指南](/oss/javascript/langchain/tools#short-term-memory)。

    ```typescript
    import { createAgent, createMiddleware } from "langchain";
    import type { AgentState } from "langchain";
    import * as z from "zod";

    const CustomState = z.object({ // [!code highlight]
      userName: z.string(),
    });

    const personalizedPrompt = createMiddleware({ // [!code highlight]
      name: "PersonalizedPrompt",
      stateSchema: CustomState,
      wrapModelCall: (request, handler) => {
        const userName = request.state.userName || "User";
        const systemPrompt = `You are a helpful assistant. User's name is ${userName}`;
        return handler({ ...request, systemPrompt });
      },
    });

    const agent = createAgent({  // [!code highlight]
      model: "anthropic:claude-sonnet-4-5",
      tools: [/* your tools here */],
      middleware: [personalizedPrompt] as const, // [!code highlight]
    });

    await agent.invoke({
      messages: [{ role: "user", content: "hi!" }],
      userName: "John Smith",
    });
    ```
  </Tab>
  <Tab title="在工作流中">
    ```typescript
    import type { BaseMessage } from "@langchain/core/messages";
    import { StateGraph, MessagesZodMeta, START } from "@langchain/langgraph";
    import { registry } from "@langchain/langgraph/zod";
    import * as z from "zod";

    const CustomState = z.object({  // [!code highlight]
      messages: z
        .array(z.custom<BaseMessage>())
        .register(registry, MessagesZodMeta),
      extraField: z.number(),
    });

    const builder = new StateGraph(CustomState)
      .addNode("node", async (state) => {  // [!code highlight]
        const messages = state.messages;
        // ...
        return {  // [!code highlight]
          extraField: state.extraField + 1,
        };
      })
      .addEdge(START, "node");

    const graph = builder.compile();
    ```
  </Tab>
</Tabs>


**开启记忆功能**
  请参阅[记忆指南](/oss/javascript/langgraph/add-memory)以获取有关如何启用记忆功能的更多详细信息。这是一个强大的功能，允许您在多次调用之间保持代理的状态。否则，状态仅限于单次运行。


## 动态跨对话上下文

**动态跨对话上下文**表示跨越多次对话或会话的持久性、可变数据，并通过LangGraph存储进行管理。这包括用户档案、偏好和历史交互。LangGraph存储在多次运行中充当[长期记忆](/oss/javascript/concepts/memory#long-term-memory)。这可用于读取或更新持久性事实（例如，用户档案、偏好、先前的交互）。

## 另请参阅

* [记忆概念概述](/oss/javascript/concepts/memory)
* [LangChain中的短期记忆](/oss/javascript/langchain/short-term-memory)
* [LangChain中的长期记忆](/oss/javascript/langchain/long-term-memory)
* [LangGraph中的记忆](/oss/javascript/langgraph/add-memory)

***

<Callout icon="pen-to-square" iconType="regular">
  [在GitHub上编辑此页面的源代码。](https://github.com/langchain-ai/docs/edit/main/src/oss/concepts/context.mdx)
</Callout>

<Tip icon="terminal" iconType="regular">
  [以编程方式连接这些文档](/use-these-docs)，通过MCP连接到Claude、VSCode等，以获取实时答案。
</Tip>