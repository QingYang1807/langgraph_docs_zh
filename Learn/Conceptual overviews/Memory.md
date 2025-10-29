# 记忆概览

[记忆](/oss/python/langgraph/add-memory)是一种能够记住先前交互信息的系统。对于AI代理来说，记忆至关重要，因为它能让它们记住之前的交互，从反馈中学习，并适应用户的偏好。随着代理处理更复杂的任务并拥有更多的用户交互，这种能力对于提高效率和用户满意度都变得必不可少。

这个概念性指南涵盖两种类型的记忆，基于它们的回忆范围：

* [短期记忆](#short-term-memory)，或称[线程](/oss/python/langgraph/persistence#threads)范围记忆，通过在会话中维护消息历史记录来跟踪正在进行的对话。LangGraph将短期记忆作为代理[状态](/oss/python/langgraph/graph-api#state)的一部分进行管理。状态通过[检查点](/oss/python/langgraph/persistence#checkpoints)持久化到数据库中，以便随时可以恢复线程。当图被调用或步骤完成时，短期记忆会更新，状态在每个步骤开始时被读取。
* [长期记忆](#long-term-memory)在会话之间存储特定于用户或应用程序级别的数据，并在*多个*对话线程之间共享。它可以*在任何时间*和*任何线程*中被回忆起来。记忆可以限定在任何自定义命名空间中，而不仅仅在单个线程ID内。LangGraph提供[存储](/oss/python/langgraph/persistence#memory-store)([参考文档](https://langchain-ai.github.io/langgraph/reference/store/#langgraph.store.base.BaseStore))，让您可以保存和检索长期记忆。

<img src="https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/short-vs-long.png?fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=62665893848db800383dffda7367438a" alt="" data-og-width="571" width="571" data-og-height="372" height="372" data-path="oss/images/short-vs-long.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/short-vs-long.png?w=280&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=b4f9851d9d5e9537fd9b4beeed7eefd5 280w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/short-vs-long.png?w=560&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=52fb6135668273aa8dfc615536c489b3 560w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/short-vs-long.png?w=840&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=0b6a2c6fe724a7db64dd4ad2677f5721 840w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/short-vs-long.png?w=1100&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=0e7ed889aef106cc3190b8b58a159b9c 1100w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/short-vs-long.png?w=1650&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=1997d5223d23ada411cce7d1170d6795 1650w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/short-vs-long.png?w=2500&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=731fe49d8d020f517e88cc90302618a5 2500w" />

## 短期记忆

[短期记忆](/oss/python/langgraph/add-memory#add-short-term-memory)让应用程序能够在单个[线程](/oss/python/langgraph/persistence#threads)或对话中记住之前的交互。[线程](/oss/python/langgraph/persistence#threads)在会话中组织多个交互，类似于电子邮件将消息分组到单个对话中的方式。

LangGraph将短期记忆作为代理状态的一部分进行管理，通过线程范围的检查点持久化存储。此状态通常包括对话历史以及其他状态数据，如上传的文件、检索的文档或生成的工件。通过将这些存储在图的状态中，机器人可以访问给定对话的完整上下文，同时保持不同线程之间的分离。

### 管理短期记忆

对话历史是最常见的短期记忆形式，而长对话对当今的LLMs构成了挑战。完整的历史记录可能不适合放入LLM的上下文窗口，导致无法恢复的错误。即使您的LLM支持完整的上下文长度，大多数LLM在长上下文上的表现仍然很差。它们会被过时或不相关的内容"分散注意力"，同时响应时间变慢且成本更高。

聊天模型使用消息来接受上下文，包括开发人员提供的指令（系统消息）和用户输入（人类消息）。在聊天应用程序中，消息在用户输入和模型响应之间交替，导致消息列表随时间变长。由于上下文窗口有限且富含token的消息列表可能很昂贵，许多应用程序可以通过使用手动删除或遗忘过时信息的技术来受益。

<img src="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/filter.png?fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=89c50725dda7add80732bd2096e07ef2" alt="" data-og-width="594" width="594" data-og-height="200" height="200" data-path="oss/images/filter.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/filter.png?w=280&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=c5ffb27755202e7b13498e8c5e1c2765 280w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/filter.png?w=560&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=5abdad922fc7ea2770fa48825eb210ed 560w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/filter.png?w=840&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=d4dd4837a3a08a42b14f267c45f9e73e 840w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/filter.png?w=1100&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=ea09b560d904b68a4d7f370c88b908ef 1100w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/filter.png?w=1650&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=a83c084c37d9a34547f5435d9c6a6cc6 1650w, https://mintcdn.com/langchain-5e9cc07a/-_xGPoyjhyiDWTPJ/oss/images/filter.png?w=2500&fit=max&auto=format&n=-_xGPoyjhyiDWTPJ&q=85&s=164042153f379621c9d1558ea129caac 2500w" />

有关管理消息的常用技术的更多信息，请参阅[添加和管理记忆](/oss/python/langgraph/add-memory#manage-short-term-memory)指南。

## 长期记忆

LangGraph中的[长期记忆](/oss/python/langgraph/add-memory#add-long-term-memory)允许系统在不同对话或会话之间保留信息。与短期记忆不同，短期记忆是**线程范围的**，而长期记忆则保存在自定义的"命名空间"中。

长期记忆是一个复杂的挑战，没有一刀切的解决方案。然而，以下问题提供了一个框架，帮助您了解不同的技术：

* 记忆的类型是什么？人类使用记忆来记住事实([语义记忆](#semantic-memory))、经验([情景记忆](#episodic-memory))和规则([程序记忆](#procedural-memory))。AI代理可以以同样的方式使用记忆。例如，AI代理可以使用记忆来记住关于用户的特定事实来完成一项任务。
* [何时更新记忆？](#writing-memories)记忆可以作为代理应用程序逻辑的一部分进行更新（例如，"在热路径上"）。在这种情况下，代理通常在响应用户之前决定记住事实。或者，记忆可以作为后台任务（在后台运行/异步执行并生成记忆的逻辑）进行更新。我们在[下面的部分](#writing-memories)解释了这些方法之间的权衡。

不同的应用程序需要不同类型的记忆。尽管这种类比并不完美，但研究[人类记忆类型](https://www.psychologytoday.com/us/basics/memory/types-of-memory?ref=blog.langchain.dev)可能会很有启发。一些研究（例如[CoALA论文](https://arxiv.org/pdf/2309.02427)）甚至将这些人类记忆类型映射到AI代理中使用的类型。

| 记忆类型 | 存储内容 | 人类示例 | 代理示例 |
| --- | --- | --- | --- |
| [语义](#semantic-memory) | 事实 | 我在学校学到的东西 | 关于用户的事实 |
| [情景](#episodic-memory) | 经验 | 我做过的事情 | 代理过去的行动 |
| [程序](#procedural-memory) | 指令 | 本能或运动技能 | 代理系统提示 |

### 语义记忆

[语义记忆](https://en.wikipedia.org/wiki/Semantic_memory)在人类和AI代理中，都涉及保留特定的事实和概念。在人类中，它可以包括在学校学到的信息和概念及其关系的理解。对于AI代理，语义记忆通常用于通过记住过去交互中的事实或概念来个性化应用程序。

<Note>
  语义记忆与"语义搜索"不同，语义搜索是一种使用"意义"（通常是嵌入）查找相似内容的技术。语义记忆是一个心理学术语，指的是存储事实和知识，而语义搜索是一种基于意义而非精确匹配检索信息的方法。
</Note>

语义记忆可以通过不同方式管理：

#### 个人资料

记忆可以是一个持续更新的"个人资料"，包含有关用户、组织或其他实体（包括代理本身）的界定明确且具体的信息。个人资料通常只是一个JSON文档，包含您选择用来表示您领域的各种键值对。

在记住个人资料时，您需要确保每次都在**更新**个人资料。因此，您需要传入先前的个人资料，并[要求模型生成新的个人资料](https://github.com/langchain-ai/memory-template)（或一些[JSON补丁](https://github.com/hinthornw/trustcall)应用于旧个人资料）。随着个人资料变大，这可能变得容易出错，并且可能受益于将个人资料拆分为多个文档或在生成文档时使用**严格**解码以确保记忆架构保持有效。

<img src="https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/update-profile.png?fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=8843788f6afd855450986c4cc4cd6abf" alt="" data-og-width="507" width="507" data-og-height="516" height="516" data-path="oss/images/update-profile.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/update-profile.png?w=280&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=0e40fc4d0951eccd4786df184513d73c 280w, https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/update-profile.png?w=560&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=3724f1b77f1f2fee60fa9fe5e8479fc7 560w, https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/update-profile.png?w=840&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=ba05c4768f99f62034c863fc7d824a1a 840w, https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/update-profile.png?w=1100&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=055289accf7ad32f8e884697c1dcdab3 1100w, https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/update-profile.png?w=1650&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=0ae25038922234d8fd5bc6b5eb5585ac 1650w, https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/update-profile.png?w=2500&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=bb4c602c662cdc2104825deec43f113f 2500w" />

#### 集合

或者，记忆可以是一组随时间持续更新和扩展的文档。每个单独的记忆可以更窄范围地限定，也更容易生成，这意味着您随时间**丢失**信息的可能性更小。对于LLM来说，为新信息生成*新*对象，比将新信息与现有个人资料协调更容易。因此，文档集合往往会导致[更高的下游召回率](https://en.wikipedia.org/wiki/Precision_and_recall)。

然而，这会将一些复杂性转移到记忆更新上。模型现在必须*删除*或*更新*列表中的现有项目，这可能很棘手。此外，一些模型可能倾向于过度插入，而其他模型可能倾向于过度更新。有关管理此问题的一种方法，请参见[Trustcall](https://github.com/hinthornw/trustcall)包，并考虑评估（例如使用[LangSmith](https://docs.smith.langchain.com/tutorials/Developers/evaluation)等工具）来帮助您调整行为。

使用文档集合还会将复杂性转移到对列表的内存**搜索**上。`Store`当前支持[语义搜索](https://langchain-ai.github.io/langgraph/reference/store/#langgraph.store.base.SearchOp.query)和[按内容过滤](https://langchain-ai.github.io/langgraph/reference/store/#langgraph.store.base.SearchOp.filter)。

最后，使用记忆集合可能会给模型提供全面上下文带来挑战。虽然单个记忆可能遵循特定的架构，但这种结构可能无法捕捉记忆之间完整的关系。因此，在使用这些记忆生成响应时，模型可能缺乏统一个人资料方法中更容易获得的重要上下文信息。

<img src="https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/update-list.png?fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=38851b242981cc87128620091781f7c9" alt="" data-og-width="483" width="483" data-og-height="491" height="491" data-path="oss/images/update-list.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/update-list.png?w=280&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=1086a0d1728b213a85180db2c327b038 280w, https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/update-list.png?w=560&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=1e1b85bbc04bfef17f131e5b65cababc 560w, https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/update-list.png?w=840&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=77eb8e9b9d8afd4a2e7a05626b98afa4 840w, https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/update-list.png?w=1100&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=2745997869c9e572b3b2fe2826aa3b7f 1100w, https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/update-list.png?w=1650&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=3f69fc65b91598673364628ffe989b89 1650w, https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/update-list.png?w=2500&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=f8d229bd8bc606b6471b1619aed75be3 2500w" />

无论记忆管理方法如何，核心点是代理将使用语义记忆来[为其响应提供基础](/oss/python/langchain/retrieval)，这通常会导致更加个性化和相关的交互。

### 情景记忆

[情景记忆](https://en.wikipedia.org/wiki/Episodic_memory)在人类和AI代理中，都涉及回忆过去的事件或行动。[CoALA论文](https://arxiv.org/pdf/2309.02427)很好地阐述了这一点：事实可以写入语义记忆，而*经验*可以写入情景记忆。对于AI代理，情景记忆通常用于帮助代理记住如何完成任务。

在实践中，情景记忆通常通过少量示例提示来实现，代理从过去的序列中学习以正确执行任务。有时候"展示"比"告诉"更容易，LLM从示例中学习效果很好。少样本学习允许您通过使用输入-输出示例更新提示来"编程"您的LLM，以说明预期行为。虽然可以使用各种最佳实践来生成少样本示例，但挑战通常在于根据用户输入选择最相关的示例。

请注意，记忆[存储](/oss/python/langgraph/persistence#memory-store)只是将数据存储为少样本示例的一种方法。如果您希望有更多的开发人员参与，或将少样本更紧密地与您的评估工具绑定，您还可以使用[LangSmith数据集](/langsmith/index-datasets-for-dynamic-few-shot-example-selection)来存储您的数据。然后可以开箱即用地使用动态少样本示例选择器来实现相同的目标。LangSmith将为您索引数据集，并启用基于关键词相似性（[使用类似BM25的算法](/langsmith/index-datasets-for-dynamic-few-shot-example-selection)进行基于关键词的相似性）检索与用户输入最相关的少样本示例。

有关LangSmith中动态少样本示例选择的示例用法，请观看此[视频](https://www.youtube.com/watch?v=37VaU7e7t5o)。另请参阅此[博客文章](https://blog.langchain.dev/few-shot-prompting-to-improve-tool-calling-performance/)，展示了使用少样本提示来提高工具调用性能，以及此[博客文章](https://blog.langchain.dev/aligning-llm-as-a-judge-with-human-preferences)，使用少样本示例将LLM与人类偏好保持一致。

### 程序记忆

[程序记忆](https://en.wikipedia.org/wiki/Procedural_memory)在人类和AI代理中，都涉及记住用于执行任务的规则。在人类中，程序记忆就像执行任务的内化知识，例如通过基本的运动技能和平衡骑自行车。另一方面，情景记忆涉及回忆特定的经验，例如您第一次在没有辅助轮的情况下成功骑自行车，或一次穿过风景如画路线的难忘骑行。对于AI代理，程序记忆是模型权重、代理代码和代理提示的组合，共同决定代理的功能。

在实践中，代理修改其模型权重或重写其代码的情况相当少见。然而，代理修改自身提示的情况更为常见。

完善代理指令的一种有效方法是使用["反思"](https://blog.langchain.dev/reflection-agents/)或元提示。这涉及向代理提示其当前指令（例如系统提示）以及最近的对话或明确的用户反馈。然后，代理根据此输入调整自己的指令。这种方法对于指令难以预先指定的任务特别有用，因为它允许代理从其交互中学习和适应。

例如，我们使用外部反馈和提示重写构建了一个[Tweet生成器](https://www.youtube.com/watch?v=Vn8A3BxfplE)，用于生成高质量的Twitter论文摘要。在这种情况下，特定的摘要提示难以预先指定，但用户很容易评论生成的推文并提供有关如何改进摘要过程的反馈。

下面的伪代码展示了如何使用LangGraph记忆[存储](/oss/python/langgraph/persistence#memory-store)来实现这一点，使用存储保存提示，`update_instructions`节点获取当前提示（以及与用户对话中捕获在`state["messages"]`中的反馈），更新提示，然后将新提示保存回存储。然后，`call_model`从存储中获取更新的提示并使用它生成响应。

```python  theme={null}
# 使用指令的节点
def call_model(state: State, store: BaseStore):
    namespace = ("agent_instructions", )
    instructions = store.get(namespace, key="agent_a")[0]
    # 应用逻辑
    prompt = prompt_template.format(instructions=instructions.value["instructions"])
    ...

# 更新指令的节点
def update_instructions(state: State, store: BaseStore):
    namespace = ("instructions",)
    instructions = store.search(namespace)[0]
    # 记忆逻辑
    prompt = prompt_template.format(instructions=instructions.value["instructions"], conversation=state["messages"])
    output = llm.invoke(prompt)
    new_instructions = output['new_instructions']
    store.put(("agent_instructions",), "agent_a", {"instructions": new_instructions})
    ...
```

<img src="https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/update-instructions.png?fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=13644c954ed79a45b8a1a762b3e39da1" alt="" data-og-width="493" width="493" data-og-height="515" height="515" data-path="oss/images/update-instructions.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/update-instructions.png?w=280&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=90632c71febee5777be6ae2c338f0880 280w, https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/update-instructions.png?w=560&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=aefcc771a030a2d6a89f815b87e60fd4 560w, https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/update-instructions.png?w=840&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=9115490b76daffe987e3867bc9176386 840w, https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/update-instructions.png?w=1100&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=0df26f2e6f669f2fbea59a9a49482fb4 1100w, https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/update-instructions.png?w=1650&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=132e7b1f377e0b57c03ab31c6d788df4 1650w, https://mintcdn.com/langchain-5e9cc07a/ybiAaBfoBvFquMDz/oss/images/update-instructions.png?w=2500&fit=max&auto=format&n=ybiAaBfoBvFquMDz&q=85&s=651cd1bb14e445a972a671a196b6a893 2500w" />

### 写入记忆

代理写入记忆有两种主要方法：["在热路径中"](#in-the-hot-path)和["在后台中"](#in-the-background)。

<img src="https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/hot_path_vs_background.png?fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=edd006d6189dc29a2edcba57c41fd744" alt="" data-og-width="842" width="842" data-og-height="418" height="418" data-path="oss/images/hot_path_vs_background.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/hot_path_vs_background.png?w=280&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=3efd9962012347a64b596d1d36925b33 280w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/hot_path_vs_background.png?w=560&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=add54b5469d7b4a8f22d7da250c19ddf 560w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/hot_path_vs_background.png?w=840&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=1e763d0f2ee8aa4f5b302ad44bc19d2f 840w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/hot_path_vs_background.png?w=1100&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=ce84a68af250e53a4693332d39179136 1100w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/hot_path_vs_background.png?w=1650&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=5f807accb9c63dae57c27c9a1d17f29a 1650w, https://mintcdn.com/langchain-5e9cc07a/dL5Sn6Cmy9pwtY0V/oss/images/hot_path_vs_background.png?w=2500&fit=max&auto=format&n=dL5Sn6Cmy9pwtY0V&q=85&s=ecef974859d58f691dbb22a4a1cc1572 2500w" />

#### 在热路径中

在运行时创建记忆既有优势也有挑战。在积极的一面，这种方法允许实时更新，使新记忆能够立即在后续交互中使用。它还实现了透明度，因为可以在创建和存储记忆时通知用户。

然而，这种方法也存在挑战。如果代理需要新工具来决定要提交什么记忆，它可能会增加复杂性。此外，关于保存什么到记忆的推理过程可能会影响代理延迟。最后，代理必须在记忆创建和其他职责之间多任务处理，这可能影响记忆创建的数量和质量。

例如，ChatGPT使用一个[save_memories](https://openai.com/index/memory-and-new-controls-for-chatgpt/)工具来将记忆作为内容字符串插入，决定是否以及如何使用此工具处理每个用户消息。请参阅我们的[memory-agent](https://github.com/langchain-ai/memory-agent)模板作为参考实现。

#### 在后台中

将记忆创建作为单独的后台任务提供了几个优势。它消除了主应用程序中的延迟，将应用程序逻辑与内存管理分开，并允许代理更专注地完成任务。这种方法还提供了在时间上安排记忆创建的灵活性，以避免冗余工作。

然而，这种方法也有其自身的挑战。确定记忆写入的频率变得至关重要，因为不频繁的更新可能会使其他线程没有新上下文。决定何时触发记忆形成也很重要。常见策略包括在设定的时间段后调度（如果有新事件发生则重新调度），使用cron计划，或允许用户或应用程序逻辑手动触发。

请参阅我们的[memory-service](https://github.com/langchain-ai/memory-template)模板作为参考实现。

### 记忆存储

LangGraph将长期记忆作为JSON文档存储在[存储](/oss/python/langgraph/persistence#memory-store)中。每个记忆都组织在一个自定义的`namespace`（类似于文件夹）和一个独特的`key`（如文件名）下。命名空间通常包括用户或组织ID或其他标签，使组织信息更容易。这种结构实现了记忆的分层组织。然后可以通过内容过滤器支持跨命名空间搜索。

```python  theme={null}
from langgraph.store.memory import InMemoryStore


def embed(texts: list[str]) -> list[list[float]]:
    # 替换为实际的嵌入函数或LangChain嵌入对象
    return [[1.0, 2.0] * len(texts)]


# InMemoryStore将数据保存到内存字典中。在生产环境中使用基于数据库的存储。
store = InMemoryStore(index={"embed": embed, "dims": 2})
user_id = "my-user"
application_context = "chitchat"
namespace = (user_id, application_context)
store.put(
    namespace,
    "a-memory",
    {
        "rules": [
            "用户喜欢简短直接的语言",
            "用户只讲英语和python",
        ],
        "my-key": "my-value",
    },
)
# 按ID获取"记忆"
item = store.get(namespace, "a-memory")
# 在此命名空间内搜索"记忆"，基于内容等效性过滤，按向量相似性排序
items = store.search(
    namespace, filter={"my-key": "my-value"}, query="语言偏好"
)
```

有关记忆存储的更多信息，请参阅[Persistence](/oss/python/langgraph/persistence#memory-store)指南。

***

<Callout icon="pen-to-square" iconType="regular">
  [在GitHub上编辑此页面源代码。](https://github.com/langchain-ai/docs/edit/main/src/oss/concepts/memory.mdx)
</Callout>

<Tip icon="terminal" iconType="regular">
  [通过MCP以编程方式连接这些文档](/use-these-docs)到Claude、VSCode等，以获得实时答案。
</Tip>