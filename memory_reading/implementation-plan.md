# WyRadar MVP 实施计划

## 总览

- **总工期：** 19-24 天
- **人力：** 1-2 名 AI 开发者
- **分支策略：** 每个 Phase 在独立分支开发，Phase 完成后合入 master
- **步骤粒度：** 每个步骤可在 1-4 小时内完成并独立验证

---

## Phase 1：基础设施（3-4 天）

### 步骤 1.1：初始化项目仓库与目录结构

**目标：** 创建 backend/ 和 frontend/ 两个独立目录，配置各自包管理。

**指令：**
1. 在项目根目录创建 backend/ 目录和 frontend/ 目录
2. 在 backend/ 下创建空的 requirements.txt
3. 按照 tech_stack.md 第 4.1 节列出的依赖清单，将每个包及版本号写入 requirements.txt
4. 按照 tech_stack.md 第 4.2 节的 package.json 内容，在 frontend/ 下初始化 Vue 3 + Vite + TypeScript 项目
5. 在 backend/ 下创建以下空子目录：api/、graph/nodes/、graph/tools/、models/
6. 在 frontend/src/ 下创建以下空子目录：api/、views/、components/、styles/
7. 在项目根目录创建 data/ 和 logs/ 目录，添加到 .gitignore

**验证：**
- 执行 `pip install -r backend/requirements.txt`，所有依赖安装成功无报错
- 执行 `cd frontend && npm install`，所有依赖安装成功无报错
- 目录结构与 design_document.md 第 8.3 节完全一致

---

### 步骤 1.2：创建 Docker Compose 编排

**目标：** 编写 docker-compose.yml，实现 FastAPI 容器一键启动，挂载 data 和 logs 卷。

**指令：**
1. 在项目根目录创建 docker-compose.yml
2. 定义 FastAPI 服务：基于 python:3.11-slim 镜像，端口映射 8000:8000，挂载 ./data:/data 和 ./logs:/logs 卷，启动命令为 uvicorn main:app --host 0.0.0.0 --port 8000 --reload
3. 可选定义 Nginx 服务（注释掉，标注 MVP 阶段可选）
4. 在 backend/ 下创建 Dockerfile：基于 python:3.11-slim，安装 requirements.txt，设置 PYTHONUNBUFFERED=1

**验证：**
- 执行 `docker compose up`，FastAPI 容器启动（暂时无 main.py 会报错退出，确认容器能正常启动即可）
- 执行 `docker compose down`，容器正常停止

---

### 步骤 1.3：创建 FastAPI 入口与配置管理

**目标：** 创建 main.py 和 config.py，启动可访问的 FastAPI 服务。

**指令：**
1. 创建 backend/config.py，实现配置管理功能：从 .env 文件和环境变量读取以下配置项——DEEPSEEK_API_KEY、DEEPSEEK_BASE_URL（默认 https://api.deepseek.com/v1）、FEISHU_APP_ID、FEISHU_APP_SECRET、DATABASE_URL（默认 sqlite+aiosqlite:///./data/wyradar.db）、CHROMA_PERSIST_PATH（默认 ./data/chroma）、CRON_COLLECT（默认 0 7 * * *）、CRON_DISPATCH（默认 0 8 * * *）、KEYWORDS_CONFIG_PATH（默认 ./config/keywords.yaml）
2. 创建 backend/main.py，初始化 FastAPI 应用，设置 CORS 中间件（允许前端开发服务器 localhost:5173），挂载 /api 前缀的路由器，添加根路径 / 返回服务健康状态 JSON
3. 创建 backend/.env.example 文件，列出所有必需的环境变量及示例值

**验证：**
- 执行 `uvicorn backend.main:app --port 8000`，服务启动
- 浏览器访问 http://localhost:8000/，返回 JSON `{"status": "ok", "service": "WyRadar"}`
- 访问 http://localhost:8000/docs，Swagger UI 正常显示（暂时无自定义路由）

---

### 步骤 1.4：建立 SQLAlchemy 数据模型

**目标：** 创建 database.py，定义 4 个核心表并执行建表迁移。

**指令：**
1. 创建 backend/models/database.py，实现异步 SQLAlchemy 引擎和会话管理：使用 create_async_engine 连接 DATABASE_URL，创建 async_sessionmaker，定义 Base 基类，提供 get_db 异步上下文管理器
2. 在 database.py 中按 design_document.md 第 6.1 节定义 4 个 ORM 模型类：RawItem、AnalyzedItem、DailyReport、ExecutionLog
3. RawItem 表必须包含这些列：id(String, PK)、source(String)、title(String)、url(String)、description(Text)、author(String)、raw_tags(Text, JSON)、stars_count(Integer)、fetched_at(DateTime)、content_snapshot(Text, JSON)、status(String, default='pending')、filter_reason(Text)、category_l1(String)、category_l2(String)、curated_at(DateTime)、created_at(DateTime)
4. AnalyzedItem 表必须包含这些列：id(String, PK)、raw_item_id(String, FK→raw_items.id, unique)、summary_one_liner(Text)、summary_highlights(Text, JSON)、summary_comparison(Text)、weight_adjustment(String)、score_business(Integer, 1-5)、score_business_reason(Text)、score_business_confidence(Float)、score_deploy(Integer, 1-5)、score_deploy_reason(Text)、score_deploy_confidence(Float)、score_performance(Integer, 1-5)、score_performance_reason(Text)、score_performance_confidence(Float)、score_compatibility(Integer, 1-5)、score_compatibility_reason(Text)、score_compatibility_confidence(Float)、score_total(Float)、confidence_overall(Float)、recommend_level(String)、rag_context_used(Text, JSON)、llm_model_used(String)、analyzed_at(DateTime)、created_at(DateTime)
5. DailyReport 表必须包含这些列：id(String, PK)、report_date(Date, unique)、run_id(String)、total_fetched(Integer)、total_curated(Integer)、total_analyzed(Integer)、recommend_high(Integer)、recommend_mid(Integer)、recommend_low(Integer)、report_markdown(Text)、report_status(String, default='complete')、feishu_msg_id(Text)、feishu_doc_url(Text)、execution_time_seconds(Integer)、created_at(DateTime)
6. ExecutionLog 表必须包含这些列：id(String, PK)、run_id(String, indexed)、trigger(String)、node_name(String)、status(String)、items_processed(Integer, default=0)、error_message(Text)、started_at(DateTime)、finished_at(DateTime)、created_at(DateTime)
7. 创建 backend/models/__init__.py，导出所有模型类和 get_db

