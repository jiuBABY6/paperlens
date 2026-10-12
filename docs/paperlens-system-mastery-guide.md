# PaperLens 系统掌握手册

> 本文是 PaperLens 的学习总纲。事实优先级为：当前代码 > 最新冻结报告 > README 与其他说明文档。本文描述的是本地 `D:\Desktop\job\demo` 当前实现，不代表远端仓库或未来版本。

## 1. 先记住一句话

PaperLens 是一个面向学术论文的证据驱动型多模态 RAG 系统：它解析正文、Figure 和 Table，在 Standard RAG 与受约束的 LangGraph Supervisor–Worker 工作流之间分流，并把回答绑定到可回到原文核验的页码、BBox 和 Evidence ID。

它解决的不是“让模型自由总结论文”，而是三个更具体的问题：

1. 如何从结构复杂的 PDF 中得到可检索、可定位的文本、图和表；
2. 如何让不同问题进入成本和复杂度合适的执行链路；
3. 如何让答案只使用已检索、已校验、可追踪的论文证据。

## 2. 首次出现的核心术语

- **RAG（Retrieval-Augmented Generation，检索增强生成）**：先从论文中检索证据，再让模型基于证据回答，而不是只依赖模型参数中的知识。
- **Evidence（证据对象）**：系统内部统一表示正文句子、Figure 或 Table 的结构，包含稳定 ID、页码、BBox、内容、章节和元数据。
- **BBox（Bounding Box，边界框）**：证据在 PDF 页面上的矩形坐标，前端可据此高亮原文。
- **Standard RAG**：单次检索、证据化生成与 Claim 校验链路，适合简单正文问题。
- **Agentic RAG**：由 Supervisor 拆分任务，Text/Figure/Table Specialist 获取证据，Critic 验收，再由 Answer Agent 合成回答的受约束工作流。
- **Function Calling**：模型只能从服务端提供的工具 Schema 中选择工具并提交结构化参数；服务端仍负责校验、执行和预算控制。
- **Reducer**：LangGraph 中合并并行状态更新的规则，不是一个会独立决策的 Agent。

## 3. 两条主链路

```text
论文摄取链
PDF -> 校验/哈希去重 -> Docling + PyMuPDF -> Figure 离线理解
    -> Text/Figure/Table 检索表示 -> Qdrant/SQLite -> 可问答论文

在线问答链
Question -> Query Resolver -> Router
    -> Standard RAG
    或
    -> Supervisor -> Specialists -> Evidence Critic -> Answer/Repair/Refusal
    -> Claim 校验 -> Citation/Trace/Run/Memory/SSE
```

“两条主链路”指摄取与问答；问答链内部再分成 Standard 和 Agentic 两条执行路线。

## 4. 核心数据对象

### 4.1 论文对象

`app/domain.py` 中的 `Paper` 聚合：

- `Chunk`：用于检索的较大文本单元；
- `Sentence`：用于最终引用的句子级单元，可保存多个 BBox；
- `Figure`：同时表示 Picture 和 Table，使用 `kind` 区分；
- 解析器名称、分析版本和索引版本等论文状态。

### 4.2 EvidenceObject

`EvidenceObject` 是三种模态进入 Agent 和答案链路后的统一契约：

```text
evidence_id
type: text | figure | table
page
bbox
content
section
metadata
```

统一 Evidence 的价值是让 Critic、Answer Agent、Citation API 和前端定位不必分别理解三套数据模型。

### 4.3 Conversation、Message 与 Run

- `Conversation`：一篇论文下的持久化会话；
- `Message`：按轮次保存用户或助手内容、Citation 和状态；
- `Run`：一次回答的运行实例，保存 queued/running/completed/partial/failed/cancelled 等状态。

一次用户消息和对应 Run 在 SQLite 中原子创建，并通过 `client_message_id` 支持幂等提交。

## 5. 上传论文后发生什么

入口：`POST /api/papers`，实现位于 `app/main.py::upload`。

### 5.1 文件校验与去重

