# 部署指南（Deploy）

LangSmith 是将智能体（Agent）快速投入生产环境的最快方式。
传统的托管平台主要面向**无状态、短生命周期**的 Web 应用，而 **LangGraph** 专为**有状态、长生命周期**的 Agent 设计，能够让你在几分钟内从代码仓库部署到云端的可靠运行环境。

---

## 前置条件

开始前，请确保你已准备好以下内容：

* 一个 [GitHub 账号](https://github.com/)
* 一个 [LangSmith 账号](https://smith.langchain.com/)（注册免费）

---

## 部署你的 Agent

### 1. 在 GitHub 上创建仓库

要在 LangSmith 上部署应用，必须先将代码托管到 GitHub 仓库中。
支持 **公共仓库** 和 **私有仓库**。

若你尚未完成 LangGraph 的本地环境配置，请先参考 [本地服务搭建指南](/oss/javascript/langgraph/studio#setup-local-langgraph-server)。
完成后，将你的项目代码推送至 GitHub。

---

### 2. 部署到 LangSmith

#### 步骤 1：进入 LangSmith 部署面板

登录 [LangSmith](https://smith.langchain.com/)，在左侧菜单栏中选择 **Deployments（部署）**。

#### 步骤 2：创建新部署

点击 **+ New Deployment（新建部署）** 按钮，系统将弹出配置面板。

#### 步骤 3：连接 GitHub 仓库

首次使用或连接新的私有仓库时，点击 **Add new account（添加新账户）** 按钮，按照提示授权连接 GitHub 账号。

#### 步骤 4：部署仓库

选择你的应用仓库并点击 **Submit（提交）**。
部署过程约需 15 分钟，可在 **Deployment details（部署详情）** 页面查看进度。

---

### 3. 在 Studio 中测试你的应用

部署完成后：

1. 选择刚创建的部署以查看详情；
2. 点击右上角的 **Studio** 按钮，即可在可视化界面中查看你的 Agent 结构图。

---

### 4. 获取部署 API 地址

1. 在 LangGraph 的 **Deployment details（部署详情）** 页面中，点击 **API URL**；
2. 点击后即可将地址复制到剪贴板。

---

### 5. 测试 API 接口

部署成功后，可以通过 Python SDK 或 REST API 测试你的 Agent。

---

#### **Python 方式**

1. 安装 LangGraph SDK：

```bash
pip install langgraph-sdk
```

2. 向 Agent 发送请求：

```python
from langgraph_sdk import get_sync_client  # 或使用 get_client 以支持异步调用

client = get_sync_client(
    url="your-deployment-url",
    api_key="your-langsmith-api-key"
)

for chunk in client.runs.stream(
    None,    # 无会话线程（Threadless run）
    "agent", # Agent 名称，与 langgraph.json 中定义一致
    input={
        "messages": [{
            "role": "human",
            "content": "What is LangGraph?",
        }],
    },
    stream_mode="updates",
):
    print(f"收到新事件类型：{chunk.event}...")
    print(chunk.data)
    print("\n\n")
```

---

#### **REST API 方式**

```bash
curl -s --request POST \
    --url <DEPLOYMENT_URL>/runs/stream \
    --header 'Content-Type: application/json' \
    --header "X-Api-Key: <LANGSMITH_API_KEY>" \
    --data "{
        \"assistant_id\": \"agent\",  # 与 langgraph.json 中定义一致
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

---

> 💡 **提示**：
> LangSmith 还提供自托管与混合部署方案。
> 详情请参阅 [Hosting Overview（托管概览）](/langsmith/hosting)。

---

## 附录

✏️ [在 GitHub 上编辑本页源码](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/deploy.mdx)

💻 [通过 MCP 将本文档连接至 Claude、VSCode 等工具，实现实时问答](/use-these-docs)
