# WyRadar MVP 技术栈方案

## 1. 选型原则

| 原则 | 说明 |
|------|------|
| Python 优先 | 团队主力语言，AI/ML 生态最完善 |
| MVP 够用不铺张 | 每个组件选能满足当前需求的最简方案，预留平滑升级路径 |
| 异步优先 | 系统核心瓶颈在 I/O（API 调用、网络采集），异步架构可最大化吞吐 |
| 单仓库可部署 | `docker compose up` 一条命令即可运行，不依赖外部服务 |
| 国产模型优先 | LLM 选 DeepSeek，中文场景有优势，成本可控 |

---

## 2. 技术栈总览

```
┌──────────────────────────────────────────────────────────┐
│                      WyRadar MVP 技术栈                    │
├──────────────┬──────────────────┬────────────────────────┤
│ 前端          │ 后端              │ 基础设施                │
│              │                  │                        │
│ Vue 3        │ FastAPI          │ Docker Compose         │
│ Vite 5       │ LangGraph        │ SQLite                 │
│ TypeScript   │ DeepSeek-v4-pro  │ ChromaDB               │
│ Tailwind CSS │ SQLAlchemy 2.x   │ APScheduler            │
│              │ httpx            │ lark-oapi              │
└──────────────┴──────────────────┴────────────────────────┘
```

---

## 3. 详细选型

### 3.1 后端框架：FastAPI

| 维度 | 选型理由 |
|------|---------|
| 异步原生 | 基于 Starlette + asyncio，与 LangGraph async 模式、httpx 异步采集天然匹配 |
| 自动文档 | OpenAPI / Swagger UI 自动生成，前后端联调无需手写接口文档 |
| 类型安全 | Pydantic v2 做请求/响应校验，字段类型错误在开发阶段暴露 |
| 生态成熟 | 与 SQLAlchemy、APScheduler、lark-oapi 均有成熟的集成方案 |
| 部署简单 | `uvicorn main:app` 单进程即可运行，MVP 不引入 Gunicorn |

**版本：** FastAPI 0.115+, Pydantic v2, uvicorn 0.34+

**备选方案对比：**

| 方案 | 不选的理由 |
|------|-----------|
| Flask | 同步模型，与 LangGraph async 不匹配，缺少自动 API 文档 |
| Django REST | 过重，ORM/模板/中间件大量用不上，学习成本高 |
| Litestar | API 风格类似但社区和文档不如 FastAPI 成熟 |

---

### 3.2 多智能体框架：LangGraph

| 维度 | 选型理由 |
|------|---------|
| 有向图编排 | StateGraph 将采集→策展→分析→报告→分发建模为 DAG，流程可观测可调试 |
| 显式 State | AgentState 在节点间流转，每一步的输入输出可追踪，排查问题直观 |
| 并行支持 | Send API 支持多节点并行，4 信源同步采集、N 条情报并行分析 |
| 流式输出 | `astream()` 原生支持，FastAPI SSE 推送到前端展示实时进度 |
| 中断恢复 | MemorySaver checkpointer 支持状态持久化，节点失败可从中断处恢复 |
| 生态 | LangChain 生态，与 ChatOpenAI（DeepSeek 兼容）、ChromaDB 无缝集成 |

**版本：** LangGraph 0.3+, LangChain 0.3+, langchain-openai 0.3+

**备选方案对比：**

| 方案 | 不选的理由 |
|------|-----------|
| CrewAI | 线性顺序任务模型，不支持条件分支和并行 Send API，状态管理不透明 |
| AutoGen | 对话驱动编排，适合代码生成和动态协商场景，不适合确定的流水线 |
| 自建 | 重复造轮子，LangGraph 的 checkpointer、streaming 等基础设施无需重写 |

---

### 3.3 LLM：DeepSeek-v4-pro

| 维度 | 选型理由 |
|------|---------|
| 上下文窗口 | 1M token，轻松处理长论文（arXiv 全文）、大段 README、多源对比 |
| API 兼容 | 完全兼容 OpenAI SDK 格式，`langchain-openai.ChatOpenAI` 改 base_url 即可接入 |
| 中文能力 | 国产模型，对知乎、公众号、机器之心等中文信源的理解优于海外模型 |
| 成本 | 比 Claude Opus / GPT-4o 便宜一个数量级，日均分析 15-30 条情报成本可控 |
| 温度控制 | 支持 temperature=0.1~0.3 低温度，保证评分一致性和摘要质量 |

**接入方式：**

```python
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(
    model="deepseek-chat",
    api_key="sk-xxx",
    base_url="https://api.deepseek.com/v1",
    temperature=0.2,
    max_tokens=4096,
)
```