`stage_pdf()` 流式写入临时文件，同时检查：

- 扩展名；
- 文件大小；
- `%PDF` 文件头；
- SHA-256 内容哈希。

`Repository.reserve_upload()` 使用 SQLite 事务建立文档注册记录：

- 相同哈希且已完成：复用已有论文；
- 相同哈希仍在处理：返回 processing；
- 新内容：创建新的 `paper_id` 和 processing 状态。

### 5.2 解析正文、Figure 与 Table

`PaperParser.parse()` 的主路径是 Docling，PyMuPDF 同时负责页面文字、句子和坐标：

- Docling 提供章节结构、Picture、Table、OCR 和表格结构；
- PyMuPDF 提供页码、文本行、单词坐标和句子 BBox；
- 两者通过页码、文本相似度和 BBox 重叠对齐；
- Docling 整体失败时降级到 PyMuPDF。

PyMuPDF fallback 可以保住正文问答，但 Figure/Table 结构能力会明显弱于 Docling 主路径，尤其不能保证提取矢量图和完整表格结构。

### 5.3 Figure 离线理解

`FigureUnderstandingService.enrich_paper()` 只处理 `kind=picture` 且存在原图的 Figure。Qwen-VL 生成：

- summary、entities、components；
- relations、visual_evidence、numerical_evidence；
- retrieval_keywords、potential_questions、uncertainty。

失败会把该 Figure 标记为 error/unavailable，但不会让整篇论文上传失败。

### 5.4 建立索引

`HybridRetriever.index_chunks()` 将三类内容放进同一篇论文的检索空间：

- Text：原始 Chunk；
- Figure：Caption + Qwen-VL 离线描述；
- Table：Caption + Docling 提取的 Markdown 内容。

BGE-M3 生成向量，Qdrant 为每篇论文建立独立 collection。BM25 不单独持久化，而是在当前论文 Chunk 上按查询实时计算。

索引失败不会必然回滚论文。当前代码可能把论文保存为 completed，但设置 `vector_indexed=false`、`index_version=0`，后续问答会尝试重新建立索引或以可用能力降级。

### 5.5 持久化与输出

SQLite 保存论文、Chunk、Sentence、Figure、阅读卡片、分析版本和索引状态。上传处理通过线程池执行以避免阻塞 FastAPI 事件循环，但上传 HTTP 请求仍等待处理结果；项目没有分布式后台任务队列。

## 6. 一个问题怎样进入系统

直接单轮接口为 `POST /api/papers/{paper_id}/ask`；持久化会话接口为 `POST /api/papers/{paper_id}/conversations/{conversation_id}/messages`。

会话接口先创建 Message 和 Run，返回 `202`，再由后台协程执行。执行过程会：

1. 标记 Run 为 running；
2. 加载对话上下文和论文级学习记忆；
3. 解析追问；
4. 调用统一 `_ask_paper()` 内核；
5. 持久化 Answer、Citation、Trace 和最终 Run 状态；
6. 更新会话摘要与长期记忆；
7. 发布 SSE 终态事件。

直接 `/ask` 不经过完整的持久化会话 Query Resolver，因此分析多轮行为时应以 Conversation API 为准。

## 7. 三层 Query Rewrite

### 7.1 多轮追问消歧

`QueryResolver` 读取最近有限消息、会话摘要、会话 memory、论文学习 memory 和最近引用 ID。出现“它、这个方法、为什么”等上下文标记时，可调用 DeepSeek 将追问改写为独立问题。

模型不可用时，规则 fallback 通常把上一用户问题和当前追问拼接。只有在无法找到足够上下文等情况下才请求澄清，因此不能概括成“规则 fallback 总会让用户澄清”。

### 7.2 中文问题到英文论文的检索改写

`ReadingService.plan_query()` 为检索生成：

- `semantic_query`：中文问题通常保留原文，以利用多语言向量能力；
- `lexical_query`：调用 DeepSeek 改写成适合英文论文关键词匹配的表达；
- 缩写扩展与章节提示。

