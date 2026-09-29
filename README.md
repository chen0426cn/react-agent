# react-agent · 机器人智能客服 Agent

基于 **LangChain 1.x** 的 ReAct 风格智能体，面向扫地/扫拖一体机器人场景，结合 **RAG 检索增强**与**动态提示词切换**，并通过 **Streamlit** 提供对话式 Web 界面。

> 本项目为示例 / 教学演示项目：部分工具（天气、用户位置、用户 ID、当前月份）为模拟实现，用于演示 Agent 的工具调用与中间件机制。

---

## ✨ 功能特性

- **ReAct Agent**：使用 `langchain.agents.create_agent` 构建，具备自主规划与工具调用能力。
- **RAG 检索增强**：基于 Chroma 向量库 + 通义 `text-embedding-v4` 嵌入，从机器人知识库文档中检索参考资料并总结作答。
- **多工具协作**：内置 7 个工具，覆盖知识检索、外部数据查询、上下文注入等。
- **动态提示词切换**：通过 Agent 中间件（`middleware`）实现运行时提示词动态替换——检测到"生成报告"场景时自动切换为报告生成提示词。
- **流式对话 UI**：Streamlit 实现，支持打字机式流式输出。

---

## 🧱 技术栈

| 类别 | 技术 |
| --- | --- |
| Agent 框架 | LangChain 1.x（`create_agent`）、LangGraph runtime |
| 对话模型 | DeepSeek（`deepseek-v4-flash`，经 `langchain-deepseek`） |
| 嵌入模型 | 通义 DashScope `text-embedding-v4` |
| 向量库 | Chroma（`langchain-chroma`） |
| Web UI | Streamlit |
| 配置 | PyYAML |
| 语言 | Python 3.11+ |

---

## 📁 目录结构

```
react-agent/
├── app.py                 # Streamlit 入口（机器人智能客服 UI）
├── agent/
│   ├── react_agent.py     # ReactAgent：封装 create_agent
│   └── tools/
│       ├── agent_tools.py # 工具定义（RAG检索 / 天气 / 用户 / 外部数据 / 报告上下文）
│       └── middleware.py  # 中间件（工具监控 / 模型前日志 / 动态提示词切换）
├── rag/
│   ├── rag_service.py     # RAG 总结服务（检索 + 总结链）
│   └── vector_store.py    # Chroma 向量库构建与检索（含 md5 去重）
├── model/
│   └── factory.py         # 模型工厂（DeepSeek 对话模型 + 通义向量模型）
├── config/                # YAML 配置（模型 / 向量库 / 提示词 / 外部数据路径）
│   ├── agent.yml
│   ├── rag.yml
│   ├── chroma.yml
│   └── prompts.yml
├── prompts/               # 提示词文本
│   ├── main_prompt.txt    # 系统提示词
│   ├── rag_summarize.txt  # RAG 总结提示词
│   └── report_prompt.txt  # 报告生成提示词
├── data/                  # 知识库文档（机器人 FAQ）+ 外部使用记录
│   ├── external/records.csv
│   └── *.txt / *.pdf
├── utils/                 # 配置 / 文件 / 日志 / 路径 / 提示词加载工具
├── requirements.txt       # 锁定版本的依赖
└── .gitignore
```

> 注：`rag/chroma_db/`（向量索引）与 `logs/`（运行日志）为**运行时生成产物**，已在 `.gitignore` 中忽略，不会进入版本库。

---

## 🚀 快速开始

### 1. 克隆仓库

```bash
git clone https://github.com/chen0426cn/react-agent.git
cd react-agent
```

### 2. 创建并激活虚拟环境

```bash
python -m venv .venv
# Windows
.\.venv\Scripts\activate
# macOS / Linux
source .venv/bin/activate
```

### 3. 安装依赖

```bash
pip install -r requirements.txt
```

### 4. 配置环境变量

在项目根目录创建 `.env` 文件，填入以下密钥：

```dotenv
# DeepSeek 对话模型密钥
DEEPSEEK_API_KEY=你的DeepSeek密钥
# DeepSeek API 地址（可选，默认 https://api.deepseek.com）
DEEPSEEK_API_BASE=https://api.deepseek.com

# 通义 DashScope 密钥（向量嵌入模型 text-embedding-v4 需要）
DASHSCOPE_API_KEY=你的通义千问密钥
```

> 密钥通过 `python-dotenv` 在 `model/factory.py` 中加载，并已加入 `.gitignore`，请勿提交。

### 5. 构建知识库向量索引（首次必跑）

将 `data/` 下的知识库文档切分、嵌入并写入 Chroma 向量库（基于文件 md5 去重，重复运行不会重复加载）：

```bash
python -m rag.vector_store
```

### 6. 启动 Web 界面

```bash
streamlit run app.py
```

浏览器打开 Streamlit 输出的本地地址即可开始对话。

---

## ⚙️ 配置说明

所有配置位于 `config/` 下的 YAML 文件，由 `utils/config_handler.py` 统一加载：

| 文件 | 关键项 | 说明 |
| --- | --- | --- |
| `config/rag.yml` | `chat_model_name` / `embedding_model_name` | 对话模型与嵌入模型名称 |
| `config/chroma.yml` | `persist_directory` / `collection_name` / `k` / `data_path` / `chunk_size` / `chunk_overlap` | 向量库持久化路径、检索数量、知识库目录、切分参数 |
| `config/agent.yml` | `external_data_path` | 外部使用记录 CSV 路径 |
| `config/prompts.yml` | `*_prompt_path` | 各提示词文本文件路径 |

---

## 🔍 工作原理

1. **Agent 编排**：`ReactAgent` 用 `create_agent` 组装模型、系统提示词、`agent_tools` 中的工具列表，以及 `middleware` 中的中间件。
2. **工具集**（`agent/tools/agent_tools.py`）：
   - `rag_summarize` —— 从向量库检索参考资料并总结；
   - `fetch_external_data` —— 按 `user_id` + `month` 查询 `records.csv` 中的使用记录；
   - `get_weather` / `get_user_location` / `get_user_id` / `get_current_month` —— 示例/模拟工具；
   - `fill_context_for_report` —— 触发报告场景的上下文标记。
3. **中间件**（`agent/tools/middleware.py`）：
   - `monitor_tool`（`wrap_tool_call`）—— 记录工具调用；当调用 `fill_context_for_report` 时，在运行时上下文标记 `report = True`；
   - `log_before_model`（`before_model`）—— 模型调用前日志；
   - `report_prompt_switch`（`dynamic_prompt`）—— **动态提示词切换**：检测到 `report` 上下文为 `True` 时，将系统提示词替换为报告生成提示词。
4. **RAG 链路**（`rag/rag_service.py`）：检索 → 拼接参考资料 → 经提示词模板 + 模型 + 解析器生成最终回答。
5. **UI 层**（`app.py`）：Streamlit 维护会话状态，调用 `agent.execute_stream` 实现流式输出。

---

## ⚠️ 注意事项

- 本项目为**演示 / 教学用途**，部分工具返回随机模拟数据，生产环境需替换为真实接口。
- 首次运行前务必执行 `python -m rag.vector_store` 构建索引，否则 RAG 检索为空。
- 向量库与日志为生成产物，已忽略；如需重置知识库，删除 `rag/chroma_db/` 后重新构建即可。

---

## 📄 许可证

暂未指定许可证。如需开源，请补充 `LICENSE` 文件（建议 MIT）。