**验证：**
- 编写并执行一个独立的初始化脚本，调用 `Base.metadata.create_all`，无报错
- 使用 sqlite3 命令行工具打开 ./data/wyradar.db，执行 `.tables`，确认 raw_items、analyzed_items、daily_reports、execution_logs 四个表存在
- 执行 `.schema raw_items`，确认列定义与设计一致（特别检查 CHECK 约束和 JSON 字段）
- 删除 ./data/wyradar.db，再次执行初始化脚本，确认可重复建表

---

### 步骤 1.5：搭建 LangGraph 基础框架

**目标：** 创建 state.py、graph.py，定义 AgentState 和空的 StateGraph，验证图可编译和调用。

**指令：**
1. 创建 backend/graph/state.py，定义 AgentState TypedDict，字段与 design_document.md 第 3.2 节完全一致：run_id(str)、trigger(str)、current_stage(str)、raw_items(List[dict])、fetch_errors(List[str])、curated_items(List[dict])、analyses(List[dict])、analysis_errors(List[str])、report_markdown(str)、report_summary(dict)、feishu_msg_id(str)、feishu_doc_url(str)、error(Optional[str])、status(str)
2. 创建 backend/graph/graph.py，实现 build_wyradar_graph 函数：创建 StateGraph(AgentState)，添加 5 个占位节点（collector/curator/analyst/editor/dispatcher），每个节点函数暂为 pass（直接 return state），按 collector→curator→analyst→editor→dispatcher→END 添加边，编译时使用 MemorySaver checkpointer
3. 创建 backend/graph/__init__.py，导出 build_wyradar_graph
4. 创建 backend/graph/nodes/__init__.py（空）
5. 创建 backend/graph/tools/__init__.py（空）

**验证：**
- 编写并执行一个独立测试脚本：调用 build_wyradar_graph() 获取编译后的 graph
- 构造一个最小 AgentState（所有字段为默认值），调用 `graph.invoke(initial_state)`，确认执行无报错，返回的 state 与输入一致
- 检查 MemorySaver checkpointer 正常工作，使用同一 thread_id 多次调用，确认状态可持久化

---

### 步骤 1.6：接入 DeepSeek LLM

**目标：** 创建 llm.py，封装 DeepSeek API 调用，验证一次简单的 LLM 调用。

**指令：**
1. 创建 backend/graph/tools/llm.py，实现 get_llm 工厂函数：返回 ChatOpenAI 实例，model 参数为 deepseek-chat，base_url 从 config 读取，api_key 从 config 读取，默认 temperature=0.2、max_tokens=4096
2. 同文件中实现三个场景特定的 LLM 创建函数：get_classification_llm(temperature=0.1, max_tokens=1024)、get_analysis_llm(temperature=0.2, max_tokens=4096)、get_editor_llm(temperature=0.4, max_tokens=2048)
3. 确保 LLM 支持结构化 JSON 输出（使用 with_structured_output 或 response_format 参数）

**验证：**
- 编写并执行一个独立测试脚本：调用 get_llm() 获取 LLM 实例
- 发送一条简单消息（如"请用 JSON 格式返回：{'answer': 'yes'}")，确认返回内容可解析为有效 JSON
- 发送一条 2000 字的长输入，确认上下文处理正常，无截断
- 如果 API 返回错误，确认错误信息可读（不抛出裸 traceback）

---

### 步骤 1.7：初始化 Vue 3 前端骨架

**目标：** 搭建 Vue 3 + Vite + TypeScript + Naive UI 的前端骨架，三个页面 Tab 切换正常。

**指令：**
1. 在 frontend/src/ 下创建 App.vue，包含顶部导航栏（WyRadar 标题 + 三个 Tab 按钮：今日日报/历史检索/手动触发），使用 Naive UI 的 n-tabs 组件，默认选中第一个 Tab，底部使用条件渲染显示对应页面组件
2. 创建 frontend/src/views/DailyReport.vue，内容仅显示占位文本"今日日报"
3. 创建 frontend/src/views/HistorySearch.vue，内容仅显示占位文本"历史检索"
4. 创建 frontend/src/views/TriggerRun.vue，内容仅显示占位文本"手动触发"
5. 创建 frontend/src/api/index.ts，定义 API_BASE_URL 常量（默认 http://localhost:8000/api），导出通用的 fetch 封装函数（GET/POST，自动 JSON 解析，错误处理）
6. 配置 Vite 开发代理：将 /api 前缀的请求代理到 localhost:8000
7. 确保 main.ts 正确注册 Naive UI 组件库

**验证：**
- 执行 `cd frontend && npm run dev`，Vite 启动无报错
- 浏览器访问 http://localhost:5173，页面显示导航栏和"今日日报"Tab 内容
- 点击"历史检索"Tab，切换到对应占位文本
- 点击"手动触发"Tab，切换到对应占位文本
- 三个 Tab 之间切换无报错、无白屏

---

### 步骤 1.8：创建关键词配置文件

**目标：** 创建 keywords.yaml 配置文件，供 Curator 节点的 L1 关键词过滤使用。

**指令：**
1. 在项目根目录创建 config/ 目录
2. 创建 config/keywords.yaml 文件，定义以下结构：一个 target_keywords 列表，包含至少以下关键词（Agent、LLM、Multimodal、NLP、Transformer、RLHF、RAG、MCP、function-calling、tool-use、大模型、多模态、智能体、对齐、推理、微调、预训练、提示工程）。将关键词按语言分组（en 和 zh）
3. 确保配置文件支持注释，每个关键词旁备注该词覆盖的场景（如"MCP — Model Context Protocol 相关项目和论文"）
4. 创建 backend/config/keywords.py，实现 read_keywords 函数：读取 YAML 文件，返回扁平化的关键词列表

**验证：**
- 编写并执行一个独立测试脚本：调用 read_keywords()，确认返回的关键词列表包含至少 18 个词
- 修改 keywords.yaml 新增一个关键词"扩散模型"，再次读取，确认新词出现在列表中
- 删除 keywords.yaml 文件后执行读取，确认抛出明确的 FileNotFoundError 并附路径信息

---

## Phase 2：采集 + 策展节点（4-5 天）

### 步骤 2.1：实现 GitHub Trending 采集器

**目标：** 创建 fetchers.py 中的 fetch_github_trending 函数，抓取 GitHub Trending Python 仓库列表。