模型不可用时，会退化为 ASCII、数字、已知缩写等可安全提取的词面信息。

### 7.3 Specialist 子任务检索词

`Planner` 根据 Router 给出的模态、显式 Figure/Table 编号和问题关键词生成 Specialist 子任务。当前 Planner 是确定性 Python 逻辑，不是一个独立 LLM Agent，也不是单独的 LangGraph 节点。

## 8. Standard RAG

适用场景：单一正文事实、定义、作者明确陈述的结论，不需要读图表或跨模态汇总。

处理过程：

1. 生成 semantic/lexical query；
2. BM25 找词面匹配；
3. BGE-M3 + Qdrant 找语义匹配；
4. RRF 合并两个排名；
5. CrossEncoder Reranker 重排候选；
6. 从命中 Chunk 中选择句子级候选；
7. DeepSeek 生成最多若干条原子 Claim，并为每条绑定 Sentence ID；
8. 单独的 Claim verifier 判断支持、部分支持或不支持；
9. 返回受支持的回答和句子 Citation，证据不足时结构化拒答。

设计取舍：Chunk 更适合召回上下文，Sentence 更适合精确引用，所以系统“检索 Chunk，最终引用 Sentence”。

在线检索允许降级：Dense 或 Reranker 不可用时可退回其他策略并写入 warning；严格离线评测模式则抛出后端错误，避免悄悄改变评测条件。

## 9. Agentic RAG

### 9.1 Router

Router 使用规则识别 Figure/Table 编号、视觉词、表格词、跨模态关系和复杂论证。简单正文题走 Standard；图、表、跨模态或复合任务走 Agentic。

### 9.2 Supervisor 与 Planner

Supervisor 将逻辑任务展开成 text/figure/table 模态子任务，分配工具、步数、模型轮次、Qwen-VL 次数和超时预算。正常应用配置默认串行；开启并行开关且存在多个无依赖 Specialist 时才使用线程池并行。冻结 v4 Dev 最终报告明确使用了 `parallel_enabled=true`。

### 9.3 Specialist

- TextResearchAgent：`search_text`，得到句子级 Evidence；
- FigureAnalysisAgent：搜索 Figure，并对需要视觉判断的问题调用 Qwen-VL 看原图；
- TableAnalysisAgent：搜索并读取结构化 Table，必要时才使用表格图像分析。

每个 Specialist 可以走 Function Calling，也保留代码固定的执行流程作为可靠性回退。

### 9.4 Evidence Critic

Critic 是确定性验收层，主要检查：

- Router 要求的模态是否都有可用证据；
- Figure 分析是否 `answerable`；
- Table 是否包含可读结构；
- Specialist 是否报告结构化冲突；
- 是否还有重试预算。

结果是 approved、partial、retry 或 refuse。retry 只定向重做缺失或失败的任务，不会让全部 Agent 无条件重跑。

### 9.5 Answer Agent 与拒答

Answer Agent 只接收共享 State 中已通过检查的 Evidence，再调用 `ReadingService.answer_from_evidence()` 生成和校验 Claim。缺证据、关键模态缺失或答案生成失败时返回 partial/refusal，而不是用常识补全论文内容。

## 10. Function Calling 的真实边界

### 10.1 已实现

- Specialist 级工具白名单；
- 严格 JSON Schema 和 `additionalProperties=false`；
- 服务端参数类型、必填项、范围二次校验；
- 重复工具签名检测；
- 模型轮次、工具步数、Qwen-VL 次数和结果长度预算；
- Figure/Table 目标 ID 的候选集合检查；
- Figure/Table 完成条件；
- Provider/tool-completion 层熔断；
- 异常、空证据或流程不完整时回退固定 Specialist 流程。

### 10.2 模型能控制什么

模型可以在允许的工具中决定下一次调用及参数，但不能：

- 提供任意文件路径、URL 或其他 `paper_id`；
- 注册新工具；
- 绕过服务端 Schema、预算和完成条件；
- 直接把工具结果写成最终用户答案。

