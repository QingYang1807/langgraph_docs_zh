# 运行本地服务器

本指南将向您展示如何在本地运行 LangGraph 应用程序。

## 先决条件

在开始之前，请确保您已具备以下条件：

* 一个 [LangSmith](https://smith.langchain.com/settings) 的 API 密钥 - 注册免费

## 1. 安装 LangGraph CLI

```shell  theme={null}
npx @langchain/langgraph-cli
```

## 2. 创建 LangGraph 应用 🌱

使用 [`new-langgraph-project-js` 模板](https://github.com/langchain-ai/new-langgraphjs-project)创建一个新应用。该模板演示了一个单节点应用程序，您可以用自己的逻辑对其进行扩展。

```shell  theme={null}
npm create langgraph
```

## 3. 安装依赖项

在您的新 LangGraph 应用的根目录下，以 `edit` 模式安装依赖项，以便服务器使用您的本地更改：

```shell  theme={null}
cd path/to/your/app
npm install
```

## 4. 创建 `.env` 文件

您会在新 LangGraph 应用的根目录下找到一个 `.env.example` 文件。在应用根目录下创建一个 `.env` 文件，将 `.env.example` 文件的内容复制进去，并填入必要的 API 密钥：

```bash  theme={null}
LANGSMITH_API_KEY=lsv2...
```

## 5. 启动 LangGraph 服务器 🚀

在本地启动 LangGraph API 服务器：

```shell  theme={null}
npx @langchain/langgraph-cli dev
```

示例输出：

```
>    已就绪！
>
>    - API: [http://localhost:2024](http://localhost:2024/)
>
>    - 文档: http://localhost:2024/docs
>
>    - LangGraph Studio Web UI: https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:2024
```

`langgraph dev` 命令会以内存模式启动 LangGraph 服务器。此模式适用于开发和测试目的。对于生产环境，请部署可访问持久化存储后端的 LangGraph 服务器。更多信息，请参阅[托管概述](/langsmith/hosting)。

## 6. 在 Studio 中测试您的应用程序

[Studio](/langsmith/studio) 是一个专用 UI，您可以连接到 LangGraph API 服务器，以在本地可视化、交互和调试您的应用程序。通过访问 `langgraph dev` 命令输出中提供的 URL，在 Studio 中测试您的图：

```
>    - LangGraph Studio Web UI: https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:2024
```

对于运行在自定义主机/端口上的 LangGraph 服务器，请更新 baseURL 参数。

<Accordion title="Safari 兼容性">
  使用命令中的 `--tunnel` 标志创建一个安全隧道，因为 Safari 在连接到 localhost 服务器时存在限制：

  ```shell  theme={null}
  langgraph dev --tunnel
  ```
</Accordion>

## 7. 测试 API

<Tabs>
  <Tab title="Javascript SDK">
    1. 安装 LangGraph JS SDK：

    ```shell  theme={null}
    npm install @langchain/langgraph-sdk
    ```

    2. 向助手发送消息（无线程运行）：

    ```js  theme={null}
    const { Client } = await import("@langchain/langgraph-sdk");

    // 仅当您在调用 langgraph dev 时更改了默认端口时才设置 apiUrl
    const client = new Client({ apiUrl: "http://localhost:2024"});

    const streamResponse = client.runs.stream(
        null, // 无线程运行
        "agent", // 助手 ID
        {
            input: {
                "messages": [
                    { "role": "user", "content": "什么是 LangGraph？"}
                ]
            },
            streamMode: "messages-tuple",
        }
    );

    for await (const chunk of streamResponse) {
        console.log(`正在接收类型为 ${chunk.event} 的新事件...`);
        console.log(JSON.stringify(chunk.data));
        console.log("\n\n");
    }
    ```
  </Tab>
  <Tab title="Rest API">
    ```bash  theme={null}
    curl -s --request POST \
        --url "http://localhost:2024/runs/stream" \
        --header 'Content-Type: application/json' \
        --data "{
            \"assistant_id\": \"agent\",
            \"input\": {
                \"messages\": [
                    {
                        \"role\": \"human\",
                        \"content\": \"什么是 LangGraph？\"
                    }
                ]
            },
            \"stream_mode\": \"messages-tuple\"
        }"
    ```
  </Tab>
</Tabs>

## 下一步

现在您已经有了一个在本地运行的 LangGraph 应用，您可以通过探索部署和高级功能来进一步您的学习之旅：

* [部署快速入门](/langsmith/deployment-quickstart)：使用 LangSmith 部署您的 LangGraph 应用。

* [LangSmith](/langsmith/home)：了解 LangSmith 的基础概念。

* [JS/TS SDK 参考](https://reference.langchain.com/javascript/modules/_langchain_langgraph-sdk.html)：探索 JS/TS SDK API 参考。

***

<Callout icon="pen-to-square" iconType="regular">
  [在 GitHub 上编辑本页的源码。](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/local-server.mdx)
</Callout>

<Tip icon="terminal" iconType="regular">
  [以编程方式连接这些文档](/use-these-docs)到 Claude、VSCode 等工具，通过 MCP 获取实时答案。
</Tip>