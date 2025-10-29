# 工作流与智能体

本指南回顾了常见的工作流和智能体模式。

* 工作流具有预定的代码路径，并按照特定顺序运行。
* 智能体是动态的，可以自主定义其流程和工具使用方式。

<img src="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/agent_workflow.png?fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=c217c9ef517ee556cae3fc928a21dc55" alt="智能体工作流" data-og-width="4572" width="4572" data-og-height="2047" height="2047" data-path="oss/images/agent_workflow.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/agent_workflow.png?w=280&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=290e50cff2f72d524a107421ec8e3ff0 280w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/agent_workflow.png?w=560&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=a2bfc87080aee7dd4844f7f24035825e 560w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/agent_workflow.png?w=840&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=ae1fa9087b33b9ff8bc3446ccaa23e3d 840w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/agent_workflow.png?w=1100&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=06003ee1fe07d7a1ea8cf9200e7d0a10 1100w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/agent_workflow.png?w=1650&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=bc98b459a9b1fb226c2887de1696bde0 1650w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/agent_workflow.png?w=2500&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=1933bcdfd5c5b69b98ce96aafa456848 2500w" />

LangGraph 在构建智能体和工作流方面提供了多项优势，包括[持久化](/oss/javascript/langgraph/persistence)、[流式传输](/oss/javascript/langgraph/streaming)以及对调试和[部署](/oss/javascript/langgraph/deploy)的支持。

## 设置

要构建工作流或智能体，您可以使用任何支持结构化输出和工具调用的[聊天模型](/oss/javascript/integrations/chat)。以下示例使用 Anthropic：

1. 安装依赖

<CodeGroup>
  ```bash npm theme={null}
  npm install @langchain/langgraph @langchain/core
  ```

  ```bash pnpm theme={null}
  pnpm add @langchain/langgraph @langchain/core
  ```

  ```bash yarn theme={null}
  yarn add @langchain/langgraph @langchain/core
  ```

  ```bash bun theme={null}
  bun add @langchain/langgraph @langchain/core
  ```
</CodeGroup>

2. 初始化 LLM：

```typescript  theme={null}
import { ChatAnthropic } from "@langchain/anthropic";

const llm = new ChatAnthropic({
  model: "claude-sonnet-4-5",
  apiKey: "<your_anthropic_key>"
});
```

## LLM 与增强功能

工作流和智能体系统基于 LLM 以及您添加的各种增强功能。[工具调用](/oss/javascript/langchain/tools)、[结构化输出](/oss/javascript/langchain/structured-output)和[短期记忆](/oss/javascript/langchain/short-term-memory)是根据需求定制 LLM 的几个选项。

<img src="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/augmented_llm.png?fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=7ea9656f46649b3ebac19e8309ae9006" alt="LLM 增强功能" data-og-width="1152" width="1152" data-og-height="778" height="778" data-path="oss/images/augmented_llm.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/augmented_llm.png?w=280&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=53613048c1b8bd3241bd27900a872ead 280w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/augmented_llm.png?w=560&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=7ba1f4427fd847bd410541ae38d66d40 560w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/augmented_llm.png?w=840&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=503822cf29a28500deb56f463b4244e4 840w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/augmented_llm.png?w=1100&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=279e0440278d3a26b73c72695636272e 1100w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/augmented_llm.png?w=1650&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=d936838b98bc9dce25168e2b2cfd23d0 1650w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/augmented_llm.png?w=2500&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=fa2115f972bc1152b5e03ae590600fa3 2500w" />

```typescript  theme={null}

import * as z from "zod";
import { tool } from "langchain";

// 结构化输出的模式
const SearchQuery = z.object({
  search_query: z.string().describe("针对网络搜索优化的查询。"),
  justification: z
    .string()
    .describe("为什么此查询与用户请求相关。"),
});

// 使用模式增强 LLM 以实现结构化输出
const structuredLlm = llm.withStructuredOutput(SearchQuery);

// 调用增强的 LLM
const output = await structuredLlm.invoke(
  "钙 CT 评分与高胆固醇有何关系？"
);

// 定义工具
const multiply = tool(
  ({ a, b }) => {
    return a * b;
  },
  {
    name: "multiply",
    description: "两个数相乘",
    schema: z.object({
      a: z.number(),
      b: z.number(),
    }),
  }
);

// 使用工具增强 LLM
const llmWithTools = llm.bindTools([multiply]);

// 调用 LLM 并输入触发工具调用
const msg = await llmWithTools.invoke("2 乘以 3 是多少？");

// 获取工具调用
console.log(msg.tool_calls);
```

## 提示链接

提示链接是指每个 LLM 调用处理前一个调用的输出。它通常用于执行明确定义的任务，这些任务可以被分解为更小的、可验证的步骤。一些例子包括：

* 将文档翻译成不同的语言
* 验证生成内容的一致性

