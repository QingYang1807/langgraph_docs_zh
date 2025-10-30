# 应用结构

## 概述

一个 LangGraph 应用程序由一个或多个图、一个配置文件（`langgraph.json`）、一个指定依赖项的文件以及一个可选的指定环境变量的 `.env` 文件组成。

本指南展示了应用程序的典型结构，并说明了如何指定使用 LangSmith 部署应用程序所需的信息。

## 核心概念

要使用 LangSmith 进行部署，应提供以下信息：

1. 一个 [LangGraph 配置文件](#configuration-file-concepts)（`langgraph.json`），指定应用程序的依赖项、图和环境变量。
2. 实现应用程序逻辑的[图](#graphs)。
3. 一个指定运行应用程序所需的[依赖项](#dependencies)的文件。
4. 应用程序运行所需的[环境变量](#environment-variables)。

## 文件结构

以下是应用程序目录结构的示例：

### Python (requirements.txt)

```plaintext
my-app/
├── my_agent # 所有的项目代码都在这里
│   ├── utils # 图的工具
│   │   ├── __init__.py
│   │   ├── tools.py # 图的工具
│   │   ├── nodes.py # 图的节点函数
│   │   └── state.py # 图的状态定义
│   ├── __init__.py
│   └── agent.py # 构建图的代码
├── .env # 环境变量
├── requirements.txt # 包依赖项
└── langgraph.json # LangGraph 的配置文件
```

### Python (pyproject.toml)

```plaintext
my-app/
├── my_agent # 所有的项目代码都在这里
│   ├── utils # 图的工具
│   │   ├── __init__.py
│   │   ├── tools.py # 图的工具
│   │   ├── nodes.py # 图的节点函数
│   │   └── state.py # 图的状态定义
│   ├── __init__.py
│   └── agent.py # 构建图的代码
├── .env # 环境变量
├── langgraph.json  # LangGraph 的配置文件
└── pyproject.toml # 项目的依赖项
```

> LangGraph 应用程序的目录结构可能因编程语言和使用的包管理器而异。


## 配置文件

`langgraph.json` 文件是一个 JSON 文件，指定了部署 LangGraph 应用程序所需的依赖项、图、环境变量和其他设置。

有关 JSON 文件中所有支持的键的详细信息，请参阅 [LangGraph 配置文件参考](/langsmith/cli#configuration-file)。

> [LangGraph CLI](/langsmith/cli) 默认使用当前目录中的 `langgraph.json` 配置文件。

### 示例

* 依赖项涉及一个自定义本地包和 `langchain_openai` 包。
* 单个图将从文件 `./your_package/your_file.py` 中的变量 `variable` 加载。
* 环境变量从 `.env` 文件加载。

```json
{
  "dependencies": ["langchain_openai", "./your_package"],
  "graphs": {
    "my_agent": "./your_package/your_file.py:agent"
  },
  "env": "./.env"
}
```

## 依赖项

LangGraph 应用程序可能依赖于其他 Python 包。

通常，您需要指定以下信息以正确设置依赖项：

1. 目录中指定依赖项的文件（例如 `requirements.txt`、`pyproject.toml` 或 `package.json`）。

2. [LangGraph 配置文件](#configuration-file-concepts)中的 `dependencies` 键，指定运行 LangGraph 应用程序所需的依赖项。

3. 任何额外的二进制文件或系统库都可以使用 [LangGraph 配置文件](#configuration-file-concepts)中的 `dockerfile_lines` 键指定。

## 图

使用 [LangGraph 配置文件](#configuration-file-concepts)中的 `graphs` 键来指定部署的 LangGraph 应用程序中将提供哪些图。

您可以在配置文件中指定一个或多个图。每个图由一个名称（应该是唯一的）和一个路径标识，该路径可以是：(1) 编译后的图或 (2) 定义了创建图的函数。

## 环境变量

如果您在本地处理部署的 LangGraph 应用程序，可以在 [LangGraph 配置文件](#configuration-file-concepts)的 `env` 键中配置环境变量。

对于生产部署，您通常需要在部署环境中配置环境变量。

***

[在 GitHub 上编辑此页面的源代码。](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/application-structure.mdx)

[以编程方式连接这些文档](/use-these-docs)，通过 MCP 将其连接到 Claude、VSCode 等，以获得实时答案。