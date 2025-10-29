# 应用结构（Application Structure）

## 概览（Overview）

一个 **LangGraph 应用（Application）** 通常由以下部分组成：

* 一个或多个 **图（graphs）**，实现应用逻辑；
* 一个 **配置文件**（`langgraph.json`），定义依赖、图、环境变量；
* 一个 **依赖说明文件**（如 `package.json`）；
* 一个可选的 **`.env` 文件**，用于存放环境变量。

这些文件共同定义了一个可部署的 LangGraph 应用结构。

---

## 核心概念（Key Concepts）

部署到 **LangSmith** 平台时，你需要提供以下信息：

1. **配置文件**：
   `langgraph.json` 文件，定义依赖、图、环境变量等元信息。
2. **图定义**：
   用于实现应用逻辑的一个或多个 Graph 脚本。
3. **依赖文件**：
   如 `package.json`，描述项目依赖。
4. **环境变量**：
   运行时所需的 API Key 等配置。

---

## 目录结构示例（File Structure）

以下是一个典型的 LangGraph 应用项目结构：

```plaintext
my-app/
├── src/                     # 项目源码目录
│   ├── utils/               # 工具函数（可选）
│   │   ├── tools.ts         # Graph 工具定义
│   │   ├── nodes.ts         # 节点函数
│   │   └── state.ts         # 状态定义
│   └── agent.ts             # 构建 Graph 的核心逻辑
├── package.json             # npm/yarn/pnpm 依赖定义
├── .env                     # 环境变量文件
└── langgraph.json           # LangGraph 配置文件
```

> 💡 LangGraph 应用的目录结构可根据语言（TypeScript / JavaScript）与包管理工具而略有不同。

---

## 配置文件（Configuration File）

`langgraph.json` 是 LangGraph 应用的核心配置文件，定义了：

* 应用的依赖；
* 图（Graph）的路径；
* 环境变量；
* 其他部署相关参数。

> 🔧 CLI 默认会读取当前目录下的 `langgraph.json` 文件。
> 更多字段可参考 [LangGraph CLI 配置文件参考](https://docs.smith.langchain.com/langsmith/cli#configuration-file)。

---

### 配置文件示例（Example）

以下是一个最小可行的配置示例：

```json
{
  "dependencies": ["."],
  "graphs": {
    "my_agent": "./src/agent.js:agent"
  },
  "env": {
    "OPENAI_API_KEY": "secret-key"
  }
}
```

解释：

| 字段             | 说明                                   |
| -------------- | ------------------------------------ |
| `dependencies` | 指定依赖文件所在路径（通常为当前目录 `"."`）            |
| `graphs`       | 声明应用中包含的图（key 为图名，value 为路径 + 导出函数名） |
| `env`          | 指定运行时环境变量，可在此直接设置或通过 `.env` 文件加载     |

---

## 依赖（Dependencies）

LangGraph 应用依赖于其他 JavaScript/TypeScript 库。

确保以下条件满足：

1. 目录中存在依赖文件（如 `package.json`）；
2. 在 `langgraph.json` 的 `dependencies` 中明确声明；
3. 若依赖系统库或二进制包，可使用 `dockerfile_lines` 指定自定义构建指令。

示例：

```json
{
  "dependencies": ["."],
  "dockerfile_lines": [
    "RUN apt-get update && apt-get install -y libpq-dev"
  ]
}
```

---

## 图（Graphs）

使用 `graphs` 键声明可供部署的图实例。

每个条目需包含：

* **唯一名称**；
* **路径与导出函数名**。

```json
{
  "graphs": {
    "chat_agent": "./src/agent.js:createAgentGraph",
    "summarizer": "./src/summary.js:initGraph"
  }
}
```

> 你可以在 `src/agent.ts` 中使用：
>
> ```typescript
> export const agent = builder.compile();
> ```

---

## 环境变量（Environment Variables）

环境变量可通过以下方式配置：

* **开发阶段**：
  在 `.env` 文件或 `langgraph.json` 的 `env` 键中定义。

* **生产部署阶段**：
  通过 LangSmith 部署环境设置。

### 示例

```json
{
  "env": {
    "OPENAI_API_KEY": "sk-xxxxx",
    "ANTHROPIC_API_KEY": "sk-yyyyy"
  }
}
```

或使用 `.env` 文件：

```bash
OPENAI_API_KEY=sk-xxxxx
ANTHROPIC_API_KEY=sk-yyyyy
```

---

## 总结（Summary）

| 模块       | 文件名              | 作用          |
| -------- | ---------------- | ----------- |
| **核心逻辑** | `src/agent.ts`   | 定义与编译 Graph |
| **配置文件** | `langgraph.json` | 指定依赖、图、环境变量 |
| **依赖声明** | `package.json`   | 指定 npm 包    |
| **环境变量** | `.env`           | 存储敏感配置      |
| **工具辅助** | `src/utils/`     | 通用工具函数或节点封装 |

---

> ✏️ [在 GitHub 上编辑此页面](https://github.com/langchain-ai/docs/edit/main/src/oss/langgraph/application-structure.mdx)
> ⚙️ [将文档连接到 Claude、VSCode 等工具](/use-these-docs)，实现智能化开发体验。