**指令：**
1. 创建 backend/graph/tools/fetchers.py
2. 实现 fetch_github_trending 函数：
   - 使用 httpx 异步客户端调用 GitHub REST API：`GET /search/repositories?q=language:python+topic:machine-learning&sort=stars&order=desc&per_page=20`
   - 从响应中提取每条 repo 的字段：id(全名hash)、source(固定为"github")、title(full_name)、url(html_url)、description、author(owner.login)、raw_tags(topics列表)、stars_count(stargazers_count)
   - 设置请求头 User-Agent 为 WyRadar/1.0
   - 设置超时时间为 30 秒
   - 异常时返回空列表并记录错误日志
3. 实现 fetch_github_trending_daily 函数，使用 GitHub Trending 非官方端点（`https://api.github.com/search/repositories?q=created:>{昨日日期}&sort=stars&order=desc&per_page=20`），处理可能的 403 限流（返回空列表，不抛异常）

**验证：**
- 编写并执行一个独立测试脚本：调用 fetch_github_trending()，确认返回列表长度 ≤ 20
- 确认返回的每个条目都包含 id/source/title/url/description 五个必填字段，无空值
- 确认 source 字段值全部为 "github"
- 模拟网络异常场景：临时断开网络或设置错误的 API 端点，确认函数返回空列表而非抛异常

---

### 步骤 2.2：实现 HuggingFace 采集器

**目标：** 实现 fetch_hf_trending 函数，抓取 HuggingFace Trending 模型列表。

**指令：**
1. 在 fetchers.py 中实现 fetch_hf_trending 函数
2. 使用 HuggingFace Hub API：`GET https://huggingface.co/api/models?sort=downloads&direction=-1&limit=20`
3. 从响应中提取每条 model 的字段：id(modelId hash)、source("huggingface")、title(modelId)、url(`https://huggingface.co/{modelId}`)、description(pipeline_tag)、author(author)、raw_tags(tags 列表)、stars_count(downloads 或 likes)
4. 同上设置超时和异常处理

**验证：**
- 调用 fetch_hf_trending()，返回列表长度 ≥ 1（HF API 稳定）
- 确认返回条目中至少一条包含 author 和 tags 非空
- 确认 url 字段格式为 `https://huggingface.co/{modelId}`

---

### 步骤 2.3：实现 ModelScope 采集器

**目标：** 实现 fetch_modelscope_trending 函数，抓取魔搭社区热门模型。

**指令：**
1. 在 fetchers.py 中实现 fetch_modelscope_trending 函数
2. 使用 ModelScope 开放 API：`GET https://modelscope.cn/api/v1/models?sort=trending&limit=20`
3. 从响应中提取字段：id、source("modelscope")、title、url、description、author、raw_tags、stars_count
4. 同上设置超时和异常处理

**验证：**
- 调用 fetch_modelscope_trending()，确认返回列表不为空（确认 API 可用）
- 确认返回条目中每条 url 格式为 `https://modelscope.cn/models/{...}`

---

### 步骤 2.4：实现 arXiv 采集器

**目标：** 实现 fetch_arxiv_papers 函数，抓取前一日 AI 相关分类论文。

**指令：**
1. 在 fetchers.py 中实现 fetch_arxiv_papers 函数
2. 使用 arXiv API：`http://export.arxiv.org/api/query?search_query=cat:cs.AI+OR+cat:cs.CL+OR+cat:cs.CV+OR+cat:cs.LG&sortBy=submittedDate&sortOrder=descending&start=0&max_results=20`
3. 使用 feedparser 解析 Atom 格式响应
4. 从每个 entry 提取字段：id(entry.id)、source("arxiv")、title(entry.title)、url(entry.id)、description(entry.summary)、author(第一个 author.name)、raw_tags(entry.tags 列表)、stars_count(None——arXiv 无此概念)
5. 同上设置超时和异常处理

**验证：**
- 调用 fetch_arxiv_papers()，确认返回列表长度 ≥ 15（arXiv API 非常稳定）
- 确认返回的每条 url 以 arxiv.org/abs/ 开头
- 确认 title 和 description 字段非空
- 确认 source 字段全部为 "arxiv"

---

### 步骤 2.5：实现去重与存储逻辑

**目标：** 创建去重函数 deduplicate_items 和批量入库函数 bulk_insert_raw_items。

**指令：**
1. 在 backend/graph/tools/ 下创建 storage.py
2. 实现 deduplicate_items 函数：输入为 raw_items 列表，基于 SHA256(item['url'] + item['title']) 生成唯一 ID，去重保留最新（同一 ID 只保留一条），返回去重后列表并为每条设置 id 字段
3. 实现 bulk_insert_raw_items 异步函数：接收去重后的列表，对每条执行 INSERT OR IGNORE（如果 id 已存在则跳过），设置 fetched_at 为当前时间、status 为 'pending'
4. 实现 get_pending_items 异步函数：返回所有 status='pending' 的 raw_items，按 created_at 升序排列

**验证：**
- 编写测试脚本：创建 3 条含重复项（第 2 条与第 1 条 url+title 相同）的测试数据，调用 deduplicate_items，确认返回 2 条
- 调用 bulk_insert_raw_items 写入 2 条去重后数据，直接查数据库确认 raw_items 表有 2 条记录且 status='pending'
- 再次写入相同的 2 条数据，确认不产生重复（仍为 2 条）
- 调用 get_pending_items，确认返回这 2 条数据

---

### 步骤 2.6：实现 Collector Node

**目标：** 实现 collector_node 函数，完成 4 源并行采集→去重→入库的完整逻辑。

**指令：**
1. 创建 backend/graph/nodes/collector.py
2. 实现 collector_node 函数：
   - 输入和输出均为 AgentState
   - 使用 concurrent.futures.ThreadPoolExecutor(max_workers=4) 并行调用 4 个 fetcher 函数
   - 收集所有返回结果，合并为 raw_items 大列表
   - 每个 fetcher 失败时将其名称+错误信息追加到 state['fetch_errors']
   - 单源失败不阻塞其他源
   - 调用 deduplicate_items 去重
   - 调用 bulk_insert_raw_items 入库
   - 设置 state['current_stage'] = 'curating'
3. 确保函数执行总耗时 < 60 秒（并行调用）