### 10.3 不能夸大的地方

- Figure/Table 的“目标必须来自本轮搜索”只在已发现候选集合非空时严格执行；
- Text `read_sentence` 没有完全相同的本轮候选约束；
- 本地无效参数会记录 rejection 并触发回退，但不一定累计 Provider 熔断计数；
- 并行节点可能同时消耗预算，最终答案合成也被实现明确保留，因此预算是受控上限机制，不是分布式强一致计费系统。

最新冻结报告中出现过 2 次 `required_visual_analysis_missing` 回退，固定 Figure 流程随后成功完成任务。这说明回退不是纸面设计，而是实际评测路径的一部分。

## 11. Text、Figure、Table 的差异

| 模态 | 原始表示 | 检索表示 | 最终可用证据 | 主要失败方式 |
|---|---|---|---|---|
| Text | Chunk、Sentence、页码、BBox | 原始文本与章节 | Sentence Evidence | 解析错位、召回遗漏、Claim 不受支持 |
| Figure | 图片、Caption、邻近文本、BBox | Qwen-VL 离线结构描述 | 按问题查看原图后的 Figure Evidence | 命中错图、未执行视觉分析、图片不可读 |
| Table | 图片、Caption、Markdown、BBox | Caption + 结构化内容 | columns/rows 或表格内容 | 表结构缺失、数字/列名读取失败 |

三种模态共用 Evidence 契约和向量基础设施，但解析方式、完成条件和失败恢复不同。

## 12. 对话、SSE 与取消

### 12.1 对话持久化

Conversation、Message、Run 写入 SQLite。服务启动时，遗留的 queued/running Run 会被标记为 interrupted，避免永远悬挂。

短期上下文默认取最近约 6 轮消息，并受字符预算限制。当前摘要是最近消息片段的确定性拼接，不是 LLM 压缩摘要。

### 12.2 SSE

`RunManager` 在进程内维护每个 Run 的事件队列，支持：

- 有序事件 ID；
- `Last-Event-ID` 续接；
- heartbeat；
- Router、Node、Tool、Critic、Answer 和终态事件。

答案不是 Provider 原生 Token Streaming。系统先完成证据与 Claim 校验，再把安全答案按字符块拆成 `answer.delta`。好处是不会把未验证内容先发给前端；代价是首个答案字符出现得更晚。

事件缓冲只在当前进程内，服务重启或多 Worker 之间不能完整续播。

### 12.3 取消

取消接口写入 `cancel_requested`，执行器在关键边界检查并把 Run 结束为 cancelled。由于模型调用在线程中同步执行，系统不能强制杀死已经发出的 Provider 请求；这是协作式取消，不是抢占式中断。

## 13. 论文级长期记忆

长期记忆的目标是记录“用户在这篇论文上学过什么、哪些问题仍未解决”，不是保存新的论文事实。

每条记忆保存：

- `paper_id + user_scope`；
- 来源 conversation/run 和分析版本；
- resolved question、回答摘要；
- Evidence ID、模态、章节；
- importance、confidence、过期时间、用户备注和置顶状态。

状态语义：

- `active`：完成、可回答且有 Citation；
- `unresolved`：拒答、partial 或缺少可验证证据；
- `stale`：旧分析版本的有效记忆；
- `archived`：过期或容量回收；
- `forgotten`：用户主动遗忘。

当前版本更新逻辑只把旧版本 `active` 条目标记为 `stale`；已有 `unresolved` 不会自动转为 stale。记忆可用于追问消歧和学习导航，但论文事实仍必须重新检索当前 Evidence。

当前默认 `user_scope=local`，没有登录、鉴权、RBAC 或完整多租户隔离。

## 14. 数据分别存在哪里

