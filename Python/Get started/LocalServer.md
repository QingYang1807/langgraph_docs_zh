# 运行本地服务器

本指南向您展示如何本地运行 LangGraph 应用程序。

## 先决条件

开始之前，请确保您具备以下条件：

* [LangSmith](https://smith.langchain.com/settings) 的 API 密钥 - 免费注册

## 1. 安装 LangGraph CLI

pip
```bash
# 需要 Python >= 3.11。
pip install -U "langgraph-cli[inmem]"
```

uv
```bash
# 需要 Python >= 3.11。
uv add langgraph-cli[inmem]
```

## 2. 创建 LangGraph 应用 🌱

从 [`new-langgraph-project-python` 模板](https://github.com/langchain-ai/new-langgraph-project)创建一个新应用。此模板演示了一个单节点应用程序，您可以使用自己的逻辑对其进行扩展。

```shell
langgraph new path/to/your/app --template new-langgraph-project-python
```


> **额外模板**
> 如果您使用 `langgraph new` 而不指定模板，将会显示一个交互式菜单，让您从可用模板列表中进行选择。

## 3. 安装依赖项

在新的 LangGraph 应用程序根目录中，以 `edit` 模式安装依赖项，以便服务器使用您的本地更改：

```bash
cd path/to/your/app
pip install -e .
```

```bash
cd path/to/your/app
uv add .
```

## 4. 创建 `.env` 文件

您将在新的 LangGraph 应用程序根目录中找到一个 `.env.example` 文件。在新的 LangGraph 应用程序根目录中创建一个 `.env` 文件，并将 `.env.example` 文件的内容复制到其中，填入必要的 API 密钥：

```bash
LANGSMITH_API_KEY=lsv2...
```

## 5. 启动 LangGraph 服务器 🚀

在本地启动 LangGraph API 服务器：

```shell
langgraph dev
```

示例输出：

```
>    准备就绪！
>
>    - API: [http://localhost:2024](http://localhost:2024/)
>
>    - 文档: http://localhost:2024/docs
>
>    - LangGraph Studio Web UI: https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:2024
```

`langgraph dev` 命令以内存模式启动 LangGraph 服务器。此模式适用于开发和测试目的。对于生产使用，请部署具有持久存储后端访问权限的 LangGraph 服务器。有关更多信息，请参阅[托管概述](/langsmith/hosting)。

## 6. 在 Studio 中测试您的应用程序

[Studio](/langsmith/studio) 是一个专门的 UI，您可以连接到 LangGraph API 服务器，以在本地可视化、交互和调试您的应用程序。通过访问 `langgraph dev` 命令输出中提供的 URL 来测试您的图形：

```
>    - LangGraph Studio Web UI: https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:2024
```

对于在自定义主机/端口上运行的 LangGraph 服务器，请更新 baseURL 参数。


**Safari 兼容性**

使用 `--tunnel` 标志创建安全隧道，因为 Safari 在连接到本地服务器时存在限制：

```shell
langgraph dev --tunnel
```

## 7. 测试 API

<Tabs>
  <Tab title="Python SDK (异步)">
    1. 安装 LangGraph Python SDK：

    ```shell
    pip install langgraph-sdk
    ```

    2. 向助手发送消息（无线程运行）：

    ```python
    from langgraph_sdk import get_client
    import asyncio

    client = get_client(url="http://localhost:2024")

    async def main():
        async for chunk in client.runs.stream(
            None,  # 无线程运行
            "agent", # 助手名称。在 langgraph.json 中定义。
            input={
            "messages": [{
                "role": "human",
                "content": "什么是 LangGraph？",
                }],
            },
        ):
            print(f"接收到新类型的事件: {chunk.event}...")
            print(chunk.data)
            print("\n\n")

    asyncio.run(main())
    ```
  </Tab>

  <Tab title="Python SDK (同步)">
    1. 安装 LangGraph Python SDK：

    ```shell
    pip install langgraph-sdk
    ```

    2. 向助手发送消息（无线程运行）：

    ```python
    from langgraph_sdk import get_sync_client

    client = get_sync_client(url="http://localhost:2024")

    for chunk in client.runs.stream(
        None,  # 无线程运行
        "agent", # 助手名称。在 langgraph.json 中定义。
        input={
            "messages": [{
                "role": "human",
                "content": "什么是 LangGraph？",
            }],
        },
        stream_mode="messages-tuple",
    ):
        print(f"接收到新类型的事件: {chunk.event}...")
        print(chunk.data)
        print("\n\n")
    ```
  </Tab>

  <Tab title="Rest API">
    ```bash
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

既然您已经在本地运行了 LangGraph 应用程序，请通过探索部署和高级功能来进一步扩展您的旅程：

* [部署快速入门](/langsmith/deployment-quickstart)：使用 LangSmith 部署您的 LangGraph 应用程序。

* [LangSmith](/langsmith/home)：了解基础 LangSmith 概念。

* [Python SDK 参考](https://reference.langchain.com/python/platform/python_sdk/)：探索 Python SDK API 参考。

***

[在 GitHub 上编辑此页面源代码。](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/local-server.mdx)

[通过 MCP 将这些文档编程连接](/use-these-docs)到 Claude、VSCode 等，以获得实时答案。