<img src="https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/prompt_chain.png?fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=762dec147c31b8dc6ebb0857e236fc1f" alt="提示链接" data-og-width="1412" width="1412" data-og-height="444" height="444" data-path="oss/images/prompt_chain.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/prompt_chain.png?w=280&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=fda27cf4f997e350d4ce48be16049c47 280w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/prompt_chain.png?w=560&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=1374b6de11900d394fc73722a3a6040e 560w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/prompt_chain.png?w=840&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=25246c7111a87b5df5a2af24a0181efe 840w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/prompt_chain.png?w=1100&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=0c57da86a49cf966cc090497ade347f1 1100w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/prompt_chain.png?w=1650&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=a1b5c8fc644d7a80c0792b71769c97da 1650w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/prompt_chain.png?w=2500&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=8a3f66f0e365e503a85b30be48bc1a76 2500w" />

<CodeGroup>
  ```typescript Graph API theme={null}
  import { StateGraph, Annotation } from "@langchain/langgraph";

  // 图状态
  const StateAnnotation = Annotation.Root({
    topic: Annotation<string>,
    joke: Annotation<string>,
    improvedJoke: Annotation<string>,
    finalJoke: Annotation<string>,
  });

  // 定义节点函数

  // 第一次 LLM 调用生成初始笑话
  async function generateJoke(state: typeof StateAnnotation.State) {
    const msg = await llm.invoke(`写一个关于${state.topic}的简短笑话`);
    return { joke: msg.content };
  }

  // 检查笑话是否有笑点的门控函数
  function checkPunchline(state: typeof StateAnnotation.State) {
    // 简单检查 - 笑话是否包含 "?" 或 "!"
    if (state.joke?.includes("?") || state.joke?.includes("!")) {
      return "Pass";
    }
    return "Fail";
  }

    // 第二次 LLM 调用以改进笑话
  async function improveJoke(state: typeof StateAnnotation.State) {
    const msg = await llm.invoke(
      `通过添加文字游戏使这个笑话更有趣：${state.joke}`
    );
    return { improvedJoke: msg.content };
  }

  // 第三次 LLM 调用进行最终润色
  async function polishJoke(state: typeof StateAnnotation.State) {
    const msg = await llm.invoke(
      `为这个笑话添加一个令人惊讶的转折：${state.improvedJoke}`
    );
    return { finalJoke: msg.content };
  }

  // 构建工作流
  const chain = new StateGraph(StateAnnotation)
    .addNode("generateJoke", generateJoke)
    .addNode("improveJoke", improveJoke)
    .addNode("polishJoke", polishJoke)
    .addEdge("__start__", "generateJoke")
    .addConditionalEdges("generateJoke", checkPunchline, {
      Pass: "improveJoke",
      Fail: "__end__"
    })
    .addEdge("improveJoke", "polishJoke")
    .addEdge("polishJoke", "__end__")
    .compile();

  // 调用
  const state = await chain.invoke({ topic: "猫" });
  console.log("初始笑话：");
  console.log(state.joke);
  console.log("\n--- --- ---\n");
  if (state.improvedJoke !== undefined) {
    console.log("改进的笑话：");
    console.log(state.improvedJoke);
    console.log("\n--- --- ---\n");

    console.log("最终笑话：");
    console.log(state.finalJoke);
  } else {
    console.log("笑话未通过质量检查 - 未检测到笑点！");
  }
  ```

  ```typescript Functional API theme={null}
  import { task, entrypoint } from "@langchain/langgraph";

  // 任务

  // 第一次 LLM 调用生成初始笑话
  const generateJoke = task("generateJoke", async (topic: string) => {
    const msg = await llm.invoke(`写一个关于${topic}的简短笑话`);
    return msg.content;
  });

  // 检查笑话是否有笑点的门控函数
  function checkPunchline(joke: string) {
    // 简单检查 - 笑话是否包含 "?" 或 "!"
    if (joke.includes("?") || joke.includes("!")) {
      return "Pass";
    }
    return "Fail";
  }

    // 第二次 LLM 调用以改进笑话
  const improveJoke = task("improveJoke", async (joke: string) => {
    const msg = await llm.invoke(
      `通过添加文字游戏使这个笑话更有趣：${joke}`
    );
    return msg.content;
  });

  // 第三次 LLM 调用进行最终润色
  const polishJoke = task("polishJoke", async (joke: string) => {
    const msg = await llm.invoke(
      `为这个笑话添加一个令人惊讶的转折：${joke}`
    );
    return msg.content;
  });

  const workflow = entrypoint(
    "jokeMaker",
    async (topic: string) => {
      const originalJoke = await generateJoke(topic);
      if (checkPunchline(originalJoke) === "Pass") {
        return originalJoke;
      }
      const improvedJoke = await improveJoke(originalJoke);
      const polishedJoke = await polishJoke(improvedJoke);
      return polishedJoke;
    }
  );

  const stream = await workflow.stream("猫", {
    streamMode: "updates",
  });

  for await (const step of stream) {
    console.log(step);
  }
  ```
</CodeGroup>

## 并行化

通过并行化，多个 LLM 同时处理一个任务。这可以通过同时运行多个独立的子任务，或多次运行相同的任务以检查不同的输出来实现。并行化通常用于：