**验证：**
- 编写测试脚本：构造初始 AgentState，调用 collector_node(state)，确认 state['raw_items'] 非空（至少 30 条）
- 确认所有条目的 source 字段分布在 4 个信源中
- 确认 state['fetch_errors'] 中仅有真正失败的源（API 异常），不包含成功的源
- 确认数据库 raw_items 表中存在与 state['raw_items'] 数量一致的记录
- 故意停掉一个信源（如设置错误 URL），确认 collector_node 不抛异常，state['fetch_errors'] 记录该错误，其他 3 个信源数据正常入库

---

### 步骤 2.7：实现 Curator Node — L1 关键词过滤

**目标：** 实现 L1 关键词过滤逻辑。

**指令：**
1. 创建 backend/graph/nodes/curator.py
2. 实现 keyword_filter 独立函数：
   - 输入：raw_items 列表 + keywords 列表（从 keywords.yaml 读取）
   - 对每条 raw_item，检查其 title 和 description 是否包含任一关键词（大小写不敏感，中文精确匹配）
   - 匹配任意关键词→保留，无匹配→标记 status='filtered_out_keyword' 和 filter_reason='L1: no keyword match'
   - 返回 (通过列表, 淘汰列表) 二元组
3. 实现 batch_update_item_status 函数：批量更新 raw_items 表中条目的 status 和 filter_reason 字段

**验证：**
- 编写测试脚本：准备 5 条测试 raw_item（其中 3 条 title 含 Agent/大模型/LLM，2 条为"天气预测""股票分析"），调用 keyword_filter，确认通过 3 条、淘汰 2 条
- 确认淘汰条目 status 更新为 'filtered_out_keyword'
- 修改 keywords.yaml 新增"预测"，重新读取关键词并调用过滤，确认"天气预测"现在通过

---

### 步骤 2.8：实现 Curator Node — L2/L3 LLM 分类与去噪

**目标：** 实现 LLM 批量分类（L2）和去噪（L3）逻辑。

**指令：**
1. 在 curator.py 中实现 llm_batch_classify 函数：
   - 输入：L1 通过的 items 列表
   - 构造 Prompt 要求 LLM 对每条 item 做两件事：(1)判断是否属于 AI Agent/多模态大模型/NLP 三个领域之一，不属于则标记为"其他"；(2)打二级标签（新模型发布/论文/开源工具/框架/技术路线/观点/数据集/产品动态）
   - Prompt 要求 LLM 以 JSON 数组格式输出：[{"id": "...", "category_l1": "...", "category_l2": "...", "is_relevant": true/false}]
   - 使用 get_classification_llm(temperature=0.1)
   - 将分类结果写回对应 raw_item 的 category_l1、category_l2 字段
   - 不相关条目（is_relevant=false）标记 status='filtered_out_noise'、filter_reason='L2: not in target domains'
2. 实现 llm_batch_denoise 函数：
   - 输入：L2 通过的 items 列表
   - 构造 Prompt 要求 LLM 判断每条 item 是否为灌水/营销/与三个领域明确无关的内容
   - Prompt 给出灌水的典型特征：无具体技术内容、纯推广文案、重复率高、摘要空洞
   - LLM 以 JSON 数组输出判定结果
   - 灌水条目标记 status='filtered_out_noise'、filter_reason='L3: noise/spam'

**验证：**
- 准备 10 条真实 AI 相关和无关混合的条目，调用 L2 分类，确认分类准确率 ≥ 85%（抽样 100 条人工核对）
- 确认每条通过的条目 category_l1 和 category_l2 非空
- 准备 5 条灌水/营销条目，调用 L3 去噪，确认至少 4 条被正确识别为噪声
- 确认淘汰条目在数据库中的 status 和 filter_reason 字段正确

---

### 步骤 2.9：实现完整 Curator Node

**目标：** 组装 L1→L2→L3 三级过滤为 curator_node 函数。

**指令：**
1. 在 curator.py 中实现 curator_node 函数：
   - 输入和输出均为 AgentState
   - 依次调用：keyword_filter(L1) → llm_batch_classify(L2) → llm_batch_denoise(L3)
   - 将通过 L3 的条目赋值给 state['curated_items']
   - 设置 state['current_stage'] = 'analyzing'
   - 记录各级过滤数量到日志
2. 确保 curator_node 处理时间 < 30 秒（对 50-60 条输入）

**验证：**
- 调用 collector_node(state) 获取真实数据后，再调用 curator_node(state)，确认 state['curated_items'] 非空（10-20 条）
- 确认每条 curated_item 的 category_l1 属于 AI Agent/多模态大模型/NLP 之一
- 确认数据库 raw_items 表中，status 分布合理（curated 约 10-20 条，filtered_out_keyword 约 25-35 条，filtered_out_noise 约 5-10 条）

---

## Phase 3：分析 + RAG + 主编节点（5-6 天）

### 步骤 3.1：初始化 ChromaDB 并实现 Embedding 函数

**目标：** 创建 ChromaDB 持久化客户端，实现向量写入和检索函数。

**指令：**
1. 创建 backend/graph/tools/rag.py
2. 实现 init_chroma_client 函数：返回 chromadb.PersistentClient，路径从 config.CHROMA_PERSIST_PATH 读取，如果目录不存在则自动创建
3. 实现 get_or_create_collection 函数：获取或创建名为 "analyzed_items" 的 collection，设置 distance_metric 为 cosine
4. 实现 embed_text 函数：调用 text-embedding-3-small API（兼容 OpenAI 格式）将文本转为 512 维向量，使用 httpx 异步调用
5. 实现 index_analyzed_item 函数：接收 analyzed_item dict，将其 title + summary_one_liner + summary_highlights 拼接为 document 文本，调用 embed_text 向量化，写入 ChromaDB collection，metadata 包含 category_l1/category_l2/score_total/recommend_level/source/analyzed_at/url
6. 实现 hybrid_search 函数：
   - 输入：查询文本 + n_results=5
   - 调用 embed_text 获取查询向量
   - 执行 ChromaDB 向量相似度查询（获取 top n_results×2 候选）
   - 对候选结果使用 BM25Okapi 进行关键词重排（权重：向量 0.7 + BM25 0.3）
   - 返回最终 top-5，每项包含 id/metadata/similarity_score

**验证：**
- 调用 init_chroma_client() 和 get_or_create_collection()，确认返回的 collection 对象可用
- 构造一条测试 analyzed_item，调用 index_analyzed_item，确认无报错
- 调用 hybrid_search("multi agent framework")，首次运行返回空列表（无报错）
- 写入 3 条语义相近的测试数据后，再次搜索"多智能体协作"，确认返回至少 1 条相关结果
- 确认返回结果的 similarity_score 在 0-1 范围内