| 数据 | 存储 | 说明 |
|---|---|---|
| 论文、Chunk、Sentence、Figure、Card | SQLite | 业务真相来源 |
| Conversation、Message、Run、应用 Trace | SQLite | 支持会话恢复与审计 |
| 长期记忆条目及聚合 | SQLite | 以 paper_id 和 local scope 隔离 |
| Dense 向量 | Qdrant | 一篇论文一个 collection |
| 原始 PDF、Figure/Table 图片 | 本地文件系统 | Evidence API 和前端读取 |
| LangGraph checkpoint | SQLite | 当前在线请求通常以 Run UUID 作为 thread_id |
| SSE 事件缓存 | 进程内内存 | 有保留时间和容量限制 |
| Prometheus 指标 | Prometheus | Compose 基础设施 |
| OpenTelemetry Trace | Tempo | 经 OTel Collector 转发 |

## 15. Evaluation：怎样读结果

评测实现将指标分为：

1. Retrieval：预期证据是否进入 Top-K；
2. Evidence：最终 Citation 与 Gold Evidence 的命中、完整率、Precision、Recall；
3. Execution Success：工作流是否无异常并产生合法状态；
4. Answer Correct：答案或拒答是否正确；
5. Task Success：执行、可回答性、答案与证据条件的综合成功。

最新完整冻结报告是 `evals/report-dev-v4-final.json`：

- v4 共 5 篇论文、60 题；Dev 36，Test 24；
- 已完整冻结的是 36 条 Dev，其中 30 条可回答、6 条不可回答；
- Task Success 36/36；
- Gold Citation Hit 100%；
- Complete Gold Citation Hit 93.33%；
- Evidence Precision/Recall 92.54%/96.94%；
- 平均端到端延迟 12.98s -> 10.53s；
- P95 41.35s -> 36.67s。

36/36 是固定小规模 Dev 集上的 judge-enabled 结果：30 条可回答题进入 DeepSeek/Qwen-VL 裁判体系，6 条不可回答题按拒答逻辑判定。它不等于现实准确率 100%，也不能证明开放世界泛化。

v4 的 24 条 Test 尚未完成正式冻结评测。历史 v2 Test 的 8/8 属于不同数据版本，不能与 v4 Dev 合并。

冻结说明记录 177 passed；当前测试源码的静态收集结构也对应 177 个 Case。需要区分“冻结时通过”与“当前重新运行通过”，后者必须实际执行测试后才能声称。

## 16. LLMOps 的真实程度

当前实现包括：

- FastAPI `/metrics`；
- HTTP、Run、Node、Tool、Model、Token、Memory 指标；
- 结构化日志与敏感字段过滤；
- OpenTelemetry Span/Event；
- Prometheus、Tempo、OTel Collector、Grafana Compose 配置；
- System、Agents & Tools、Models & Memory 三套 Dashboard；
- 仓库内保存的实际 Dashboard 截图。

FastAPI 本身不在 Compose 中，Prometheus 通过 `host.docker.internal:8010` 抓取本地 Uvicorn。当前没有完整告警规则、SLO、自动质量门禁、集中日志平台、生产权限和长期 Trace 保留策略，因此应描述为“具备可运行的观测闭环”，而不是“生产级 LLMOps 平台”。

## 17. 失败、降级与拒答

| 失败位置 | 当前行为 |
|---|---|
| Docling 解析失败 | 降级 PyMuPDF，保正文能力 |
| Figure 离线理解失败 | 保存 error/unavailable 描述，继续上传 |
| Qdrant 或 Reranker 在线不可用 | 非严格模式降级并写 warning |
| Query Resolver 模型不可用 | 使用有限规则拼接上下文 |
| Function Calling 异常/空证据/未完成 | 回退固定 Specialist 流程 |
| 关键模态缺证据 | Critic 定向返工；无预算则 partial/refusal |
| Claim 不受 Evidence 支持 | 删除该 Claim 或结构化拒答 |
| Run 进程中断 | 服务启动时标记 interrupted |
| 用户取消 | 边界检查后结束为 cancelled，不能抢占 Provider 请求 |
| 论文重新解析失败 | 恢复论文 completed 状态并保留旧分析数据 |

系统的恢复原则是“显式记录降级原因并尽量保住可信能力”，不是悄悄用模型常识补答案。

