# 部署

LangSmith 是将智能体投入生产系统的最快方式。传统托管平台是为无状态的、短暂的 Web 应用程序构建的，而 LangGraph 则**专为实现有状态的、长期运行的智能体而构建**，因此您可以在几分钟内从代码库切换到可靠的云部署。

## 先决条件

在开始之前，请确保您已具备以下条件：

* 一个 [GitHub 账户](https://github.com/)
* 一个 [LangSmith 账户](https://smith.langchain.com/)（免费注册）

## 部署您的智能体

### 1. 在 GitHub 上创建仓库

您应用程序的代码必须位于 GitHub 仓库中才能在 LangSmith 上部署。支持公有和私有仓库。对于本快速入门，首先请确保您的应用程序与 LangGraph 兼容，遵循[本地服务器设置指南](/oss/python/langgraph/studio#setup-local-langgraph-server)。然后，将您的代码推送到仓库。

### 2. 部署到 LangSmith

<Steps>
  <Step title="导航到 LangSmith 部署">
    登录 [LangSmith](https://smith.langchain.com/)。在左侧边栏中，选择 **Deployments**（部署）。
  </Step>

  <Step title="创建新部署">
    点击 **+ New Deployment**（+ 新部署）按钮。将打开一个面板，您可以在其中填写必填字段。
  </Step>

  <Step title="链接仓库">
    如果您是首次用户或添加之前未连接过的私有仓库，请点击 **Add new account**（添加新账户）按钮并按照说明连接您的 GitHub 账户。
  </Step>

  <Step title="部署仓库">
    选择您应用程序的仓库。点击 **Submit**（提交）进行部署。这可能需要大约 15 分钟完成。您可以在 **Deployment details**（部署详情）视图中检查状态。
  </Step>
</Steps>

### 3. 在 Studio 中测试您的应用程序

一旦您的应用程序部署完成：

1. 选择您刚创建的部署以查看更多详情。
2. 点击右上角的 **Studio** 按钮。Studio 将打开并显示您的图表。

### 4. 获取您部署的 API URL

1. 在 LangGraph 的 **Deployment details**（部署详情）视图中，点击 **API URL** 将其复制到剪贴板。
2. 点击 `URL` 将其复制到剪贴板。

### 5. 测试 API

现在您可以测试 API：

<Tabs>
  <Tab title="Python">
    1. 安装 LangGraph Python：

    ```shell  theme={null}
    pip install langgraph-sdk
    ```

    2. 向智能体发送消息：

    ```python  theme={null}
    from langgraph_sdk import get_sync_client # 或使用 get_client 进行异步操作

    client = get_sync_client(url="your-deployment-url", api_key="your-langsmith-api-key")

    for chunk in client.runs.stream(
        None,    # 无线程运行
        "agent", # 智能体名称。在 langgraph.json 中定义。
        input={
            "messages": [{
                "role": "human",
                "content": "What is LangGraph?",
            }],
        },
        stream_mode="updates",
    ):
        print(f"Receiving new event of type: {chunk.event}...")
        print(chunk.data)
        print("\n\n")
    ```
  </Tab>

  <Tab title="Rest API">
    ```bash  theme={null}
    curl -s --request POST \
        --url <DEPLOYMENT_URL>/runs/stream \
        --header 'Content-Type: application/json' \
        --header "X-Api-Key: <LANGSMITH API KEY> \
        --data "{
            \"assistant_id\": \"agent\", `# 智能体名称。在 langgraph.json 中定义。`
            \"input\": {
                \"messages\": [
                    {
                        \"role\": \"human\",
                        \"content\": \"What is LangGraph?\"
                    }
                ]
            },
            \"stream_mode\": \"updates\"
        }"
    ```
  </Tab>
</Tabs>

<Tip>
  LangSmith 提供额外的托管选项，包括自托管和混合托管。更多信息，请参见[托管概览](/langsmith/hosting)。
</Tip>

***

<Callout icon="pen-to-square" iconType="regular">
  [在 GitHub 上编辑本页的源代码。](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/deploy.mdx)
</Callout>

<Tip icon="terminal" iconType="regular">
  [以编程方式连接这些文档](/use-these-docs)，通过 MCP 连接到 Claude、VSCode 等，以获取实时答案。
</Tip>