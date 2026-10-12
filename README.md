# LocalRAG

基于 LangChain + Qdrant + Ollama 的本地化知识库智能问答系统。上传文档后即可基于文档内容进行问答，全程本地运行，数据不出域。

## 技术栈

| 层级 | 技术 |
|---|---|
| Web 框架 | FastAPI + Uvicorn |
| RAG 编排 | LangChain（LCEL 链式调用） |
| 向量数据库 | Qdrant（本地磁盘持久化模式） |
| 嵌入模型 | `bge-small-zh-v1.5`（中文优化，通过 Ollama） |
| 大语言模型 | `qwen2:7b`（通过 Ollama 本地推理） |
| 文档解析 | PyPDF / python-docx / TextLoader |

## 功能特性

- **文档入库**：支持 TXT / PDF / DOCX，自动解析、中文标点优化切分（chunk 500，重叠 50）、向量化存储
- **智能问答**：基于检索到的 Top3 文档片段生成回答，严格限定在已有资料范围内，资料不足时明确返回"根据现有资料无法回答该问题"
- **来源追溯**：回答同时返回引用来源文件名及内容片段，按文件去重
- **RESTful API**：提供文档上传、问答、健康检查接口，自带 Swagger UI
- **全本地部署**：模型、向量库、服务均运行在本地，无需外部 API 调用

## 项目结构

```
local-rag-system/
├── main.py                  # FastAPI 入口，CORS、全局异常处理、服务初始化
├── requirements.txt         # Python 依赖
├── test_demo.py             # 端到端测试脚本
├── data/                    # 测试文档目录
├── app/
│   ├── api/
│   │   └── endpoints.py     # /api/upload、/api/chat、/api/health
│   ├── core/
│   │   └── config.py        # 集中配置（模型、向量库、切分参数等）
│   ├── rag/
│   │   ├── chain.py         # RAG 问答链（LCEL）、来源去重
│   │   ├── vector_store.py  # Qdrant 向量库管理、文档加载/切分/入库
│   │   └── qa_chain.py
│   └── utils/
│       └── document_loader.py
└── storage/                 # Qdrant 本地持久化数据（git 已忽略）
```

## 快速开始

### 环境要求

- Python 3.10+
- [Ollama](https://ollama.com/) 已安装并运行
- 已拉取所需模型：
  ```bash
  ollama pull quentinz/bge-small-zh-v1.5
  ollama pull qwen2:7b
  ```

### 安装与运行

```bash
cd local-rag-system

# 创建虚拟环境
python -m venv .venv
.venv\Scripts\activate

# 安装依赖
pip install -r requirements.txt

# 启动服务
python main.py
```

服务启动后访问 http://localhost:8004/docs 打开 Swagger UI 进行测试。

### API 接口

| 方法 | 路径 | 说明 |
|---|---|---|
| POST | `/api/upload` | 上传文档（TXT/PDF/DOCX），自动解析入库 |
| POST | `/api/chat` | 提问，返回回答及引用来源 |
| GET | `/api/health` | 健康检查 |

**问答示例：**
```bash
curl -X POST "http://localhost:8004/api/chat" \
  -H "Content-Type: application/json" \
  -d '{"question":"什么是 RAG？"}'
```

**响应示例：**
```json
{
  "answer": "RAG（检索增强生成）是一种...",
  "sources": [
    {"source": "rag_intro.txt", "content": "..."}
  ]
}
```

## 配置说明

所有配置集中在 `app/core/config.py`，可通过 `.env` 文件覆盖：

| 配置项 | 默认值 | 说明 |
|---|---|---|
| `QDRANT_PATH` | `./storage/qdrant_local` | Qdrant 本地存储路径 |
| `COLLECTION_NAME` | `rag_knowledge_base` | 向量集合名 |
| `EMBEDDING_MODEL` | `quentinz/bge-small-zh-v1.5` | 嵌入模型 |
| `OLLAMA_LLM_MODEL` | `qwen2:7b` | 大语言模型 |
| `LLM_TEMPERATURE` | `0.1` | 生成温度 |
| `CHUNK_SIZE` | `500` | 文本切分大小 |
| `CHUNK_OVERLAP` | `50` | 切分重叠 |
| `RETRIEVER_TOP_K` | `3` | 检索返回片段数 |

## 扩展方向

- **Java 双栈架构**：用 Spring Boot 做外层业务后端（用户管理、历史记录、权限控制），Python 单独提供 AI 推理服务，两者通过 HTTP 调用
- **多轮对话**：引入对话历史记忆，支持上下文追问
- **检索优化**：加入重排序（Reranker）、混合检索（关键词 + 向量）
- **前端界面**：配套 Web 聊天界面，支持文档管理与对话展示