* 分割子任务并并行运行，提高速度
* 多次运行任务以检查不同的输出，提高置信度

一些例子包括：

* 运行一个处理文档关键词的子任务，同时运行第二个子任务检查格式错误
* 基于不同标准（如引用数量、使用的来源数量和来源质量）多次运行一个评估文档准确性的任务

<img src="https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/parallelization.png?fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=8afe3c427d8cede6fed1e4b2a5107b71" alt="parallelization.png" data-og-width="1020" width="1020" data-og-height="684" height="684" data-path="oss/images/parallelization.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/parallelization.png?w=280&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=88e51062b14d9186a6f0ea246bc48635 280w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/parallelization.png?w=560&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=934941ca52019b7cbce7fbdd31d00f0f 560w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/parallelization.png?w=840&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=30b5c86c545d0e34878ff0a2c367dd0a 840w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/parallelization.png?w=1100&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=6227d2c39f332eaeda23f7db66871dd7 1100w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/parallelization.png?w=1650&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=283f3ee2924a385ab88f2cbfd9c9c48c 1650w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/parallelization.png?w=2500&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=69f6a97716b38998b7b399c3d8ac7d9c 2500w" />

<CodeGroup>
  ```typescript Graph API theme={null}
  import { StateGraph, Annotation } from "@langchain/langgraph";

  // 图状态
  const StateAnnotation = Annotation.Root({
    topic: Annotation<string>,
    joke: Annotation<string>,
    story: Annotation<string>,
    poem: Annotation<string>,
    combinedOutput: Annotation<string>,
  });

  // 节点
  // 第一次 LLM 调用生成初始笑话
  async function callLlm1(state: typeof StateAnnotation.State) {
    const msg = await llm.invoke(`写一个关于${state.topic}的笑话`);
    return { joke: msg.content };
  }

  // 第二次 LLM 调用生成故事
  async function callLlm2(state: typeof StateAnnotation.State) {
    const msg = await llm.invoke(`写一个关于${state.topic}的故事`);
    return { story: msg.content };
  }

  // 第三次 LLM 调用生成诗歌
  async function callLlm3(state: typeof StateAnnotation.State) {
    const msg = await llm.invoke(`写一首关于${state.topic}的诗`);
    return { poem: msg.content };
  }

  // 将笑话、故事和诗歌合并为单一输出
  async function aggregator(state: typeof StateAnnotation.State) {
    const combined = `这是一个关于${state.topic}的故事、笑话和诗歌！\n\n` +
      `故事：\n${state.story}\n\n` +
      `笑话：\n${state.joke}\n\n` +
      `诗歌：\n${state.poem}`;
    return { combinedOutput: combined };
  }

  // 构建工作流
  const parallelWorkflow = new StateGraph(StateAnnotation)
    .addNode("callLlm1", callLlm1)
    .addNode("callLlm2", callLlm2)
    .addNode("callLlm3", callLlm3)
    .addNode("aggregator", aggregator)
    .addEdge("__start__", "callLlm1")
    .addEdge("__start__", "callLlm2")
    .addEdge("__start__", "callLlm3")
    .addEdge("callLlm1", "aggregator")
    .addEdge("callLlm2", "aggregator")
    .addEdge("callLlm3", "aggregator")
    .addEdge("aggregator", "__end__")
    .compile();

  // 调用
  const result = await parallelWorkflow.invoke({ topic: "猫" });
  console.log(result.combinedOutput);
  ```

  ```typescript Functional API theme={null}
  import { task, entrypoint } from "@langchain/langgraph";

  // 任务

  // 第一次 LLM 调用生成初始笑话
  const callLlm1 = task("generateJoke", async (topic: string) => {
    const msg = await llm.invoke(`写一个关于${topic}的笑话`);
    return msg.content;
  });

  // 第二次 LLM 调用生成故事
  const callLlm2 = task("generateStory", async (topic: string) => {
    const msg = await llm.invoke(`写一个关于${topic}的故事`);
    return msg.content;
  });

  // 第三次 LLM 调用生成诗歌
  const callLlm3 = task("generatePoem", async (topic: string) => {
    const msg = await llm.invoke(`写一首关于${topic}的诗`);
    return msg.content;
  });

  // 合并输出
  const aggregator = task("aggregator", async (params: {
    topic: string;
    joke: string;
    story: string;
    poem: string;
  }) => {
    const { topic, joke, story, poem } = params;
    return `这是一个关于${topic}的故事、笑话和诗歌！\n\n` +
      `故事：\n${story}\n\n` +
      `笑话：\n${joke}\n\n` +
      `诗歌：\n${poem}`;
  });

  // 构建工作流
  const workflow = entrypoint(
    "parallelWorkflow",
    async (topic: string) => {
      const [joke, story, poem] = await Promise.all([
        callLlm1(topic),
        callLlm2(topic),
        callLlm3(topic),
      ]);

      return aggregator({ topic, joke, story, poem });
    }
  );

  // 调用
  const stream = await workflow.stream("猫", {
    streamMode: "updates",
  });

  for await (const step of stream) {
    console.log(step);
  }
  ```