**使用场景与参数：**

| 场景 | temperature | max_tokens | 说明 |
|------|------------|------------|------|
| L2 分类 + L3 去噪 | 0.1 | 1024 | 需要稳定一致的判断 |
| 摘要 + 四维评分 | 0.2 | 4096 | 需要结构化 JSON 输出 |
| 日报润色 | 0.4 | 2048 | 允许一定语言多样性 |

---

### 3.4 前端框架：Vue 3 + Vite + TypeScript

| 维度 | 选型理由 |
|------|---------|
| 体积 | Vue 3 运行时 ~30KB gzip，首屏加载快，MVP 不需要引入全量框架 |
| 开发效率 | 单文件组件（SFC），模板/逻辑/样式一体化，新人半天上手 |
| Vite | 秒级 HMR（热更新），开发体验极好，TypeScript 原生支持 |
| 解耦 | 纯静态文件，FastAPI 只提供 REST API，前后端独立开发独立部署 |
| 渐进式 | MVP 仅 3 个页面，不引入 Vue Router 和 Pinia；后续需要时可加 |
| 生态 | Naive UI / Tailwind CSS 社区方案成熟 |

**版本：** Vue 3.5+, Vite 5+, TypeScript 5.5+, Tailwind CSS 3.4+ 或 Naive UI 2.40+

**备选方案对比：**

| 方案 | 不选的理由 |
|------|-----------|
| React | JSX 学习成本高于 Vue 模板语法，打包体积更大 |
| Streamlit | 交互能力有限（筛选/分页/实时进度），不适合生产级 Web 页面 |
| 纯 HTML + jQuery | 组件化开发困难，代码维护性差 |
| Next.js / Nuxt | SSR 对纯 SPA 场景过度，增加构建复杂度 |

**组件库选择：Naive UI > Tailwind CSS**

| 方案 | 优势 | 劣势 |
|------|------|------|
| Naive UI | Vue 3 原生，Tree Shaking，组件齐全（Table/DatePicker/Modal/Progress） | 包体积 ~200KB gzip（按需引入实际更小） |
| Tailwind CSS | 完全设计自由度，极小体积 | 无预制组件，Table/DatePicker 需手写或引入 Headless UI |

推荐 MVP 用 **Naive UI** 快速搭建 Table/DatePicker/Modal/Progress，后续有定制设计需求再引入 Tailwind CSS。

---

### 3.5 关系数据库：SQLite（ORM: SQLAlchemy 2.x async）

| 维度 | 选型理由 |
|------|---------|
| 零部署 | 单文件数据库，Docker 挂载 volume 即可持久化，不需要独立容器 |
| 够用 | 日均写入几十条情报、几百条分析记录，SQLite 的读写能力完全满足 |
| 升级路径 | SQLAlchemy ORM 抽象，切换 PostgreSQL 仅需更换 `DATABASE_URL` |
| async | SQLAlchemy 2.x 原生支持 asyncio，与 FastAPI `async/await` 一致 |

**版本：** SQLAlchemy 2.0+, aiosqlite 0.20+

**何时切换 PostgreSQL：**
- 日均情报量 > 500 条
- 并发写入场景增多（多用户同时触发）
- 需要全文检索（PostgreSQL `tsvector`）

---

### 3.6 向量数据库：ChromaDB（嵌入式模式）

| 维度 | 选型理由 |
|------|---------|
| 嵌入式 | `chromadb.PersistentClient(path="./data/chroma")` 零部署，与 SQLite 同理 |
| Python 原生 | `pip install chromadb` 即用，不依赖 Docker 或外部服务 |
| LangChain 集成 | `langchain-chroma` 无缝接入，向量化→存储→检索一条链 |
| 轻量 | 日均 15-30 条向量写入，ChromaDB 完全够用 |

**版本：** chromadb 0.5+, langchain-chroma 0.2+

**Embedding 模型：** `text-embedding-3-small`（OpenAI 兼容 API，512 维）或 `bge-large-zh-v1.5`（本地部署，1024 维）。MVP 推荐前者（API 调用即可，无需 GPU）。

**备选方案对比：**

| 方案 | 不选的理由 |
|------|-----------|
| Qdrant / Weaviate / Milvus | 需要独立容器部署，MVP 阶段过度 |
| FAISS | 仅内存索引，不支持持久化，重启丢失 |

---

### 3.7 定时调度：APScheduler

| 维度 | 选型理由 |
|------|---------|
| 内嵌 | 与 FastAPI 同进程运行，不需要独立 Worker 容器 |
| Cron 语法 | 标准 Cron 表达式，清晰直观（`0 7 * * *`） |
| 轻量 | 两个定时任务（7:00 采集 + 8:00 推送），APScheduler 完全够用 |

