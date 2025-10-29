# LangChain 安装指南（Install LangChain）

## 🧱 基础安装

安装核心 `langchain` 包：

```bash
pip install -U langchain
```

或使用更现代的包管理器 `uv`：

```bash
uv add langchain
```

> ✅ LangChain v1 已经默认支持构建智能体（Agent），无需额外依赖即可开始使用。

---

## 🔌 安装模型与服务集成包

LangChain 核心库仅包含智能体构建与运行逻辑。
如果你需要连接特定模型（如 OpenAI、Anthropic、Gemini、Claude 等），
请单独安装对应的 **provider 集成包**。

示例：

```bash
# 安装 OpenAI 模型支持
pip install -U langchain-openai

# 安装 Anthropic（Claude）模型支持
pip install -U langchain-anthropic
```

或使用 uv：

```bash
uv add langchain-openai
uv add langchain-anthropic
```

---

## 🔍 可选的官方集成包

| Provider                          | 包名称                       | 支持功能                              |
| --------------------------------- | ------------------------- | --------------------------------- |
| **OpenAI**                        | `langchain-openai`        | GPT 系列、嵌入模型、结构化输出、流式推理            |
| **Anthropic**                     | `langchain-anthropic`     | Claude 系列、reasoning traces、PII 过滤 |
| **Google**                        | `langchain-google-genai`  | Gemini 模型、多模态输入、搜索辅助              |
| **AWS Bedrock**                   | `langchain-aws`           | 企业集成、私有云模型访问                      |
| **Ollama / Local**                | `langchain-ollama`        | 本地 LLM 部署与调用                      |
| **TogetherAI / Groq / Mistral 等** | 对应 `langchain-<provider>` | 多云多厂商支持                           |

> 💡 完整列表请见：
> [Integrations overview →](https://python.langchain.com/oss/python/integrations/providers/overview)

---

## 🧠 验证安装

测试是否成功安装：

```python
from langchain.agents import create_agent

def hello():
    return "Hello LangChain!"

agent = create_agent(model="openai:gpt-4o-mini", tools=[hello])
result = agent.invoke({"messages": [{"role": "user", "content": "say hi"}]})
print(result)
```

若能正常运行，说明安装成功 🎉

---

## 🛠️ 常见问题

| 问题                                                        | 解决方案                                                               |
| --------------------------------------------------------- | ------------------------------------------------------------------ |
| `ModuleNotFoundError: No module named 'langchain_openai'` | 未安装对应 provider 包，运行 `pip install langchain-openai`                 |
| `Python version < 3.10`                                   | LangChain v1 仅支持 Python ≥ 3.10                                     |
| 无法联网安装                                                    | 使用 `pip install --no-index --find-links=./packages langchain` 离线安装 |

---

✏️ [在 GitHub 上编辑本页](https://github.com/langchain-ai/docs/edit/main/src/oss/langchain/install.mdx)

💻 [通过 MCP 接入 Claude、VSCode 等工具，实现实时问答](/use-these-docs)