</CodeGroup>

## 路由

路由工作流处理输入，然后将它们定向到特定于上下文的任务。这使您能够为复杂任务定义专门的流程。例如，一个用于回答产品相关问题的工作流可能会先处理问题类型，然后将请求路由到特定的流程，如定价、退款、退货等。

<img src="https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/routing.png?fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=272e0e9b681b89cd7d35d5c812c50ee6" alt="routing.png" data-og-width="1214" width="1214" data-og-height="678" height="678" data-path="oss/images/routing.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/routing.png?w=280&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=ab85efe91d20c816f9a4e491e92a61f7 280w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/routing.png?w=560&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=769e29f9be058a47ee85e0c9228e6e44 560w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/routing.png?w=840&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=3711ee40746670731a0ce3e96b7cfeb1 840w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/routing.png?w=1100&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=9aaa28410da7643f4a2587f7bfae0f21 1100w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/routing.png?w=1650&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=6706326c7fef0511805c684d1e4f7082 1650w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/routing.png?w=2500&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=f6d603145ca33791b18c8c8afec0bb4d 2500w" />

<CodeGroup>
  ```typescript Graph API theme={null}
  import { StateGraph, Annotation } from "@langchain/langgraph";
  import * as z from "zod";

  // 用于路由逻辑的结构化输出模式
  const routeSchema = z.object({
    step: z.enum(["poem", "story", "joke"]).describe(
      "路由过程中的下一步"
    ),
  });

  // 使用模式增强 LLM 以实现结构化输出
  const router = llm.withStructuredOutput(routeSchema);

  // 图状态
  const StateAnnotation = Annotation.Root({
    input: Annotation<string>,
    decision: Annotation<string>,
    output: Annotation<string>,
  });

  // 节点
  // 写故事
  async function llmCall1(state: typeof StateAnnotation.State) {
    const result = await llm.invoke([{
      role: "system",
      content: "你是一位专业的讲故事的人。",
    }, {
      role: "user",
      content: state.input
    }]);
    return { output: result.content };
  }

  // 讲笑话
  async function llmCall2(state: typeof StateAnnotation.State) {
    const result = await llm.invoke([{
      role: "system",
      content: "你是一位专业的喜剧演员。",
    }, {
      role: "user",
      content: state.input
    }]);
    return { output: result.content };
  }

  // 写诗
  async function llmCall3(state: typeof StateAnnotation.State) {
    const result = await llm.invoke([{
      role: "system",
      content: "你是一位专业的诗人。",
    }, {
      role: "user",
      content: state.input
    }]);
    return { output: result.content };
  }

  async function llmCallRouter(state: typeof StateAnnotation.State) {
    // 将输入路由到适当的节点
    const decision = await router.invoke([
      {
        role: "system",
        content: "根据用户的请求，将输入路由到故事、笑话或诗歌。"
      },
      {
        role: "user",
        content: state.input
      },
    ]);

    return { decision: decision.step };
  }

  // 将路由到适当节点的条件边缘函数
  function routeDecision(state: typeof StateAnnotation.State) {
    // 返回您想要访问的下一个节点的名称
    if (state.decision === "story") {
      return "llmCall1";
    } else if (state.decision === "joke") {
      return "llmCall2";
    } else if (state.decision === "poem") {
      return "llmCall3";
    }
  }

  // 构建工作流
  const routerWorkflow = new StateGraph(StateAnnotation)
    .addNode("llmCall1", llmCall1)
    .addNode("llmCall2", llmCall2)
    .addNode("llmCall3", llmCall3)
    .addNode("llmCallRouter", llmCallRouter)
    .addEdge("__start__", "llmCallRouter")
    .addConditionalEdges(
      "llmCallRouter",
      routeDecision,
      ["llmCall1", "llmCall2", "llmCall3"],
    )
    .addEdge("llmCall1", "__end__")
    .addEdge("llmCall2", "__end__")
    .addEdge("llmCall3", "__end__")
    .compile();

  // 调用
  const state = await routerWorkflow.invoke({
    input: "给我讲一个关于猫的笑话"
  });
  console.log(state.output);
  ```

  ```typescript Functional API theme={null}
  import * as z from "zod";
  import { task, entrypoint } from "@langchain/langgraph";

  // 用于路由逻辑的结构化输出模式
  const routeSchema = z.object({
    step: z.enum(["poem", "story", "joke"]).describe(
      "路由过程中的下一步"
    ),
  });

  // 使用模式增强 LLM 以实现结构化输出
  const router = llm.withStructuredOutput(routeSchema);

  // 任务
  // 写故事
  const llmCall1 = task("generateStory", async (input: string) => {
    const result = await llm.invoke([{
      role: "system",
      content: "你是一位专业的讲故事的人。",
    }, {
      role: "user",
      content: input
    }]);
    return result.content;
  });

  // 讲笑话
  const llmCall2 = task("generateJoke", async (input: string) => {
    const result = await llm.invoke([{
      role: "system",
      content: "你是一位专业的喜剧演员。",
    }, {
      role: "user",
      content: input
    }]);
    return result.content;
  });

  // 写诗
  const llmCall3 = task("generatePoem", async (input: string) => {
    const result = await llm.invoke([{
      role: "system",
      content: "你是一位专业的诗人。",
    }, {
      role: "user",
      content: input
    }]);
    return result.content;
  });

  // 将输入路由到适当的节点
  const llmCallRouter = task("router", async (input: string) => {
    const decision = await router.invoke([
      {
        role: "system",
        content: "根据用户的请求，将输入路由到故事、笑话或诗歌。"
      },
      {
        role: "user",
        content: input
      },
    ]);
    return decision.step;
  });

  // 构建工作流
  const workflow = entrypoint(
    "routerWorkflow",
    async (input: string) => {
      const nextStep = await llmCallRouter(input);

      let llmCall;
      if (nextStep === "story") {
        llmCall = llmCall1;
      } else if (nextStep === "joke") {
        llmCall = llmCall2;
      } else if (nextStep === "poem") {
        llmCall = llmCall3;
      }

      const finalResult = await llmCall(input);
      return finalResult;
    }
  );

  // 调用
  const stream = await workflow.stream("给我讲一个关于猫的笑话", {
    streamMode: "updates",
  });

  for await (const step of stream) {
    console.log(step);
  }
  ```