**版本：** APScheduler 3.10+

**备选方案对比：**

| 方案 | 不选的理由 |
|------|-----------|
| Celery | 需要 Redis/RabbitMQ Broker，MVP 阶段基础设施过重 |
| Linux Cron | 依赖宿主机，与 Docker 容器化理念冲突 |
| schedule（Python 库） | 不支持 Cron 表达式，功能太弱 |

---

### 3.8 HTTP 客户端：httpx (async)

| 维度 | 选型理由 |
|------|---------|
| async | 原生 asyncio 支持，与 FastAPI 异步路由匹配 |
| 连接池 | 内置连接池复用，4 源并行采集时 TCP 连接不重复建立 |
| API 风格 | 与 requests 几乎一致，学习成本为零 |

**版本：** httpx 0.27+

---

### 3.9 飞书 SDK：lark-oapi

| 维度 | 选型理由 |
|------|---------|
| 官方 | 飞书开放平台官方 Python SDK，API 覆盖完整 |
| 类型安全 | 请求/响应模型为 dataclass，字段有类型提示 |

**版本：** lark-oapi 1.4+

---

### 3.10 部署：Docker Compose

| 维度 | 选型理由 |
|------|---------|
| 单仓库 | `docker compose up` 一条命令启动全部服务 |
| 环境一致 | 开发/测试/生产环境完全一致 |
| 数据持久化 | volume 挂载 SQLite + ChromaDB，容器重启不丢数据 |

**容器规划：**

| 容器 | 端口 | 职责 |
|------|------|------|
| FastAPI | 8000 | 后端 API + LangGraph Agent 引擎 + APScheduler + ChromaDB |
| Nginx | 80 | 代理前端静态文件 + 反向代理 `/api/*` 到 FastAPI |

MVP 阶段可选不单独部署 Nginx，由 FastAPI 通过 `StaticFiles` 中间件直接托管前端构建产物。

---

## 4. 依赖清单

### 4.1 Python (backend/requirements.txt)

```
# Web 框架
fastapi==0.115.*
uvicorn[standard]==0.34.*
pydantic==2.*

# AI/Agent
langgraph==0.3.*
langchain==0.3.*
langchain-openai==0.3.*
langchain-chroma==0.2.*
chromadb==0.5.*

# 数据库
sqlalchemy[asyncio]==2.0.*
aiosqlite==0.20.*

# 采集
httpx==0.27.*
feedparser==6.*      # arXiv RSS 解析

# 飞书
lark-oapi==1.4.*

# 调度
apscheduler==3.10.*

# 工具
python-dotenv==1.*   # 环境变量管理
```

### 4.2 Node.js (frontend/package.json)

```json
{
  "dependencies": {
    "vue": "^3.5",
    "naive-ui": "^2.40"
  },
  "devDependencies": {
    "@vitejs/plugin-vue": "^5",
    "typescript": "^5.5",
    "vite": "^5",
    "vue-tsc": "^2"
  }
}
```

---

## 5. 技术风险与缓解

| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|---------|
| DeepSeek API 不稳定或限流 | 中 | 高 | 重试机制（max 3 次）；langgraph MemorySaver 支持中断恢复 |
| 信源 API 限流/变更 | 中 | 中 | 每源独立 try-catch，单源失败不影响整体；关键源备用爬虫方案 |
| ChromaDB 大规模数据下性能下降 | 低 | 低 | MVP 日均几十条，无此问题；后续可迁移至 Qdrant |
| SQLite 并发写入瓶颈 | 低 | 低 | MVP 单用户触发，无并发写；后续可升级 PostgreSQL |
| 飞书 API 推送限流（群消息 10KB 限制） | 中 | 低 | 群消息只推概览+强推摘要，完整内容放飞书文档 |

---

## 6. 升级路径总览

| 组件 | MVP | 下一个迭代 | 远期 |
|------|-----|-----------|------|
| LLM | DeepSeek-v4-pro | — | 可评估多模型混合（摘要用 DeepSeek，评分用 Claude） |
| 关系数据库 | SQLite | — | PostgreSQL |
| 向量数据库 | ChromaDB（嵌入式） | — | Qdrant（独立部署） |
| 前端 | Vue 3 SPA（3 页，无路由） | 引入 Vue Router + Pinia | 引入 ECharts 可视化 |
| 调度 | APScheduler（内嵌） | — | Celery + Redis（如需复杂调度） |
| 部署 | Docker Compose（2 容器） | — | Kubernetes（如需高可用） |
| 认证 | 无（内网部署） | 简单 Token 认证 | OAuth2 / 飞书登录 |
