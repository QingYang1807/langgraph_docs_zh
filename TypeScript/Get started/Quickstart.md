# 快速入门

本快速入门演示了如何使用 LangGraph Graph API 或 Functional API 构建一个计算器智能体。

* 如果您倾向于将智能体定义为节点和边的图，请[使用 Graph API](#use-the-graph-api)。
* 如果您倾向于将智能体定义为单个函数，请[使用 Functional API](#use-the-functional-api)。

<Tip>
  有关概念性信息，请参阅 [Graph API 概述](/oss/javascript/langgraph/graph-api) 和 [Functional API 概述](/oss/javascript/langgraph/functional-api)。
</Tip>

<Info>
  对于此示例，您需要设置一个 [Claude (Anthropic)](https://www.anthropic.com/) 账户并获取 API 密钥。然后，在您的终端中设置 `ANTHROPIC_API_KEY` 环境变量。
</Info>

<Tabs>
  <Tab title="使用 Graph API">
    ## 1. 定义工具和模型

    在本示例中，我们将使用 Claude Sonnet 4.5 模型，并定义用于加法、乘法和除法的工具。

    ```typescript  theme={null}
    import { ChatAnthropic } from "@langchain/anthropic";
    import { tool } from "@langchain/core/tools";
    import * as z from "zod";

    const model = new ChatAnthropic({
      model: "claude-sonnet-4-5",
      temperature: 0,
    });

    // 定义工具
    const add = tool(({ a, b }) => a + b, {
      name: "add",
      description: "将两个数字相加",
      schema: z.object({
        a: z.number().describe("第一个数字"),
        b: z.number().describe("第二个数字"),
      }),
    });

    const multiply = tool(({ a, b }) => a * b, {
      name: "multiply",
      description: "将两个数字相乘",
      schema: z.object({
        a: z.number().describe("第一个数字"),
        b: z.number().describe("第二个数字"),
      }),
    });

    const divide = tool(({ a, b }) => a / b, {
      name: "divide",
      description: "将两个数字相除",
      schema: z.object({
        a: z.number().describe("第一个数字"),
        b: z.number().describe("第二个数字"),
      }),
    });

    // 使用工具增强 LLM
    const toolsByName = {
      [add.name]: add,
      [multiply.name]: multiply,
      [divide.name]: divide,
    };
    const tools = Object.values(toolsByName);
    const modelWithTools = model.bindTools(tools);
    ```

    ## 2. 定义状态

    图的状态用于存储消息和 LLM 调用次数。

    <Tip>
      LangGraph 中的状态在智能体执行过程中持续存在。

      带有 `operator.add` 的 `Annotated` 类型确保新消息会附加到现有列表，而不是替换它。
    </Tip>

    ```typescript  theme={null}
    import { StateGraph, START, END } from "@langchain/langgraph";
    import { MessagesZodMeta } from "@langchain/langgraph";
    import { registry } from "@langchain/langgraph/zod";
    import { type BaseMessage } from "@langchain/core/messages";

    const MessagesState = z.object({
      messages: z
        .array(z.custom<BaseMessage>())
        .register(registry, MessagesZodMeta),
      llmCalls: z.number().optional(),
    });
    ```

    ## 3. 定义模型节点

    模型节点用于调用 LLM 并决定是否调用工具。

    ```typescript  theme={null}
    import { SystemMessage } from "@langchain/core/messages";
    async function llmCall(state: z.infer<typeof MessagesState>) {
      return {
        messages: await modelWithTools.invoke([
          new SystemMessage(
            "你是一个有帮助的助手，负责对一组输入执行算术运算。"
          ),
          ...state.messages,
        ]),
        llmCalls: (state.llmCalls ?? 0) + 1,
      };
    }
    ```

    ## 4. 定义工具节点

    工具节点用于调用工具并返回结果。

    ```typescript  theme={null}
    import { isAIMessage, ToolMessage } from "@langchain/core/messages";
    async function toolNode(state: z.infer<typeof MessagesState>) {
      const lastMessage = state.messages.at(-1);

      if (lastMessage == null || !isAIMessage(lastMessage)) {
        return { messages: [] };
      }

      const result: ToolMessage[] = [];
      for (const toolCall of lastMessage.tool_calls ?? []) {
        const tool = toolsByName[toolCall.name];
        const observation = await tool.invoke(toolCall);
        result.push(observation);
      }

      return { messages: result };
    }
    ```

    ## 5. 定义结束逻辑

    条件边函数用于根据 LLM 是否进行了工具调用来路由到工具节点或结束。

    ```typescript  theme={null}
    async function shouldContinue(state: z.infer<typeof MessagesState>) {
      const lastMessage = state.messages.at(-1);
      if (lastMessage == null || !isAIMessage(lastMessage)) return END;

      // 如果 LLM 进行了工具调用，则执行一个操作
      if (lastMessage.tool_calls?.length) {
        return "toolNode";
      }

      // 否则，我们停止（回复用户）
      return END;
    }
    ```

    ## 6. 构建和编译智能体

    智能体使用 [`StateGraph`](https://langchain-ai.github.io/langgraphjs/reference/classes/langgraph.StateGraph.html) 类构建，并使用 @\[`compile`]\[StateGraph.compile] 方法编译。

    ```typescript  theme={null}
    const agent = new StateGraph(MessagesState)
      .addNode("llmCall", llmCall)
      .addNode("toolNode", toolNode)
      .addEdge(START, "llmCall")
      .addConditionalEdges("llmCall", shouldContinue, ["toolNode", END])
      .addEdge("toolNode", "llmCall")
      .compile();

    // 调用
    import { HumanMessage } from "@langchain/core/messages";
    const result = await agent.invoke({
      messages: [new HumanMessage("将 3 和 4 相加。")],
    });

    for (const message of result.messages) {
      console.log(`[${message.getType()}]: ${message.text}`);
    }
    ```

    恭喜！您已使用 LangGraph Graph API 构建了您的第一个智能体。

    <Accordion title="完整代码示例">
      ```typescript  theme={null}
      // 步骤 1：定义工具和模型

      import { ChatAnthropic } from "@langchain/anthropic";
      import { tool } from "@langchain/core/tools";
      import * as z from "zod";

      const model = new ChatAnthropic({
        model: "claude-sonnet-4-5",
        temperature: 0,
      });

      // 定义工具
      const add = tool(({ a, b }) => a + b, {
        name: "add",
        description: "将两个数字相加",
        schema: z.object({
          a: z.number().describe("第一个数字"),
          b: z.number().describe("第二个数字"),
        }),
      });

      const multiply = tool(({ a, b }) => a * b, {
        name: "multiply",
        description: "将两个数字相乘",
        schema: z.object({
          a: z.number().describe("第一个数字"),
          b: z.number().describe("第二个数字"),
        }),
      });

      const divide = tool(({ a, b }) => a / b, {
        name: "divide",
        description: "将两个数字相除",
        schema: z.object({
          a: z.number().describe("第一个数字"),
          b: z.number().describe("第二个数字"),
        }),
      });

      // 使用工具增强 LLM
      const toolsByName = {
        [add.name]: add,
        [multiply.name]: multiply,
        [divide.name]: divide,
      };
      const tools = Object.values(toolsByName);
      const modelWithTools = model.bindTools(tools);

      // 步骤 2：定义状态

      import { StateGraph, START, END } from "@langchain/langgraph";
      import { MessagesZodMeta } from "@langchain/langgraph";
      import { registry } from "@langchain/langgraph/zod";
      import { type BaseMessage } from "@langchain/core/messages";

      const MessagesState = z.object({
        messages: z
          .array(z.custom<BaseMessage>())
          .register(registry, MessagesZodMeta),
        llmCalls: z.number().optional(),
      });

      // 步骤 3：定义模型节点

      import { SystemMessage } from "@langchain/core/messages";
      async function llmCall(state: z.infer<typeof MessagesState>) {
        return {
          messages: await modelWithTools.invoke([
            new SystemMessage(
              "你是一个有帮助的助手，负责对一组输入执行算术运算。"
            ),
            ...state.messages,
          ]),
          llmCalls: (state.llmCalls ?? 0) + 1,
        };
      }

      // 步骤 4：定义工具节点

      import { isAIMessage, ToolMessage } from "@langchain/core/messages";
      async function toolNode(state: z.infer<typeof MessagesState>) {
        const lastMessage = state.messages.at(-1);

        if (lastMessage == null || !isAIMessage(lastMessage)) {
          return { messages: [] };
        }

        const result: ToolMessage[] = [];
        for (const toolCall of lastMessage.tool_calls ?? []) {
          const tool = toolsByName[toolCall.name];
          const observation = await tool.invoke(toolCall);
          result.push(observation);
        }

        return { messages: result };
      }

      // 步骤 5：定义判断是否结束的逻辑

      async function shouldContinue(state: z.infer<typeof MessagesState>) {
        const lastMessage = state.messages.at(-1);
        if (lastMessage == null || !isAIMessage(lastMessage)) return END;

        // 如果 LLM 进行了工具调用，则执行一个操作
        if (lastMessage.tool_calls?.length) {
          return "toolNode";
        }

        // 否则，我们停止（回复用户）
        return END;
      }

      // 步骤 6：构建和编译智能体

      const agent = new StateGraph(MessagesState)
        .addNode("llmCall", llmCall)
        .addNode("toolNode", toolNode)
        .addEdge(START, "llmCall")
        .addConditionalEdges("llmCall", shouldContinue, ["toolNode", END])
        .addEdge("toolNode", "llmCall")
        .compile();

      // 调用
      import { HumanMessage } from "@langchain/core/messages";
      const result = await agent.invoke({
        messages: [new HumanMessage("将 3 和 4 相加。")],
      });

      for (const message of result.messages) {
        console.log(`[${message.getType()}]: ${message.text}`);
      }
      ```
    </Accordion>
  </Tab>

  <Tab title="使用 Functional API">
    ## 1. 定义工具和模型

    在本示例中，我们将使用 Claude Sonnet 4.5 模型，并定义用于加法、乘法和除法的工具。

    ```typescript  theme={null}
    import { ChatAnthropic } from "@langchain/anthropic";
    import { tool } from "@langchain/core/tools";
    import * as z from "zod";

    const model = new ChatAnthropic({
      model: "claude-sonnet-4-5",
      temperature: 0,
    });

    // 定义工具
    const add = tool(({ a, b }) => a + b, {
      name: "add",
      description: "将两个数字相加",
      schema: z.object({
        a: z.number().describe("第一个数字"),
        b: z.number().describe("第二个数字"),
      }),
    });

    const multiply = tool(({ a, b }) => a * b, {
      name: "multiply",
      description: "将两个数字相乘",
      schema: z.object({
        a: z.number().describe("第一个数字"),
        b: z.number().describe("第二个数字"),
      }),
    });

    const divide = tool(({ a, b }) => a / b, {
      name: "divide",
      description: "将两个数字相除",
      schema: z.object({
        a: z.number().describe("第一个数字"),
        b: z.number().describe("第二个数字"),
      }),
    });

    // 使用工具增强 LLM
    const toolsByName = {
      [add.name]: add,
      [multiply.name]: multiply,
      [divide.name]: divide,
    };
    const tools = Object.values(toolsByName);
    const modelWithTools = model.bindTools(tools);

    ```

    ## 2. 定义模型节点

    模型节点用于调用 LLM 并决定是否调用工具。

    <Tip>
      @\[`@task`] 装饰器将函数标记为可作为智能体一部分执行的任务。任务可以在您的入口点函数内同步或异步调用。
    </Tip>

    ```typescript  theme={null}
    import { task, entrypoint } from "@langchain/langgraph";
    import { SystemMessage } from "@langchain/core/messages";
    const callLlm = task({ name: "callLlm" }, async (messages: BaseMessage[]) => {
      return modelWithTools.invoke([
        new SystemMessage(
          "你是一个有帮助的助手，负责对一组输入执行算术运算。"
        ),
        ...messages,
      ]);
    });
    ```

    ## 3. 定义工具节点

    工具节点用于调用工具并返回结果。

    ```typescript  theme={null}
    import type { ToolCall } from "@langchain/core/messages/tool";
    const callTool = task({ name: "callTool" }, async (toolCall: ToolCall) => {
      const tool = toolsByName[toolCall.name];
      return tool.invoke(toolCall);
    });
    ```

    ## 4. 定义智能体

    智能体使用 @\[`@entrypoint`] 函数构建。

    <Note>
      在 Functional API 中，您无需显式定义节点和边，而是在单个函数内编写标准的控制流逻辑（循环、条件语句）。
    </Note>

    ```typescript  theme={null}
    import { addMessages } from "@langchain/langgraph";
    import { type BaseMessage, isAIMessage } from "@langchain/core/messages";

    const agent = entrypoint({ name: "agent" }, async (messages: BaseMessage[]) => {
      let modelResponse = await callLlm(messages);

      while (true) {
        if (!modelResponse.tool_calls?.length) {
          break;
        }

        // 执行工具
        const toolResults = await Promise.all(
          modelResponse.tool_calls.map((toolCall) => callTool(toolCall))
        );
        messages = addMessages(messages, [modelResponse, ...toolResults]);
        modelResponse = await callLlm(messages);
      }

      return messages;
    });

    // 调用
    import { HumanMessage } from "@langchain/core/messages";

    const result = await agent.invoke([new HumanMessage("将 3 和 4 相加。")]);

    for (const message of result) {
      console.log(`[${message.getType()}]: ${message.text}`);
    }
    ```

    恭喜！您已使用 LangGraph Functional API 构建了您的第一个智能体。

    <Accordion title="完整代码示例" icon="code">
      ```typescript  theme={null}
      // 步骤 1：定义工具和模型

      import { ChatAnthropic } from "@langchain/anthropic";
      import { tool } from "@langchain/core/tools";
      import * as z from "zod";

      const model = new ChatAnthropic({
        model: "claude-sonnet-4-5",
        temperature: 0,
      });

      // 定义工具
      const add = tool(({ a, b }) => a + b, {
        name: "add",
        description: "将两个数字相加",
        schema: z.object({
          a: z.number().describe("第一个数字"),
          b: z.number().describe("第二个数字"),
        }),
      });

      const multiply = tool(({ a, b }) => a * b, {
        name: "multiply",
        description: "将两个数字相乘",
        schema: z.object({
          a: z.number().describe("第一个数字"),
          b: z.number().describe("第二个数字"),
        }),
      });

      const divide = tool(({ a, b }) => a / b, {
        name: "divide",
        description: "将两个数字相除",
        schema: z.object({
          a: z.number().describe("第一个数字"),
          b: z.number().describe("第二个数字"),
        }),
      });

      // 使用工具增强 LLM
      const toolsByName = {
        [add.name]: add,
        [multiply.name]: multiply,
        [divide.name]: divide,
      };
      const tools = Object.values(toolsByName);
      const modelWithTools = model.bindTools(tools);

      // 步骤 2：定义模型节点

      import { task, entrypoint } from "@langchain/langgraph";
      import { SystemMessage } from "@langchain/core/messages";
      const callLlm = task({ name: "callLlm" }, async (messages: BaseMessage[]) => {
        return modelWithTools.invoke([
          new SystemMessage(
            "你是一个有帮助的助手，负责对一组输入执行算术运算。"
          ),
          ...messages,
        ]);
      });

      // 步骤 3：定义工具节点

      import type { ToolCall } from "@langchain/core/messages/tool";
      const callTool = task({ name: "callTool" }, async (toolCall: ToolCall) => {
        const tool = toolsByName[toolCall.name];
        return tool.invoke(toolCall);
      });

      // 步骤 4：定义智能体
      import { addMessages } from "@langchain/langgraph";
      import { type BaseMessage, isAIMessage } from "@langchain/core/messages";
      const agent = entrypoint({ name: "agent" }, async (messages: BaseMessage[]) => {
        let modelResponse = await callLlm(messages);

        while (true) {
          if (!modelResponse.tool_calls?.length) {
            break;
          }

          // 执行工具
          const toolResults = await Promise.all(
            modelResponse.tool_calls.map((toolCall) => callTool(toolCall))
          );
          messages = addMessages(messages, [modelResponse, ...toolResults]);
          modelResponse = await callLlm(messages);
        }

        return messages;
      });

      // 调用
      import { HumanMessage } from "@langchain/core/messages";
      const result = await agent.invoke([new HumanMessage("将 3 和 4 相加。")]);

      for (const message of result) {
        console.log(`[${message.getType()}]: ${message.text}`);
      }
      ```
    </Accordion>
  </Tab>
</Tabs>

***

<Callout icon="pen-to-square" iconType="regular">
  [在 GitHub 上编辑此页面的源文件。](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/quickstart.mdx)
</Callout>

<Tip icon="terminal" iconType="regular">
  [通过 MCP 以编程方式连接这些文档](/use-these-docs)，将其与 Claude、VSCode 等集成，以获取实时答案。
</Tip>