</CodeGroup>

## 编排器-工作者

在编排器-工作者配置中，编排器：

* 将任务分解为子任务
* 将子任务分配给工作者
* 将工作者输出合成为最终结果

<img src="https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/worker.png?fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=2e423c67cd4f12e049cea9c169ff0676" alt="worker.png" data-og-width="1486" width="1486" data-og-height="548" height="548" data-path="oss/images/worker.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/worker.png?w=280&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=037222991ea08f889306be035c4730b6 280w, https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/worker.png?w=560&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=081f3ff05cc1fe50770c864d74084b5b 560w, https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/worker.png?w=840&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=0ef6c1b9ceb5159030aa34d0f05f1ada 840w, https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/worker.png?w=1100&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=92ec7353a89ae96e221a5a8f65c88adf 1100w, https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/worker.png?w=1650&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=71b201dd99fa234ebfb918915aac3295 1650w, https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/worker.png?w=2500&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=4f7b6e2064db575027932394a3658fbd 2500w" />

编排器-工作者工作流提供更大的灵活性，通常用于子任务无法像[并行化](#parallelization)那样预定义的情况。这在编写代码或需要在多个文件中更新内容的工作流中很常见。例如，一个需要在未知数量的文档中更新多个 Python 库的安装说明的工作流可能会使用这种模式。

<CodeGroup>
  ```typescript Graph API theme={null}

  type SectionSchema = {
      name: string;
      description: string;
  }
  type SectionsSchema = {
      sections: SectionSchema[];
  }

  // 使用模式增强 LLM 以实现结构化输出
  const planner = llm.withStructuredOutput(sectionsSchema);
  ```

  ```typescript Functional API theme={null}
  import * as z from "zod";
  import { task, entrypoint } from "@langchain/langgraph";

  // 用于规划的结构化输出模式
  const sectionSchema = z.object({
    name: z.string().describe("报告此部分的名称。"),
    description: z.string().describe(
      "此部分将要涵盖的主要主题和概念的简要概述。"
    ),
  });

  const sectionsSchema = z.object({
    sections: z.array(sectionSchema).describe("报告的各个部分。"),
  });

  // 使用模式增强 LLM 以实现结构化输出
  const planner = llm.withStructuredOutput(sectionsSchema);

  // 任务
  const orchestrator = task("orchestrator", async (topic: string) => {
    // 生成查询
    const reportSections = await planner.invoke([
      { role: "system", content: "为报告生成一个计划。" },
      { role: "user", content: `这是报告主题：${topic}` },
    ]);

    return reportSections.sections;
  });

  const llmCall = task("sectionWriter", async (section: z.infer<typeof sectionSchema>) => {
    // 生成部分
    const result = await llm.invoke([
      {
        role: "system",
        content: "编写一个报告部分。",
      },
      {
        role: "user",
        content: `这是部分名称：${section.name} 和描述：${section.description}`,
      },
    ]);

    return result.content;
  });

  const synthesizer = task("synthesizer", async (completedSections: string[]) => {
    // 从各部分合成完整报告
    return completedSections.join("\n\n---\n\n");
  });

  // 构建工作流
  const workflow = entrypoint(
    "orchestratorWorker",
    async (topic: string) => {
      const sections = await orchestrator(topic);
      const completedSections = await Promise.all(
        sections.map((section) => llmCall(section))
      );
      return synthesizer(completedSections);
    }
  );

  // 调用
  const stream = await workflow.stream("创建一份关于 LLM 扩展法则的报告", {
    streamMode: "updates",
  });

  for await (const step of stream) {
    console.log(step);
  }
  ```
</CodeGroup>

### 在 LangGraph 中创建工作者

编排器-工作者工作流很常见，LangGraph 对它们有内置支持。`Send` API 允许您动态创建工作者节点并向它们发送特定输入。每个工作者都有自己的状态，所有工作者输出都被写入一个编排器图可访问的共享状态键。这使编排器能够访问所有工作者输出，并将它们合成为最终输出。下面的示例遍历部分列表，并使用 `Send` API 将一个部分发送给每个工作者。

```typescript  theme={null}
import { Annotation, StateGraph, Send } from "@langchain/langgraph";

// 图状态
const StateAnnotation = Annotation.Root({
  topic: Annotation<string>,
  sections: Annotation<SectionsSchema[]>,
  completedSections: Annotation<string[]>({
    default: () => [],
    reducer: (a, b) => a.concat(b),
  }),
  finalReport: Annotation<string>,
});

// 工作者状态
const WorkerStateAnnotation = Annotation.Root({
  section: Annotation<SectionsSchema>,
  completedSections: Annotation<string[]>({
    default: () => [],
    reducer: (a, b) => a.concat(b),
  }),
});

// 节点
async function orchestrator(state: typeof StateAnnotation.State) {
  // 生成查询
  const reportSections = await planner.invoke([
    { role: "system", content: "为报告生成一个计划。" },
    { role: "user", content: `这是报告主题：${state.topic}` },
  ]);

  return { sections: reportSections.sections };
}

async function llmCall(state: typeof WorkerStateAnnotation.State) {
  // 生成部分
  const section = await llm.invoke([
    {
      role: "system",
      content: "按照提供的名称和描述编写报告部分。每个部分不包括前言。使用 markdown 格式。",
    },
    {
      role: "user",
      content: `这是部分名称：${state.section.name} 和描述：${state.section.description}`,
    },
  ]);

  // 将更新的部分写入已完成的部分
  return { completedSections: [section.content] };
}

async function synthesizer(state: typeof StateAnnotation.State) {
  // 已完成部分的列表
  const completedSections = state.completedSections;

  // 将已完成的部分格式化为字符串，以用作最终部分的上下文
  const completedReportSections = completedSections.join("\n\n---\n\n");

  return { finalReport: completedReportSections };
}

// 创建 llm_call 工作者的条件边缘函数，每个工作者编写报告的一个部分
function assignWorkers(state: typeof StateAnnotation.State) {
  // 通过 Send() API 并行启动部分编写
  return state.sections.map((section) =>
    new Send("llmCall", { section })
  );
}

// 构建工作流
const orchestratorWorker = new StateGraph(StateAnnotation)
  .addNode("orchestrator", orchestrator)
  .addNode("llmCall", llmCall)
  .addNode("synthesizer", synthesizer)
  .addEdge("__start__", "orchestrator")
  .addConditionalEdges(
    "orchestrator",
    assignWorkers,
    ["llmCall"]
  )
  .addEdge("llmCall", "synthesizer")
  .addEdge("synthesizer", "__end__")
  .compile();

// 调用
const state = await orchestratorWorker.invoke({
  topic: "创建一份关于 LLM 扩展法则的报告"
});
console.log(state.finalReport);
```

## 评估器-优化器

在评估器-优化器工作流中，一个 LLM 调用创建响应，另一个评估该响应。如果评估器或[人工在环](/oss/javascript/langgraph/interrupts)确定响应需要改进，则提供反馈并重新创建响应。此循环持续进行，直到生成可接受的响应。

评估器-优化器工作流通常用于任务有特定成功标准但需要迭代才能满足该标准的情况。例如，在两种语言之间翻译文本时并不总是有完美的匹配。可能需要几次迭代才能在两种语言之间生成具有相同含义的翻译。

<img src="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/evaluator_optimizer.png?fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=9bd0474f42b6040b14ed6968a9ab4e3c" alt="evaluator_optimizer.png" data-og-width="1004" width="1004" data-og-height="340" height="340" data-path="oss/images/evaluator_optimizer.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/evaluator_optimizer.png?w=280&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=ab36856e5f9a518b22e71278aa8b1711 280w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/evaluator_optimizer.png?w=560&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=3ec597c92270278c2bac203d36b611c2 560w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/evaluator_optimizer.png?w=840&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=3ad3bfb734a0e509d9b87fdb4e808bfd 840w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/evaluator_optimizer.png?w=1100&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=e82bd25a463d3cdf76036649c03358a9 1100w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/evaluator_optimizer.png?w=1650&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=d31717ae3e76243dd975a53f46e8c1f6 1650w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/evaluator_optimizer.png?w=2500&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=a9bb4fb1583f6ad06c0b13602cd14811 2500w" />

<CodeGroup>
  ```typescript Graph API theme={null}
  import * as z from "zod";
  import { Annotation, StateGraph } from "@langchain/langgraph";

  // 图状态
  const StateAnnotation = Annotation.Root({
    joke: Annotation<string>,
    topic: Annotation<string>,
    feedback: Annotation<string>,
    funnyOrNot: Annotation<string>,
  });

  // 用于评估的结构化输出模式
  const feedbackSchema = z.object({
    grade: z.enum(["funny", "not funny"]).describe(
      "判断笑话是否有趣。"
    ),
    feedback: z.string().describe(
      "如果笑话不有趣，提供如何改进的反馈。"
    ),
  });

  // 使用模式增强 LLM 以实现结构化输出
  const evaluator = llm.withStructuredOutput(feedbackSchema);

  // 节点
  async function llmCallGenerator(state: typeof StateAnnotation.State) {
    // LLM 生成笑话
    let msg;
    if (state.feedback) {
      msg = await llm.invoke(
        `写一个关于${state.topic}的笑话，但要考虑以下反馈：${state.feedback}`
      );
    } else {
      msg = await llm.invoke(`写一个关于${state.topic}的笑话`);
    }
    return { joke: msg.content };
  }

  async function llmCallEvaluator(state: typeof StateAnnotation.State) {
    // LLM 评估笑话
    const grade = await evaluator.invoke(`评估这个笑话${state.joke}`);
    return { funnyOrNot: grade.grade, feedback: grade.feedback };
  }

  // 根据评估器的反馈路由回笑话生成器或结束的条件边缘函数
  function routeJoke(state: typeof StateAnnotation.State) {
    // 根据评估器的反馈路由回笑话生成器或结束
    if (state.funnyOrNot === "funny") {
      return "Accepted";
    } else if (state.funnyOrNot === "not funny") {
      return "Rejected + Feedback";
    }
  }

  // 构建工作流
  const optimizerWorkflow = new StateGraph(StateAnnotation)
    .addNode("llmCallGenerator", llmCallGenerator)
    .addNode("llmCallEvaluator", llmCallEvaluator)
    .addEdge("__start__", "llmCallGenerator")
    .addEdge("llmCallGenerator", "llmCallEvaluator")
    .addConditionalEdges(
      "llmCallEvaluator",
      routeJoke,
      {
        // routeJoke 返回的名称：要访问的下一个节点的名称
        "Accepted": "__end__",
        "Rejected + Feedback": "llmCallGenerator",
      }
    )
    .compile();

  // 调用
  const state = await optimizerWorkflow.invoke({ topic: "猫" });
  console.log(state.joke);
  ```

  ```typescript Functional API theme={null}
  import * as z from "zod";
  import { task, entrypoint } from "@langchain/langgraph";

  // 用于评估的结构化输出模式
  const feedbackSchema = z.object({
    grade: z.enum(["funny", "not funny"]).describe(
      "判断笑话是否有趣。"
    ),
    feedback: z.string().describe(
      "如果笑话不有趣，提供如何改进的反馈。"
    ),
  });

  // 使用模式增强 LLM 以实现结构化输出
  const evaluator = llm.withStructuredOutput(feedbackSchema);

  // 任务
  const llmCallGenerator = task("jokeGenerator", async (params: {
    topic: string;
    feedback?: z.infer<typeof feedbackSchema>;
  }) => {
    // LLM 生成笑话
    const msg = params.feedback
      ? await llm.invoke(
          `写一个关于${params.topic}的笑话，但要考虑以下反馈：${params.feedback.feedback}`
        )
      : await llm.invoke(`写一个关于${params.topic}的笑话`);
    return msg.content;
  });

  const llmCallEvaluator = task("jokeEvaluator", async (joke: string) => {
    // LLM 评估笑话
    return evaluator.invoke(`评估这个笑话${joke}`);
  });

  // 构建工作流
  const workflow = entrypoint(
    "optimizerWorkflow",
    async (topic: string) => {
      let feedback: z.infer<typeof feedbackSchema> | undefined;
      let joke: string;

      while (true) {
        joke = await llmCallGenerator({ topic, feedback });
        feedback = await llmCallEvaluator(joke);

        if (feedback.grade === "funny") {
          break;
        }
      }

      return joke;
    }
  );

  // 调用
  const stream = await workflow.stream("猫", {
    streamMode: "updates",
  });

  for await (const step of stream) {
    console.log(step);
    console.log("\n");
  }
  ```
</CodeGroup>

## 智能体

智能体通常实现为使用[工具](/oss/javascript/langchain/tools)执行操作的 LLM。它们在连续的反馈循环中运行，用于问题和解决方案不可预测的情况。智能体比工作流具有更大的自主性，可以决定它们使用的工具以及如何解决问题。您仍然可以定义可用的工具集和智能体行为的指导原则。

<img src="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/agent.png?fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=bd8da41dbf8b5e6fc9ea6bb10cb63e38" alt="agent.png" data-og-width="1732" width="1732" data-og-height="712" height="712" data-path="oss/images/agent.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/agent.png?w=280&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=f7a590604edc49cfa273b5856f3a3ee3 280w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/agent.png?w=560&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=dff9b17d345fe0fea25616b3b0dc6ebf 560w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/agent.png?w=840&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=bd932835b919f5e58be77221b6d0f194 840w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/agent.png?w=1100&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=d53318b0c9c898a6146991691cbac058 1100w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/agent.png?w=1650&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=ea66fb96bc07c595d321b8b71e651ddb 1650w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/agent.png?w=2500&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=b02599a3c9ba2a5c830b9a346f9d26c9 2500w" />

<Note>
  要开始使用智能体，请参阅[快速入门](/oss/javascript/langchain/quickstart)或在 LangChain 中阅读更多关于[它们的工作原理](/oss/javascript/langchain/agents)的信息。
</Note>

```typescript Using tools theme={null}
import { tool } from "@langchain/core/tools";
import * as z from "zod";

// 定义工具
const multiply = tool(
  ({ a, b }) => {
    return a * b;
  },
  {
    name: "multiply",
    description: "将两个数字相乘",
    schema: z.object({
      a: z.number().describe("第一个数字"),
      b: z.number().describe("第二个数字"),
    }),
  }
);

const add = tool(
  ({ a, b }) => {
    return a + b;
  },
  {
    name: "add",
    description: "将两个数字相加",
    schema: z.object({
      a: z.number().describe("第一个数字"),
      b: z.number().describe("第二个数字"),
    }),
  }
);

const divide = tool(
  ({ a, b }) => {
    return a / b;
  },
  {
    name: "divide",
    description: "将两个数字相除",
    schema: z.object({
      a: z.number().describe("第一个数字"),
      b: z.number().describe("第二个数字"),
    }),
  }
);

// 使用工具增强 LLM
const tools = [add, multiply, divide];
const toolsByName = Object.fromEntries(tools.map((tool) => [tool.name, tool]));
const llmWithTools = llm.bindTools(tools);
```

<CodeGroup>
  ```typescript Graph API theme={null}
  import { MessagesAnnotation, StateGraph } from "@langchain/langgraph";
  import { ToolNode } from "@langchain/langgraph/prebuilt";
  import {
    SystemMessage,
    ToolMessage
  } from "@langchain/core/messages";

  // 节点
  async function llmCall(state: typeof MessagesAnnotation.State) {
    // LLM 决定是否调用工具
    const result = await llmWithTools.invoke([
      {
        role: "system",
        content: "你是一个有帮助的助手，负责对一组输入执行算术运算。"
      },
      ...state.messages
    ]);

    return {
      messages: [result]
    };
  }

  const toolNode = new ToolNode(tools);

  // 路由到工具节点或结束的条件边缘函数
  function shouldContinue(state: typeof MessagesAnnotation.State) {
    const messages = state.messages;
    const lastMessage = messages.at(-1);

    // 如果 LLM 进行工具调用，则执行操作
    if (lastMessage?.tool_calls?.length) {
      return "toolNode";
    }
    // 否则，我们停止（回复用户）
    return "__end__";
  }

  // 构建工作流
  const agentBuilder = new StateGraph(MessagesAnnotation)
    .addNode("llmCall", llmCall)
    .addNode("toolNode", toolNode)
    // 添加边缘以连接节点
    .addEdge("__start__", "llmCall")
    .addConditionalEdges(
      "llmCall",
      shouldContinue,
      ["toolNode", "__end__"]
    )
    .addEdge("toolNode", "llmCall")
    .compile();

  // 调用
  const messages = [{
    role: "user",
    content: "3 加 4。"
  }];
  const result = await agentBuilder.invoke({ messages });
  console.log(result.messages);
  ```

  ```typescript Functional API theme={null}
  import { task, entrypoint, addMessages } from "@langchain/langgraph";
  import { BaseMessageLike, ToolCall } from "@langchain/core/messages";

  const callLlm = task("llmCall", async (messages: BaseMessageLike[]) => {
    // LLM 决定是否调用工具
    return llmWithTools.invoke([
      {
        role: "system",
        content: "你是一个有帮助的助手，负责对一组输入执行算术运算。"
      },
      ...messages
    ]);
  });

  const callTool = task("toolCall", async (toolCall: ToolCall) => {
    // 执行工具调用
    const tool = toolsByName[toolCall.name];
    return tool.invoke(toolCall.args);
  });

  const agent = entrypoint(
    "agent",
    async (messages) => {
      let llmResponse = await callLlm(messages);

      while (true) {
        if (!llmResponse.tool_calls?.length) {
          break;
        }

        // 执行工具
        const toolResults = await Promise.all(
          llmResponse.tool_calls.map((toolCall) => callTool(toolCall))
        );

        messages = addMessages(messages, [llmResponse, ...toolResults]);
        llmResponse = await callLlm(messages);
      }

      messages = addMessages(messages, [llmResponse]);
      return messages;
    }
  );

  // 调用
  const messages = [{
    role: "user",
    content: "3 加 4。"
  }];

  const stream = await agent.stream([messages], {
    streamMode: "updates",
  });

  for await (const step of stream) {
    console.log(step);
  }
  ```
</CodeGroup>

***

<Callout icon="pen-to-square" iconType="regular">
  [在 GitHub 上编辑此页面的源代码。](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/workflows-agents.mdx)
</Callout>

<Tip icon="terminal" iconType="regular">
  [以编程方式连接这些文档](/use-these-docs)到 Claude、VSCode 等工具，通过 MCP 获取实时答案。
</Tip>