---

### 步骤 3.2：实现 Analyst Node — 分析 Prompt 模板

**目标：** 创建分析 Prompt 模板，包含 few-shot 示例和自适应权重指令。

**指令：**
1. 创建 backend/graph/tools/prompts.py
2. 实现 build_analysis_prompt 函数：
   - 输入：item dict（title/description/content_snapshot）+ rag_context list（历史相似情报 top-5）
   - 构造包含以下部分的完整 Prompt：
     a. System Prompt：定义角色为"资深 AI 技术分析师"，要求输出严格 JSON 格式
     b. 评分标准说明：四维度的 1-5 分锚定标准（与 design_document.md 第 5 节 F-03 完全一致）
     c. 上下文自适应权重规则：开源工具→落地难度 +0.10/学术论文→性能 +0.10/产品动态→商业 +0.10
     d. 3 个 Few-shot 示例：一个高分(4.5)、一个中分(3.2)、一个低分(1.8)，每个示例包含完整 JSON 输出，箭头标注每个评分的依据
     e. 当前待分析情报（title/url/description/content）
     f. RAG 上下文（top-5 相似历史情报的标题/评分/摘要）
     g. 输出格式要求：严格 JSON，包含 summary_one_liner/summary_highlights(3)/summary_comparison/weight_adjustment/scores(四维，每维含 value/confidence/reason)/score_total/confidence_overall/recommend_level/rag_context_used
3. 实现 build_classification_prompt 函数：用于 Curator L2 分类+L3 去噪的 Prompt 模板（与步骤 2.8 对应）
4. 实现 build_editor_prompt 函数：用于 Editor 日报润色的 Prompt 模板（与步骤 3.6 对应）

**验证：**
- 调用 build_analysis_prompt 并打印输出，人工检查 Prompt 包含：System Prompt/评分标准/自适应规则/3个示例/当前情报/RAG上下文/输出格式要求 七个部分
- 确认 Few-shot 示例中每条评分都有可追溯的具体依据（如"GitHub 3 天 5k star"而非"市场热度高"）
- 确认 1-5 分锚定标准每个分数都有明确的可操作边界（如"5=含详细 README、pip 即装"而非"5=很好"）

---

### 步骤 3.3：实现 Analyst Node — 单条分析函数

**目标：** 实现 analyze_single_item 函数，完成单条情报的 RAG 检索→LLM 分析→索引写入。

**指令：**
1. 创建 backend/graph/nodes/analyst.py
2. 实现 analyze_single_item 异步函数：
   - 输入：item dict + chroma_client
   - Step 1：调用 rag.hybrid_search(item['title'] + ' ' + item['description'])，获取 top-5 历史相似情报
   - Step 2：调用 prompts.build_analysis_prompt(item, rag_results)，生成完整 Prompt
   - Step 3：调用 get_analysis_llm() 获取 LLM 实例，发送 Prompt，要求输出 JSON
   - Step 4：解析 LLM 返回的 JSON，验证必填字段完整性（summary/4维评分/置信度/推荐等级），缺失字段则用默认值填充并记录警告
   - Step 5：调用 rag.index_analyzed_item(analysis_result)，将分析结果即时写入 ChromaDB
   - Step 6：返回解析后的 analysis dict
   - 异常处理：LLM 调用超时或 JSON 解析失败时，返回一个默认 analysis（score_total=0, recommend_level='error', 含错误信息），不抛异常
3. 实现 validate_analysis 函数：检查 analysis dict 中 4 个分数是否在 1-5 范围内，置信度是否在 0-1 范围内，recommend_level 是否为四个合法值之一

**验证：**
- 准备一条真实 curated_item，调用 analyze_single_item，确认返回的分析 dict 包含所有必填字段
- 确认 score_total 在 1.0-5.0 范围内，recommend_level 为四个合法值之一
- 确认 rag_context_used 字段非空列表（如果 ChromaDB 已有数据）
- 模拟 LLM 返回非法 JSON，确认 validate_analysis 检测到并返回标记为 error 的结果，不抛异常
- 模拟网络超时，确认函数在 30 秒超时后返回 error 标记的结果

---

### 步骤 3.4：实现 Analyst Node — 并行分析主函数

**目标：** 实现 analyst_node 函数，完成 N 条情报的并行分析和结果写入。

**指令：**
1. 在 analyst.py 中实现 analyst_node 函数：
   - 输入和输出均为 AgentState
   - 初始化 ChromaDB 客户端
   - 使用 concurrent.futures.ThreadPoolExecutor(max_workers=8) 并行调用 analyze_single_item
   - 每条 item 的分析结果追加到 state['analyses']，失败则追加到 state['analysis_errors']
   - 按 score_total 降序排列 state['analyses']
   - 批量写入 analyzed_items 表
   - 设置 state['current_stage'] = 'editing'
2. 确保 analysts_node 对 15 条情报的总耗时 < 3 分钟

**验证：**
- 调用 curator_node 获取真实数据后，再调用 analyst_node(state)，确认 state['analyses'] 数量 = state['curated_items'] 数量
- 确认 state['analyses'] 按 score_total 降序排列（第 1 条评分 ≥ 第 2 条）
- 确认数据库 analyzed_items 表中存在对应记录，score_total/confidence_overall/recommend_level 字段非空
- 如果某条分析失败（模拟 LLM 超时），确认 state['analysis_errors'] 记录该条目，其他条目正常完成

---

### 步骤 3.5：评分一致性验证

**目标：** 编写评分质量验证脚本，确保 LLM 评分可复现。

**指令：**
1. 创建 backend/tests/test_scoring_consistency.py
2. 实现以下测试逻辑：
   a. 选取 3 条典型情报（一条开源工具、一条学术论文、一条产品动态）
   b. 对每条情报调用 analyze_single_item 3 次
   c. 计算 3 次 score_total 的标准差
   d. 断言标准差 < 0.5
   e. 打印每次评分的四个维度分数和置信度
3. 输出验证报告到 logs/scoring_consistency_{date}.txt

**验证：**
- 执行验证脚本，确认 3 条测试情报的标准差均 < 0.5
- 如果某条标准差 ≥ 0.5，降低 temperature 到 0.1 并重新验证
- 打开输出报告，确认人工可读、含每次的三元组(value/confidence/reason)详细信息

---

### 步骤 3.6：实现 Editor Node