## 18. 当前没有实现的能力

- 没有模型训练、微调或模型结构研究；
- PaperLens 主项目没有接入 MCP；
- 没有 Kubernetes、Celery/分布式队列或大规模集群；
- Qdrant 主要是本地/单节点使用；
- 没有登录鉴权和真实多租户记忆隔离；
- SSE 不是 Provider 原生 Token Streaming；
- 前端是可用的论文阅读与对话页面，不是复杂 React/Vue 工程；
- 多智能体不是多个 Agent 自由自治或互相聊天；
- 5 篇论文的 Benchmark 不能证明开放世界泛化。

## 19. 代码阅读主路线

建议按调用关系阅读，而不是按目录字母顺序阅读：

1. `app/domain.py`：先理解 Paper、Chunk、Sentence、Figure、EvidenceObject；
2. `app/main.py`：定位上传、问答、Conversation、Run、SSE 和 Evidence API；
3. `app/services/parser.py`：理解三模态数据从哪里来；
4. `app/services/retrieval.py`：理解索引、BM25、Dense、RRF、Reranker；
5. `app/services/reading.py`：理解 Query Rewrite、答案生成和 Claim 校验；
6. `app/agent/router.py` 与 `planner.py`：理解规则分流和子任务改写；
7. `app/multi_agent/supervisor.py`、`graph.py`、`state.py`：理解图与预算；
8. `app/multi_agent/agents/`：逐一理解 Specialist、Critic 和 Answer；
9. `app/multi_agent/function_calling.py`、`tool_registry.py`、`app/tools/evidence_tools.py`：理解模型权限边界；
10. `app/services/query_resolver.py`、`run_manager.py`、`long_term_memory.py`：理解产品交互层；
11. `app/evaluation/` 与 `scripts/evaluate.py`：理解结果口径；
12. `app/observability.py`、`docker-compose.yml`、`observability/`：理解运维边界。

## 20. 面试表达模板

### 30 秒版本

> PaperLens 是我独立推进、借助 AI 辅助编码完成的多模态论文阅读系统。上传阶段用 Docling 和 PyMuPDF解析正文、图表、页码与 BBox，并用 BGE-M3、Qdrant、BM25、RRF 和 Reranker 建立混合检索。问答阶段由规则 Router 在 Standard RAG 和受约束的 LangGraph Supervisor–Worker 工作流之间分流，通过 Text、Figure、Table Specialist、Evidence Critic 和 Claim 校验生成可回到原文定位的答案。我还实现了持久化会话、SSE 运行事件、论文级长期记忆、离线分层评测和 Prometheus–Tempo–Grafana 观测闭环。

### 深入展开顺序

1. 先讲 Evidence Contract；
2. 用一个真实问题走完整数据流；
3. 解释一次关键设计取舍；
4. 说明失败怎样降级或拒答；
5. 用冻结报告说明效果；
6. 主动说明 Benchmark 和部署边界。

## 21. 学完后的自测

不看文档，确认自己能回答：

1. 为什么检索 Chunk，最终引用 Sentence？
2. Docling 与 PyMuPDF 各自解决什么问题？
3. 三层 Query Rewrite 分别在哪里、解决什么问题？
4. Router、Supervisor、Planner、Critic 的职责为什么不能混在一起？
5. Figure 为什么既要离线理解，又要按问题查看原图？
6. Function Calling 中模型能做什么、不能做什么？
7. 工具调用成功为什么不等于 Specialist 成功？
8. SSE 为什么不是 Token Streaming？取消为什么不是硬中断？
9. 长期记忆为什么不能直接充当论文证据？
10. 36/36 为什么不能表述成现实准确率 100%？
11. 当前最重要的五条生产化缺口是什么？
12. 如果某个 Figure 问题失败，你会按哪几层定位？

如果这些问题不能从代码路径、输入、处理、输出、失败恢复和边界六个角度回答，就继续回到对应章节和代码验证，而不是背诵一句结论。
