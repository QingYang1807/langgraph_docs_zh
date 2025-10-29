# 部署

LangSmith 是将代理转变为生产系统最快的方式。传统托管平台是为无状态的短期 Web 应用程序而构建的，而 LangGraph 专为有状态、长期运行的代理而**量身打造**，因此您可以在几分钟内从代码仓库可靠地部署到云端。

## 前提条件

开始之前，请确保您具备以下条件：

* 一个 [GitHub 账户](https://github.com/)
* 一个 [LangSmith 账户](https://smith.langchain.com/)（注册免费）

## 部署您的代理

### 1. 在 GitHub 上创建仓库

您的应用程序代码必须位于 GitHub 仓库中才能在 LangSmith 上部署。支持公共和私有仓库。对于本快速入门指南，首先请按照[本地服务器设置指南](/oss/python/langchain/studio#setup-local-langgraph-server)确保您的应用程序与 LangGraph 兼容。然后，将您的代码推送到仓库中。

<deploy />

***

<Callout icon="pen-to-square" iconType="regular">
  [在 GitHub 上编辑此页面的源代码。](https://github.com/langchain-ai/docs/edit/main/src/oss/langchain/deploy.mdx)
</Callout>

<Tip icon="terminal" iconType="regular">
  通过 MCP 将这些文档[以编程方式连接](/use-these-docs)到 Claude、VSCode 等，获取实时答案。
</Tip>