**目标：** 实现 editor_node 函数，汇总分析结果生成结构化日报 Markdown。

**指令：**
1. 创建 backend/graph/nodes/editor.py
2. 实现 build_report_summary 函数：统计 state['analyses'] 中的 total_fetched(len(state['raw_items']))、total_curated(len(state['curated_items']))、total_analyzed、recommend_high/mid/low 数量，返回 summary dict
3. 实现 build_report_markdown 函数：
   - 输入：summary dict + analyses 列表
   - 按推荐等级分组（强烈推荐/值得关注/暂不跟进/不推荐）
   - 对每组的每条情报，按日报模板填充字段（设计文档 §5 F-05），包括一句话总结/亮点/评分明细表（含置信度和⚠️标记）/RAG历史对比/链接
   - 调用 get_editor_llm(temperature=0.4) 对每组生成语言衔接和润色文案（不对评分结果做任何修改）
   - 拼接为完整 Markdown（含概览表→强烈推荐→值得关注→暂不跟进→全量列表）
4. 实现 editor_node 函数：
   - 输入和输出均为 AgentState
   - 调用 build_report_summary 和 build_report_markdown
   - 将结果写入 state['report_summary'] 和 state['report_markdown']
   - 写入 daily_reports 表（report_date 为今天日期）
   - 设置 state['current_stage'] = 'dispatching'
5. 确保日报生成耗时 < 30 秒

**验证：**
- 调用 analyst_node 获取真实数据后，再调用 editor_node(state)，确认 state['report_markdown'] 非空字符串（> 2000 字符）
- 确认生成的 Markdown 包含五个板块（今日概览/强烈推荐/值得关注/暂不跟进/全量列表）
- 确认"强烈推荐"板块中每条包含评分明细表（4 行×4 列：维度/分数/置信度/理由）
- 确认低置信度(<0.6)的维度在 Markdown 中标注 ⚠️ 符号
- 确认数据库 daily_reports 表中存在今日日期的记录

---

## Phase 4：分发节点 + 前端（5-6 天）

### 步骤 4.1：实现飞书工具模块

**目标：** 创建 feishu.py，封装飞书 Bot 消息和文档创建功能。

**指令：**
1. 创建 backend/graph/tools/feishu.py
2. 实现 get_feishu_client 函数：使用 lark-oapi SDK，通过 FEISHU_APP_ID 和 FEISHU_APP_SECRET 初始化客户端
3. 实现 build_feishu_message 函数：
   - 输入：report_summary dict + report_markdown str
   - 构造群消息内容：概览数字头 + 强烈推荐条目摘要（每条含一句话总结、评分概要、置信度、链接）+ 飞书文档链接
   - 内容总长度控制在 10KB 以内（飞书消息限制）
   - 消息格式使用飞书支持的 Markdown 语法
4. 实现 send_feishu_group_message 函数：调用 lark-oapi 的 im.v1.message.create API，发送到指定 chat_id（从环境变量 FEISHU_CHAT_ID 读取），设置 at_all=True
5. 实现 create_feishu_doc 函数：调用 docx.v1.document.create API，创建文档标题为"WyRadar 日报 | {date}"，内容为完整日报 Markdown，返回文档 URL
6. 所有飞书 API 调用必须包含 try-catch：失败时记录日志，不抛异常

**验证：**
- 调用 get_feishu_client()，确认客户端正常初始化（需配置环境变量）
- 调用 build_feishu_message 传入模拟数据，确认返回字符串长度 < 10240 字节
- 在沙箱环境（或真实飞书应用）中调用 send_feishu_group_message，确认消息发送成功（消息 ID 非空）
- 调用 create_feishu_doc，确认返回的 doc_url 格式为 `https://*.feishu.cn/docx/*`

---

### 步骤 4.2：实现 Dispatcher Node

**目标：** 实现 dispatcher_node 函数，完成双渠道推送。

**指令：**
1. 创建 backend/graph/nodes/dispatcher.py
2. 实现 dispatcher_node 函数：
   - 输入和输出均为 AgentState
   - 调用 build_feishu_message 和 send_feishu_group_message 发送群消息
   - 将返回的 message_id 存入 state['feishu_msg_id']
   - 调用 create_feishu_doc 创建飞书文档
   - 将返回的 doc_url 存入 state['feishu_doc_url']
   - 更新 daily_reports 表中的 feishu_msg_id 和 feishu_doc_url
   - 设置 state['current_stage'] = 'completed'、state['status'] = 'completed'
3. 推送失败时记录日志但不阻塞——Web 页面仍可查看日报

**验证：**
- 调用 editor_node 获取真实日报后，再调用 dispatcher_node(state)，确认 state['feishu_msg_id'] 和 state['feishu_doc_url'] 非空
- 确认飞书群收到消息（检查 @all 标记）
- 确认飞书文档可访问、内容完整无截断
- 模拟飞书 API 调用失败，确认 dispatcher_node 不抛异常，state['status'] 仍为 'completed'（Web可查看），日志中记录了错误信息

---

### 步骤 4.3：实现 FastAPI 后端路由

**目标：** 实现所有 P0 REST API 端点。

**指令：**
1. 创建 backend/api/reports.py：实现 `GET /api/report/today`（查询 daily_reports 表今天日期记录）和 `GET /api/report/{date}`（查询指定日期记录），返回 JSON 含 report_summary + report_markdown + analyzed_items 列表
2. 创建 backend/api/items.py：实现 `GET /api/items`，接受 11 个查询参数（date_from/date_to/category/recommend_level/score_min/score_max/keyword/page/page_size/sort_by/sort_order），构造 SQLAlchemy 查询并返回分页 JSON（含 total/page/page_size/total_pages/items）
3. 创建 backend/api/trigger.py：实现 `POST /api/trigger/daily-run`（BackgroundTasks 中异步运行 LangGraph graph.invoke()，返回 run_id）和 `GET /api/run-status/{run_id}`（查询 execution_logs 表返回各节点状态）和 `GET /api/run-history`（返回最近 20 条执行记录）
4. 创建 backend/api/stats.py：实现 `GET /api/stats`，返回近 7 天的每日统计（total_fetched/total_analyzed/recommend_high 趋势）
5. 在 backend/api/__init__.py 中创建 APIRouter，注册所有子路由
6. 在 main.py 中挂载 api_router，添加 CORS 中间件

