# Memory 概述

[Memory](/oss/javascript/langgraph/add-memory) 是一个用于记录先前交互信息的系统。对于 AI 代理而言，内存至关重要，因为它能让代理记住先前的交互、从反馈中学习，并适应用户偏好。随着代理处理更复杂的任务和进行大量用户交互，这种能力对于提高效率和用户满意度都变得不可或缺。

本概念指南根据其回忆范围涵盖两种类型的内存：

* [短期内存](#short-term-memory)，即[线程](/oss/javascript/langgraph/persistence#threads)作用域内存，通过在会话中维护消息历史来跟踪正在进行的对话。LangGraph 将短期内存作为代理[状态](/oss/javascript/langgraph/graph-api#state)的一部分进行管理。状态通过[检查点](/oss/javascript/langgraph/persistence#checkpoints)持久化到数据库，以便线程可以随时恢复。短期内存在调用图或完成步骤时更新，并在每个步骤开始时读取状态。
* [长期内存](#long-term-memory)跨会话存储用户特定或应用程序级别的数据，并在对话线程之间*共享*。它可以*在任何时间*和*任何线程*中回忆。内存的作用域不限于单个线程 ID，而是任何自定义命名空间。LangGraph 提供[存储](/oss/javascript/langgraph/persistence#memory-store)（[参考文档](https://langchain-ai.github.io/langgraph/reference/store/#langgraph.store.base.BaseStore)）来让您保存和回忆长期内存。

<img src="https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/short-vs-long.png?fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=62665893848db800383dffda7367438a" alt="" data-og-width="571" width="571" data-og-height="372" height="372" data-path="oss/images/short-vs-long.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/short-vs-long.png?w=280&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=b4f9851d9d5e9537fd9b4beeed7eefd5 280w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/short-vs-long.png?w=560&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=52fb6135668273aa8dfc615536c489b3 560w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/short-vs-long.png?w=840&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=0b6a2c6fe724a7db64dd4ad2677f5721 840w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/short-vs-long.png?w=1100&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=0e7ed889aef106cc3190b8b58a159b9c 1100w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/short-vs-long.png?w=1650&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=1997d5223d23ada411cce7d1170d6795 1650w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/short-vs-long.png?w=2500&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=731fe49d8d020f517e88cc90302618a5 2500w" />

## 短期内存

[短期内存](/oss/javascript/langgraph/add-memory#add-short-term-memory) 让您的应用程序记住单个[线程](/oss/javascript/langgraph/persistence#threads)或对话中的先前交互。[线程](/oss/javascript/langgraph/persistence#threads)将会话中的多个交互组织起来，类似于电子邮件将消息分组在单个对话中的方式。

LangGraph 将短期内存作为代理状态的一部分进行管理，并通过线程作用域的检查点进行持久化。此状态通常可以包括对话历史以及其他有状态数据，例如上传的文件、检索的文档或生成的工件。通过将这些存储在图的状态中，机器人可以访问给定对话的完整上下文，同时保持不同线程之间的分离。

### 管理短期内存

对话历史是短期内存最常见的形式，而长对话对当今的 LLM 构成了挑战。完整的历史记录可能无法放入 LLM 的上下文窗口中，导致无法恢复的错误。即使您的 LLM 支持完整的上下文长度，大多数 LLM 在长上下文下仍然表现不佳。它们会被过时或偏离主题的内容“分散注意力”，同时还要忍受响应时间变慢和成本更高的问题。

聊天模型使用消息接受上下文，其中包括开发者提供的指令（系统消息）和用户输入（人类消息）。在聊天应用程序中，消息在人类输入和模型响应之间交替，导致消息列表随时间越来越长。由于上下文窗口有限且令牌丰富的消息列表可能成本高昂，许多应用程序可以通过使用手动移除或遗忘过时信息的技术来受益。

<img src="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/filter.png?fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=89c50725dda7add80732bd2096e07ef2" alt="" data-og-width="594" width="594" data-og-height="200" height="200" data-path="oss/images/filter.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/filter.png?w=280&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=c5ffb27755202e7b13498e8c5e1c2765 280w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/filter.png?w=560&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=5abdad922fc7ea2770fa48825eb210ed 560w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/filter.png?w=840&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=d4dd4837a3a08a42b14f267c45f9e73e 840w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/filter.png?w=1100&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=ea09b560d904b68a4d7f370c88b908ef 1100w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/filter.png?w=1650&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=a83c084c37d9a34547f5435d9c6a6cc6 1650w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/filter.png?w=2500&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=164042153f379621c9d1558ea129caac 2500w" />

有关管理消息的常用技术的更多信息，请参阅[添加和管理内存](/oss/javascript/langgraph/add-memory#manage-short-term-memory)指南。

## 长期内存

LangGraph 中的[长期内存](/oss/javascript/langgraph/add-memory#add-long-term-memory)允许系统跨不同的对话或会话保留信息。与**线程作用域**的短期内存不同，长期内存保存在自定义“命名空间”中。

长期内存是一个复杂的挑战，没有一刀切的解决方案。然而，以下问题提供了一个框架，帮助您应对不同的技术：

* 内存类型是什么？人类使用记忆来记住事实（[语义记忆](#semantic-memory)）、经验（[情景记忆](#episodic-memory)）和规则（[程序性记忆](#procedural-memory)）。AI 代理可以以相同的方式使用记忆。例如，AI 代理可以使用记忆来记住有关用户的特定事实以完成任务。
* [您希望何时更新内存？](#writing-memories) 内存可以作为代理应用程序逻辑的一部分（例如，“在关键路径上”）进行更新。在这种情况下，代理通常在响应用户之前决定记住事实。或者，内存可以作为后台任务（在后台/异步运行并生成记忆的逻辑）进行更新。我们在下面的[章节](#writing-memories)中解释了这些方法之间的权衡。

不同的应用程序需要各种类型的内存。虽然这个类比并不完美，但研究[人类记忆类型](https://www.psychologytoday.com/us/basics/memory/types-of-memory?ref=blog.langchain.dev)可能会富有洞察力。一些研究（例如 [CoALA 论文](https://arxiv.org/pdf/2309.02427)）甚至将这些人类记忆类型映射到 AI 代理中使用的类型。

| 内存类型 | 存储内容 | 人类示例 | 代理示例 |
| -------------------------------- | -------------- | -------------------------- | ------------------- |
| [语义记忆](#semantic-memory) | 事实 | 我在学校学到的知识 | 关于用户的事实 |
| [情景记忆](#episodic-memory) | 经历 | 我做过的事情 | 过去的代理行为 |
| [程序性记忆](#procedural-memory) | 指令 | 本能或运动技能 | 代理系统提示 |

### 语义记忆

无论是在人类还是 AI 代理中，[语义记忆](https://en.wikipedia.org/wiki/Semantic_memory)都涉及特定事实和概念的保留。在人类中，它可以包括在学校学到的信息以及对概念及其关系的理解。对于 AI 代理，语义记忆通常用于通过记住过去交互中的事实或概念来个性化应用程序。

<Note>
  语义记忆与“语义搜索”不同，语义搜索是一种使用“意义”（通常是嵌入向量）查找相似内容的技术。语义记忆是心理学中的一个术语，指存储事实和知识，而语义搜索是一种基于意义而非精确匹配检索信息的方法。
</Note>

语义记忆可以通过不同方式管理：

#### 档案

记忆可以是一个单一、持续更新的“档案”，其中包含关于用户、组织或其他实体（包括代理本身）的范围明确且具体的信息。档案通常只是一个 JSON 文档，其中包含您选择用来表示领域的各种键值对。

在记住档案时，您需要确保每次都在**更新**档案。因此，您需要传入先前的档案并[要求模型生成新档案](https://github.com/langchain-ai/memory-template)（或一些要应用于旧档案的 [JSON 补丁](https://github.com/hinthornw/trustcall)）。随着档案变大，这可能会变得容易出错，并且可能需要将档案拆分为多个文档或在生成文档时进行**严格**解码，以确保内存模式保持有效。

<img src="https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/update-profile.png?fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=8843788f6afd855450986c4cc4cd6abf" alt="" data-og-width="507" width="507" data-og-height="516" height="516" data-path="oss/images/update-profile.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/update-profile.png?w=280&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=0e40fc4d0951eccd4786df184513d73c 280w, https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/update-profile.png?w=560&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=3724f1b77f1f2fee60fa9fe5e8479fc7 560w, https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/update-profile.png?w=840&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=ba05c4768f99f62034c863fc7d824a1a 840w, https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/update-profile.png?w=1100&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=055289accf7ad32f8e884697c1dcdab3 1100w, https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/update-profile.png?w=1650&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=0ae25038922234d8fd5bc6b5eb5585ac 1650w, https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/update-profile.png?w=2500&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=bb4c602c662cdc2104825deec43f113f 2500w" />

#### 集合

或者，记忆可以是一个随时间持续更新和扩展的文档集合。每个单独的记忆可以更 narrowly scoped（范围更窄）且更容易生成，这意味着您随时间**丢失**信息的可能性更小。对于 LLM 来说，为新信息生成*新*对象比将新信息与现有档案协调要容易得多。因此，文档集合往往会导致下游[更高的召回率](https://en.wikipedia.org/wiki/Precision_and_recall)。

然而，这将一些复杂性转移到了内存更新上。模型现在必须*删除*或*更新*列表中的现有项目，这可能很棘手。此外，一些模型可能默认过度插入，而另一些模型可能默认过度更新。请参阅 [Trustcall](https://github.com/hinthornw/trustcall) 包以了解一种管理此问题的方法，并考虑评估（例如，使用 [LangSmith](https://docs.smith.langchain.com/tutorials/Developers/evaluation) 等工具）来帮助您调整行为。

使用文档集合也将复杂性转移到了对列表的内存**搜索**上。`Store` 目前支持[语义搜索](https://langchain-ai.github.io/langgraph/reference/store/#langgraph.store.base.SearchOp.query)和[按内容过滤](https://langchain-ai.github.io/langgraph/reference/store/#langgraph.store.base.SearchOp.filter)。

最后，使用记忆集合可能会使向模型提供全面上下文变得具有挑战性。虽然单个记忆可能遵循特定的模式，但这种结构可能无法捕获完整的上下文或记忆之间的关系。因此，当使用这些记忆生成响应时，模型可能缺乏重要的上下文信息，而这些信息在统一档案方法中更容易获得。

<img src="https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/update-list.png?fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=38851b242981cc87128620091781f7c9" alt="" data-og-width="483" width="483" data-og-height="491" height="491" data-path="oss/images/update-list.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/update-list.png?w=280&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=1086a0d1728b213a85180db2c327b038 280w, https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/update-list.png?w=560&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=1e1b85bbc04bfef17f131e5b65cababc 560w, https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/update-list.png?w=840&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=77eb8e9b9d8afd4a2e7a05626b98afa4 840w, https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/update-list.png?w=1100&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=2745997869c9e572b3b2fe2826aa3b7f 1100w, https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/update-list.png?w=1650&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=3f69fc65b91598673364628ffe989b89 1650w, https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/update-list.png?w=2500&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=f8d229bd8bc606b6471b1619aed75be3 2500w" />

无论采用何种内存管理方法，关键点在于代理将使用语义记忆来[为其响应提供基础](/oss/javascript/langchain/retrieval)，这通常会导致更加个性化和相关的交互。

### 情景记忆

无论是在人类还是 AI 代理中，[情景记忆](https://en.wikipedia.org/wiki/Episodic_memory)都涉及回忆过去的事件或行为。[CoALA 论文](https://arxiv.org/pdf/2309.02427)很好地阐述了这一点：事实可以写入语义记忆，而*经历*可以写入情景记忆。对于 AI 代理，情景记忆通常用于帮助代理记住如何完成任务。

在实践中，情景记忆通常通过少样本示例提示来实现，代理从过去的序列中学习以正确执行任务。有时“展示”比“告知”更容易，LLM 能很好地从示例中学习。少样本学习让您可以通过使用输入输出示例更新提示来“编程”您的 LLM，以说明预期行为。虽然可以使用各种最佳实践来生成少样本示例，但挑战通常在于根据用户输入选择最相关的示例。

请注意，内存[存储](/oss/javascript/langgraph/persistence#memory-store)只是将数据存储为少样本示例的一种方式。如果您希望有更多的开发者参与，或者将少样本与评估框架更紧密地联系在一起，您也可以使用 LangSmith 数据集来存储您的数据。然后，可以开箱即用地使用动态少样本示例选择器来实现相同的目标。LangSmith 将为您索引数据集，并支持根据关键词相似性检索与用户输入最相关的少样本示例。

请观看此[操作视频](https://www.youtube.com/watch?v=37VaU7e7t5o)，了解 LangSmith 中动态少样本示例选择的使用示例。另外，请参阅此[博客文章](https://blog.langchain.dev/few-shot-prompting-to-improve-tool-calling-performance/)，其中展示了少样本提示以提高工具调用性能，以及此[博客文章](https://blog.langchain.dev/aligning-llm-as-a-judge-with-human-preferences/)使用少样本示例使 LLM 与人类偏好保持一致。

### 程序性记忆

无论是在人类还是 AI 代理中，[程序性记忆](https://en.wikipedia.org/wiki/Procedural_memory)都涉及记住用于执行任务的规则。在人类中，程序性记忆就像执行任务的内在知识，例如通过基本的运动技能和平衡来骑自行车。另一方面，情景记忆涉及回忆特定的经历，例如您第一次成功骑上不带辅助轮的自行车，或在风景优美的路线上的难忘骑行。对于 AI 代理，程序性记忆是模型权重、代理代码和代理提示的组合，它们共同决定了代理的功能。

在实践中，代理修改其模型权重或重写其代码的情况相当少见。但是，代理修改其自身提示的情况更为常见。

完善代理指令的一种有效方法是通过[“反思”(Reflection)](https://blog.langchain.dev/reflection-agents/)或元提示。这涉及使用当前指令（例如，系统提示）以及最近的对话或明确的用户反馈来提示代理。然后，代理根据此输入完善其自身的指令。此方法对于指令难以预先指定的任务特别有用，因为它允许代理从其交互中学习和适应。

例如，我们使用外部反馈和提示重写构建了一个[Tweet 生成器](https://www.youtube.com/watch?v=Vn8A3BxfplE)，以为 Twitter 生成高质量的论文摘要。在这种情况下，具体的摘要提示很难*先验地*指定，但用户很容易批评生成的 Tweet 并提供有关如何改进摘要过程的反馈。

下面的伪代码展示了您如何使用 LangGraph 内存[存储](/oss/javascript/langgraph/persistence#memory-store)来实现此功能，使用存储来保存提示，`update_instructions` 节点获取当前提示（以及在与用户对话中捕获在 `state["messages"]` 中的反馈），更新提示，并将新提示保存回存储。然后，`call_model` 从存储中获取更新的提示并使用它生成响应。

```typescript  theme={null}
// *使用*指令的节点
const callModel = async (state: State, store: BaseStore) => {
    const namespace = ["agent_instructions"];
    const instructions = await store.get(namespace, "agent_a");
    // 应用程序逻辑
    const prompt = promptTemplate.format({
        instructions: instructions[0].value.instructions
    });
    // ...
};

// 更新指令的节点
const updateInstructions = async (state: State, store: BaseStore) => {
    const namespace = ["instructions"];
    const currentInstructions = await store.search(namespace);
    // 内存逻辑
    const prompt = promptTemplate.format({
        instructions: currentInstructions[0].value.instructions,
        conversation: state.messages
    });
    const output = await llm.invoke(prompt);
    const newInstructions = output.new_instructions;
    await store.put(["agent_instructions"], "agent_a", {
        instructions: newInstructions
    });
    // ...
};
```

<img src="https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/update-instructions.png?fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=13644c954ed79a45b8a1a762b3e39da1" alt="" data-og-width="493" width="493" data-og-height="515" height="515" data-path="oss/images/update-instructions.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/update-instructions.png?w=280&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=90632c71febee5777be6ae2c338f0880 280w, https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/update-instructions.png?w=560&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=aefcc771a030a2d6a89f815b87e60fd4 560w, https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/update-instructions.png?w=840&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=9115490b76daffe987e3867bc9176386 840w, https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/update-instructions.png?w=1100&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=0df26f2e6f669f2fbea59a9a49482fb4 1100w, https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/update-instructions.png?w=1650&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=132e7b1f377e0b57c03ab31c6d788df4 1650w, https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/update-instructions.png?w=2500&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=651cd1bb14e445a972a671a196b6a893 2500w" />

### 编写记忆

代理编写记忆有两种主要方法：[“在关键路径上”](#in-the-hot-path)和[“在后台”](#in-the-background)。

<img src="https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/hot_path_vs_background.png?fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=edd006d6189dc29a2edcba57c41fd744" alt="" data-og-width="842" width="842" data-og-height="418" height="418" data-path="oss/images/hot_path_vs_background.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/hot_path_vs_background.png?w=280&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=3efd9962012347a64b596d1d36925b33 280w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/hot_path_vs_background.png?w=560&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=add54b5469d7b4a8f22d7da250c19ddf 560w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/hot_path_vs_background.png?w=840&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=1e763d0f2ee8aa4f5b302ad44bc19d2f 840w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/hot_path_vs_background.png?w=1100&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=ce84a68af250e53a4693332d39179136 1100w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/hot_path_vs_background.png?w=1650&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=5f807accb9c63dae57c27c9a1d17f29a 1650w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/hot_path_vs_background.png?w=2500&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=ecef974859d58f691dbb22a4a1cc1572 2500w" />

#### 在关键路径上

在运行时创建记忆既有优点也有挑战。从积极的方面来看，这种方法允许实时更新，使新的记忆可以立即用于后续的交互。它还实现了透明度，因为可以在创建和存储记忆时通知用户。

然而，这种方法也带来了挑战。如果代理需要一个新工具来决定要提交到记忆中的内容，它可能会增加复杂性。此外，推理要保存到记忆中的内容的过程会影响代理的延迟。最后，代理必须在创建记忆和履行其他职责之间进行多任务处理，这可能会影响所创建记忆的数量和质量。

例如，ChatGPT 使用一个 [save\_memories](https://openai.com/index/memory-and-new-controls-for-chatgpt/) 工具来将记忆作为内容字符串进行 upsert（更新插入），并决定是否以及如何在每条用户消息中使用此工具。请参阅我们的 [memory-agent](https://github.com/langchain-ai/memory-agent) 模板作为参考实现。

#### 在后台

将记忆创建作为单独的后台任务提供了几个优势。它消除了主应用程序的延迟，将应用程序逻辑与内存管理分离开来，并允许代理更专注地完成任务。这种方法还在决定创建记忆的时间上提供了灵活性，以避免冗余工作。

然而，这种方法也有其自身的挑战。确定内存写入的频率变得至关重要，因为更新频率过低可能使其他线程缺乏新的上下文。决定何时触发记忆形成也很重要。常见的策略包括在设定的时间段后调度（如果发生新事件则重新调度）、使用 cron 计划，或允许用户或应用程序逻辑手动触发。

请参阅我们的 [memory-service](https://github.com/langchain-ai/memory-template) 模板作为参考实现。

### 内存存储

LangGraph 将长期内存作为 JSON 文档存储在[存储](/oss/javascript/langgraph/persistence#memory-store)中。每个内存都在一个自定义的 `namespace`（类似于文件夹）和一个独特的 `key`（像文件名）下组织。命名空间通常包括用户或组织 ID 或其他标签，以便更容易组织信息。这种结构支持内存的层次化组织。然后，通过内容过滤器支持跨命名空间搜索。

```typescript  theme={null}
import { InMemoryStore } from "@langchain/langgraph";

const embed = (texts: string[]): number[][] => {
    // 替换为实际的嵌入函数或 LangChain 嵌入对象
    return texts.map(() => [1.0, 2.0]);
};

// InMemoryStore 将数据保存到内存字典中。在生产使用中请使用支持数据库的存储。
const store = new InMemoryStore({ index: { embed, dims: 2 } });
const userId = "my-user";
const applicationContext = "chitchat";
const namespace = [userId, applicationContext];

await store.put(
    namespace,
    "a-memory",
    {
        rules: [
            "User likes short, direct language",
            "User only speaks English & TypeScript",
        ],
        "my-key": "my-value",
    }
);

// 按 ID 获取“记忆”
const item = await store.get(namespace, "a-memory");

// 在此命名空间内搜索“记忆”，按内容等效性过滤，按向量相似性排序
const items = await store.search(
    namespace,
    {
        filter: { "my-key": "my-value" },
        query: "language preferences"
    }
);
```

有关内存存储的更多信息，请参阅[持久化](/oss/javascript/langgraph/persistence#memory-store)指南。

***

<Callout icon="pen-to-square" iconType="regular">
  [在 GitHub 上编辑此页面的源代码。](https://github.com/langchain-ai/docs/edit/main/src/oss/concepts/memory.mdx)
</Callout>

<Tip icon="terminal" iconType="regular">
  [通过 MCP 以编程方式连接这些文档](/use-these-docs)，以连接到 Claude、VSCode 等工具，获取实时答案。
</Tip>