**验证：**
- 启动 FastAPI 服务，访问 /docs 确认所有端点出现在 Swagger UI 中
- 使用 curl 测试 `GET /api/report/today`，确认返回 JSON 格式正确（report_summary + report_markdown + items）
- 使用 curl 测试 `GET /api/items?category=AI Agent&page=1&page_size=5`，确认分页正确
- 使用 curl 测试 `POST /api/trigger/daily-run`，确认返回 JSON 含 run_id 和 status
- 使用 curl 测试 `GET /api/run-status/{run_id}`，确认返回各节点状态

---

### 步骤 4.4：实现前端 — 今日日报页

**目标：** 实现 DailyReport.vue，包含概览卡片、情报卡片、全量列表。

**指令：**
1. 实现 frontend/src/views/DailyReport.vue
2. 页面加载时调用 `GET /api/report/today`，获取日报数据
3. 渲染概览统计卡片行：4 个卡片（采集条目数/筛选入库数/强烈推荐数/值得关注数），使用 Naive UI 的 n-card 组件，数字加粗显示
4. 渲染"强烈推荐"板块：遍历 recommend_level='强烈推荐' 的 items，每条渲染为可展开卡片：
   - 折叠态：显示标题、分类标签（n-tag）、评分数字（绿色粗体）、置信度百分比
   - 展开态：额外显示一句话总结、3 个核心亮点（bullet 列表）、评分明细（4 行条形图，颜色绿/蓝/黄/橙/红对应分数）、置信度 ⚠️ 标记（<0.6 维度显示黄色警告）、RAG 历史对比信息、原始链接（新标签页打开）
   - 卡片点击切换展开/折叠，带 CSS transition 动画
5. 渲染"值得关注"板块：同强烈推荐结构，默认全部折叠
6. 渲染"全量列表"板块：使用 Naive UI n-data-table，列含标题/来源/分类/评分/推荐等级，提供筛选下拉框（分类/来源/推荐等级）和搜索输入框（前端过滤），支持分页（每页 20 条）
7. 板块顶部提供"全部展开/折叠"按钮

**验证：**
- 页面加载后，概览卡片显示数字（与 /api/report/today 返回一致）
- 点击强烈推荐卡片，展开显示完整评分明细，再次点击折叠
- 点击"全部展开"，所有卡片展开；点击"全部折叠"，所有卡片折叠
- 全量列表筛选：选择"AI Agent"分类，确认表格仅显示该类条目
- 低置信度条目（如果有）正确显示 ⚠️ 标记
- 点击链接，在新标签页打开正确的 URL
- 浏览器窗口调整到 1366px 和 1920px 宽度，布局正常不溢出

---

### 步骤 4.5：实现前端 — 历史检索页

**目标：** 实现 HistorySearch.vue，包含多条件筛选、结果表格、详情展开。

**指令：**
1. 实现 frontend/src/views/HistorySearch.vue
2. 渲染筛选栏：5 个筛选条件——日期范围选择器（Naive UI n-date-picker, range 模式, 默认最近 30 天）、分类下拉（全部/AI Agent/多模态大模型/NLP）、推荐等级下拉（全部/4 级）、评分范围下拉（全部/≥4.0/3.0-3.9/2.0-2.9/<2.0）、关键词搜索输入框
3. 提供"检索"按钮（点击调用 `GET /api/items` 并传入所有筛选参数）和"重置"按钮（清空所有筛选，恢复默认，重新检索）
4. 渲染结果表格：Naive UI n-data-table，列含日期/标题/来源/分类/评分/推荐等级，支持点击列头切换排序（日期/评分），点击行在表格下方展开完整分析详情（同日报卡片的展开态）
5. 渲染分页器：Naive UI n-pagination，显示页码和上下页按钮，切换不丢失筛选条件
6. 空结果时显示 Naive UI n-empty 组件，提示"未找到匹配结果"

**验证：**
- 打开页面，确认默认加载最近 30 天数据
- 选择日期范围、分类、推荐等级后点击"检索"，确认表格数据更新为符合条件的结果
- 点击表格列头"评分"，确认数据按评分重新排序
- 点击一行，确认下方展开完整详情（含评分条形图和置信度标注）
- 点击分页器切换页码，确认筛选条件保持不变
- 输入不存在的关键词，确认显示"未找到匹配结果"
- 点击"重置"，确认所有筛选条件恢复默认

---

### 步骤 4.6：实现前端 — 手动触发页

**目标：** 实现 TriggerRun.vue，包含触发按钮、确认弹窗、实时进度、执行记录。

**指令：**
1. 实现 frontend/src/views/TriggerRun.vue
2. 渲染触发说明卡片：描述触发后会执行的操作（采集→筛选→分析→RAG→报告→推送），附注意事项（每天仅保留最新结果）
3. 渲染触发按钮：Naive UI n-button，点击弹出确认弹窗（Naive UI n-modal）"确定要立即执行日报流程吗？整个过程约需 15-25 分钟。"，确认后按钮变为灰色禁用态+loading，调用 `POST /api/trigger/daily-run`
4. 渲染实时执行进度：触发后显示 run_id 和开始时间，5 个节点各一行进度条（Naive UI n-progress），每行显示节点名称、进度百分比、状态图标（✅完成/🔄运行中/⏳等待/❌失败）、耗时。使用 setInterval 每 3 秒轮询 `GET /api/run-status/{run_id}` 更新状态
5. 执行完成后：进度条区域显示"✅ 日报已生成，请查看今日日报"，按钮恢复可点击状态
6. 渲染最近执行记录表格：Naive UI n-data-table，列含时间/触发方式/状态/耗时/详情，点击"查看日志"弹窗展示该 run 的 5 个节点详细日志

**验证：**
- 点击"立即执行日报流程"按钮，确认弹出确认弹窗
- 确认弹窗中点击"取消"，按钮不触发
- 确认弹窗中点击"确定"，按钮变为禁用态+loading
- 观察进度条，确认各节点状态按实际执行顺序更新（collector→curator→analyst→editor→dispatcher）
- 等待完成后，确认显示"日报已生成"消息，按钮恢复
- 查看最近执行记录表格，确认新记录出现
- 如果后端返回"已有运行中的流程"，确认前端显示该提示（而非无声崩溃）
- 故意触发一个失败场景（如断网），确认失败节点显示红色 ❌ + 错误信息

---

## Phase 5：调度 + 联调上线（2-3 天）

### 步骤 5.1：实现 APScheduler 定时任务

**目标：** 在 FastAPI 进程中嵌入定时调度，实现每日自动执行。

**指令：**
1. 创建 backend/scheduler.py
2. 实现 init_scheduler 函数：
   - 创建 BackgroundScheduler 实例
   - 注册 Job 1：Cron 表达式来自 config.CRON_COLLECT（默认 0 7 * * *），触发函数为运行 LangGraph graph.invoke()（采集→策展→分析→主编），写入 execution_logs
   - 注册 Job 2：Cron 表达式来自 config.CRON_DISPATCH（默认 0 8 * * *），触发函数为运行 Dispatcher Node（推送），如果当日日报未完成则标记为 partial
   - 在 FastAPI 的 lifespan 事件中启动和关闭 scheduler
3. 实现防并发锁：使用数据库行锁（SELECT FOR UPDATE 在 execution_logs 上），同一时间只允许一个流程运行。如果手动触发时检测到已有运行中的定时任务，返回提示"已有运行中的流程"
4. 实现超时处理：如果 Job 1 到 8:00 仍未完成，Job 2 推送时标记日报为 partial 并推送已完成部分

**验证：**
- 修改 config 中 Cron 表达式为每分钟执行一次（"* * * * *"），观察是否自动触发
- 启动 FastAPI，查看日志确认 scheduler 已启动
- 手动触发后立即再次点击手动触发，确认后端返回"已有运行中的流程"
- 模拟 Job 1 长时间运行（暂不实现），确认 Job 2 超时检测逻辑可触发
- 恢复 Cron 为真实时间后重启

---

### 步骤 5.2：端到端集成测试

**目标：** 编写并执行完整的日报链路测试。

**指令：**
1. 创建 backend/tests/test_e2e_pipeline.py
2. 实现以下测试用例：
   a. 全链路成功测试：构造初始 State，调用 graph.invoke()，验证 5 个节点均标记为 success，state['status']='completed'，daily_reports 表有记录，analyzed_items 表有记录
   b. 单信源失败测试：模拟 GitHub 采集失败（设置错误 URL），验证其他 3 个源正常，fetch_errors 含 GitHub 错误信息，后续节点正常执行
   c. LLM 分析部分失败测试：模拟 3 条中 1 条 LLM 分析超时，验证其他 2 条正常，analysis_errors 含失败条目
   d. 飞书推送失败测试：模拟飞书 API 不可用，验证 dispatcher_node 不抛异常，state['status'] 仍为 'completed'，Web 可查看日报
   e. ChromaDB 空库降级测试：删除 ChromaDB 数据目录后运行，验证 Analyst 正常降级（RAG 返回空列表），分析正常完成
3. 每个测试用例记录执行时间和结果

**验证：**
- 执行 `pytest backend/tests/test_e2e_pipeline.py -v`，所有测试用例通过
- 确认全链路测试耗时在 15-25 分钟范围内
- 打开生成的日报 Markdown，人工检查格式和内容质量
- 检查数据库 4 个表的数据一致性（raw_items.id = analyzed_items.id，日期关联正确）

---

### 步骤 5.3：飞书通道联调

**目标：** 在真实飞书环境中完成端到端推送测试。

**指令：**
1. 配置 .env 文件中的飞书相关环境变量（APP_ID/APP_SECRET/CHAT_ID），确认 Bot 已添加到目标群聊并具有发送消息和 @all 权限
2. 运行一次完整的日报生成流程
3. 检查飞书群是否收到消息：
   - 消息包含 @all 标记
   - 消息内容包含概览数字和强烈推荐条目
   - 消息总长度 < 10KB
   - 飞书文档链接可点击
4. 点击文档链接，检查文档内容：
   - 完整 5 个板块均正确渲染
   - 评分明细表格式正确
   - 外部链接可点击
5. 测试推送失败恢复：临时修改错误的 APP_SECRET，确认系统不崩溃，Web 页面仍可正常查看日报

**验证：**
- 飞书群收到 WyRadar 日报消息
- 飞书文档完整性和格式正确
- 推送失败时 Web 页面正常
- 飞书消息未触发内容审核或限流警告

---

### 步骤 5.4：编写部署文档与一键启动脚本

**目标：** 编写 README.md 和部署说明。

**指令：**
1. 创建 README.md，包含以下章节：
   - 项目简介（一句话 + 核心能力）
   - 快速开始（环境要求 Python 3.11+/Node 18+/Docker，克隆仓库）
   - 配置说明（.env 文件各项的含义和获取方式）
   - 一键部署（docker compose up 或手动启动脚本）
   - 使用说明（Web 页面的三个 Tab 及其功能）
   - 技术架构概览
2. 创建 setup.sh 脚本：检查 Python 和 Node 版本，复制 .env.example 为 .env（如果不存在），创建 data/ 和 logs/ 目录，安装依赖（pip + npm），初始化数据库，构建前端
3. 创建 start.sh 脚本：启动 FastAPI 后端（后台）、启动 Vite 开发服务器，输出访问地址

**验证：**
- 在新目录中 clone 项目，执行 setup.sh，无报错
- 执行 start.sh，浏览器访问 http://localhost:8000 和 http://localhost:5173 均可正常访问
- 阅读 README.md，确认新人能按步骤完成首次启动

---

## 步骤依赖图

```
Phase 1:
1.1 → 1.2 → 1.3 → 1.4 → 1.5 → 1.6 → 1.7 → 1.8
                                    └──────────────┘
                                    1.5/1.6/1.7 可并行

Phase 2:
2.1 ──┐
2.2 ──┤
2.3 ──┼──→ 2.5 → 2.6 → 2.7 → 2.8 → 2.9
2.4 ──┘
(2.1-2.4 可并行开发)

Phase 3:
3.1 → 3.2 → 3.3 → 3.4 → 3.5
                         ↓
                       3.6
(3.5 评分验证可与 3.6 Editor 并行)

Phase 4:
4.1 → 4.2 → 4.3 → 4.4 → 4.5 → 4.6
              └──────────────────┘
              4.4/4.5/4.6 可并行

Phase 5:
5.1 → 5.2 → 5.3 → 5.4
```

## 总工时估算

| Phase | 步骤数 | 预估天数 |
|-------|--------|---------|
| Phase 1 | 8 | 3-4 |
| Phase 2 | 9 | 4-5 |
| Phase 3 | 6 | 5-6 |
| Phase 4 | 6 | 5-6 |
| Phase 5 | 4 | 2-3 |
| **合计** | **33** | **19-24** |
