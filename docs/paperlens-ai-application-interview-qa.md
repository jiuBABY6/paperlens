# PaperLens AI 应用开发技术面试题库

> 这 60 题用于理解后的口头练习，不应先背。每个回答都以本地当前代码和最新冻结报告为依据。回答面试官时建议遵循：直接结论 → 实现路径 → 设计原因 → 失败恢复 → 当前边界。

## 一、项目定位与总体架构

### Q1：请用 30 秒介绍 PaperLens。

PaperLens 是一个证据驱动的多模态论文阅读系统。上传阶段用 Docling 和 PyMuPDF 提取正文、Figure、Table、页码与 BBox，并通过 BM25、BGE-M3、Qdrant、RRF 和 Reranker 建索引；问答阶段由 Router 在 Standard RAG 与受约束的 LangGraph Supervisor–Worker 工作流之间分流，最终答案必须绑定可回到原文核验的 Evidence。项目还包含持久化会话、SSE 运行事件、长期记忆、分层评测和 Prometheus–Tempo–Grafana 观测闭环。

**代码锚点**：`app/main.py`、`app/domain.py::EvidenceObject`、`app/multi_agent/graph.py`。

**边界**：没有训练模型，不是自治 Agent，也不能把 v4 Dev 36/36 说成现实准确率 100%。

### Q2：PaperLens 解决了什么实际问题？

普通 PDF Chat 容易丢失论文结构、无法真正看图表、引用粒度粗，也很难解释一次回答为什么失败。PaperLens 把解析、检索、图表分析、证据校验、原文定位和运行 Trace 串成一条可审计链路，让用户不仅得到答案，还能回到具体页码和区域验证。

**代码锚点**：`app/services/parser.py`、`app/tools/evidence_tools.py`、`GET /api/papers/{paper_id}/evidence/{evidence_id}`。

### Q3：为什么它属于 RAG？

因为回答不是直接依赖模型参数知识，而是先从当前论文检索 Chunk、Sentence、Figure 或 Table，再把这些 Evidence 交给 DeepSeek 生成和校验 Claim。没有足够 Evidence 时，系统会 partial 或拒答。

**代码锚点**：`app/services/retrieval.py::search_with_trace`、`app/services/reading.py::answer_with_evidence`。

### Q4：为什么还需要 Agentic RAG？

简单正文问题用一次检索即可，但 Figure、Table 或跨模态问题需要不同工具、完成条件和模型。Agentic RAG 让 Supervisor 按模态拆任务，Specialist 独立取证，Critic 验收，再合成答案；它解决的是复杂任务分工和失败隔离，不是为了增加 Agent 数量。

**代码锚点**：`app/agent/router.py`、`app/multi_agent/supervisor.py`、`app/multi_agent/agents/`。

### Q5：系统的两条主链路是什么？

第一条是论文摄取链：上传、校验、去重、解析、Figure 离线理解、索引和持久化。第二条是在线问答链：多轮消歧、Router 分流、Standard/Agentic 执行、证据校验、Citation、Trace、Run、SSE 和记忆更新。

**代码锚点**：`app/main.py::upload/process_paper/_ask_paper/_execute_conversation_run`。

### Q6：为什么同时保留 Standard RAG 和 Agentic RAG？

Standard RAG 的调用链短、成本低、结果稳定，适合单一正文事实；Agentic RAG 能处理图、表和复合任务，但延迟、调用次数和故障面更大。Router 让系统按问题复杂度付出成本，避免所有问题都进入多 Agent。

**可能追问**：Router 是 LLM 吗？不是，当前是可测试的规则路由。

## 二、上传、解析、索引与存储

### Q7：论文上传后经历哪些步骤？

接口先流式保存临时文件，校验扩展名、大小和 PDF 文件头并计算 SHA-256；Repository 做哈希去重和 processing 占位；Parser 用 Docling/PyMuPDF 生成 Paper；Qwen-VL 离线描述 Picture；Retriever 建 Qdrant 索引；ReadingService 生成阅读卡片；最后 SQLite 事务化保存。

**代码锚点**：`app/main.py::upload/stage_pdf/process_paper`。

**失败恢复**：整体异常会释放注册记录、删除新索引和本次论文目录；单图理解失败不会中断上传。

### Q8：为什么需要 SHA-256 和文档注册表？

SHA-256 用内容而不是文件名识别重复 PDF；文档注册表用事务协调 completed、processing 和新上传，避免两个请求同时重复解析同一文件。它提供业务幂等，但不是语义级论文去重。

**代码锚点**：`app/repository.py::reserve_upload/release_upload`。

### Q9：为什么同时使用 Docling 和 PyMuPDF？

Docling 擅长论文结构、Picture、Table、OCR 和表格内容，PyMuPDF 擅长稳定读取页面文字、词和坐标。PaperLens 用 Docling 提供语义结构，用 PyMuPDF 提供精确定位，再按页面、文本相似度和 BBox 重叠对齐。

**代码锚点**：`app/services/parser.py::_parse_with_docling`。

### Q10：Docling 失败时怎样处理？

`PaperParser.parse()` 捕获 Docling 异常并调用 `_parse_with_pymupdf()`。这样通常能保住正文问答和部分嵌入式图片，但表格结构、矢量图和复杂版面能力会下降，因此是显式降级，不是等价备用解析器。

### Q11：正文怎样变成可引用证据？

正文先形成带 section/page/bbox 的 Chunk，同时由 PyMuPDF 词坐标切成 Sentence；检索阶段用 Chunk 保留上下文，回答阶段从命中 Chunk 中选择 Sentence，并把 Sentence ID 和多个 BBox 作为 Citation。

**代码锚点**：`app/domain.py::Chunk/Sentence/sentence_evidence`、`EvidenceTools.search_text`。

### Q12：Figure 怎样处理？

Parser 保存 Figure 的图片、Caption、页码、BBox、邻近正文和相关 Sentence ID。上传时 Qwen-VL 生成离线结构化描述用于检索；用户问具体视觉问题时，Figure Specialist 再让 Qwen-VL 按原问题查看原图，形成问题相关 Evidence。

**代码锚点**：`app/multimodal/figure_understanding.py::enrich_paper/analyze_for_query`。

### Q13：为什么 Figure 需要离线理解和在线分析两层？

离线描述只做一次，适合扩大召回词汇并降低每次搜索成本；在线分析关注当前问题，例如颜色、箭头或相对位置，必须重新看原图。只依赖 Caption 或通用摘要不能可靠回答所有视觉细节。

### Q14：Table 怎样处理？

Docling 导出 Table 图片和 Markdown，系统将其保存为 `kind=table` 的 Figure 对象。索引文本是 Caption 加 Markdown；在线先精确/混合检索，再由 `read_table()` 转成 columns/rows。只有结构读取不够且配置允许时，才使用 Qwen-VL 表格图像 fallback。

**代码锚点**：`app/domain.py::markdown_table_data/Figure.as_evidence`、`EvidenceTools.read_table`。

### Q15：数据分别存在哪里？

SQLite 保存论文结构、会话、Run、应用 Trace、缓存和长期记忆；Qdrant 保存 Dense 向量；PDF 和图表图片在本地文件系统；LangGraph checkpoint 也使用 SQLite；SSE 事件只在当前进程内存；Prometheus 保存指标，Tempo 保存 OpenTelemetry Trace。

**边界**：BM25 是运行时计算，不存进 Qdrant；FastAPI 不在 Docker Compose 服务中。

## 三、Query Rewrite、检索与 Evidence

### Q16：项目有哪三层 Query Rewrite？

第一层 `QueryResolver` 把多轮追问补全成独立问题；第二层 `ReadingService.plan_query()` 把中文问题转成适合英文论文的 lexical query，同时保留多语言 semantic query；第三层 Planner 为 Text/Figure/Table Specialist 生成子任务检索词。

**边界**：第三层是确定性 Python 规则，不是 LLM Planner。

### Q17：多轮追问消歧怎样实现？

Resolver 读取最近约 6 轮消息、摘要、会话 memory、论文学习 memory 和最近 Citation ID。出现上下文指代时可调用 DeepSeek 输出 resolved question 或 clarification；模型不可用时用规则把上一问题与当前追问有限拼接。

**代码锚点**：`app/services/query_resolver.py::build_context/resolve`。

### Q18：中文问题为什么同时需要 semantic 和 lexical query？

BGE-M3 具备多语言语义能力，所以中文原问题可以直接做 Dense 检索；英文论文中的专有名词对 BM25 更依赖英文词面，因此再生成英文 lexical query。两条查询分别优化语义召回和精确关键词召回。

### Q19：BM25 和 Dense 检索分别解决什么问题？

BM25 擅长模型名、缩写、数字和罕见词的精确匹配；Dense 检索擅长同义表达和跨语言语义相似。只用一种会分别面临语义缺失或词面缺失。

**代码锚点**：`app/services/retrieval.py::_bm25/_semantic`。

### Q20：RRF 为什么有用？

BM25 分数和向量相似度不在同一量纲，直接相加不稳定。RRF 只利用每个结果在两套排序中的名次，按 `1/(60+rank)` 累加，因此能低成本合并词面和语义召回。

**代码锚点**：`app/services/retrieval.py::_rrf`。

### Q21：Reranker 和 Embedding Model 有什么区别？

Embedding Model 独立编码 Query 和文档，适合从大量候选中快速召回；CrossEncoder Reranker 同时读取 Query 和候选文本，相关性判断更细但更慢，所以只重排有限候选。

**代码锚点**：`HybridRetriever._semantic/_rerank`。

### Q22：为什么检索 Chunk，最终引用 Sentence？

Chunk 上下文完整，适合召回和模型理解；Sentence 粒度小，适合判断某个 Claim 是否真的被原文支持，也方便前端精确高亮。PaperLens 因此把“召回单元”和“引用单元”分开。

### Q23：Text、Figure、Table 是否有三套完全独立的向量库？

没有。它们被转换成不同检索文本后进入同一篇论文的 Qdrant collection，共用 Dense 基础设施；但保留不同 ID、模态、工具和完成条件。BM25 同样在合并后的当前论文 Chunk 范围内运行。

### Q24：什么是 Evidence Contract？

它是三种模态统一的数据接口，包括 `evidence_id/type/page/bbox/content/section/metadata`。Critic 和 Answer Agent 只依赖该契约，不需要分别读取 Parser 的内部对象；Citation API 和前端也能使用同一标识回到原文。

**代码锚点**：`app/domain.py::EvidenceObject`。

### Q25：检索后怎样防止模型引用不存在的证据？

生成阶段要求模型返回 Evidence/Sentence ID，服务端检查 ID 是否属于候选映射，再由 verifier 判断 Claim 是否受内容支持。Agent 工具还绑定当前 Paper 对象，不能通过参数读取其他论文。

**边界**：这不是形式化证明；Figure/Table 的本轮 discovered-set 校验也只在候选集合非空时严格执行，Text 约束并不完全相同。

## 四、Router、LangGraph 与多智能体

### Q26：Router 如何选择 Standard 或 Agentic？

Router 用规则识别显式 Figure/Table 编号、视觉词、表格词、跨模态组合和复杂论证。单一正文事实走 Standard；涉及图、表或复合证据则走 Agentic，并输出所需 modalities 和路由理由。

**代码锚点**：`app/agent/router.py::route`。

### Q27：为什么 Router 不直接用 LLM？

当前路由特征较明确，规则更快、更便宜、可回归测试，也不会因为模型波动改变成本路径。代价是面对隐含视觉意图时泛化有限，需要通过评测集和规则维护补足。

### Q28：Supervisor 和 Planner 如何分工？

Planner 从问题提取引用、关键词和逻辑子任务；Supervisor 将逻辑任务按模态展开成 typed task，并分配 Agent、工具、步数、模型调用、Qwen 和超时预算。Planner 是 Supervisor 规划过程中的确定性组件。

**代码锚点**：`app/agent/planner.py`、`app/multi_agent/supervisor.py::plan`。

### Q29：LangGraph 中有哪些节点？

当前 StateGraph 包含 `plan`、`dispatch_specialists`、`evidence_critic`、`repair_dispatch`、`answer`、`structured_refusal`。Critic 后通过条件边进入 answer、repair 或 refusal。

**代码锚点**：`app/multi_agent/graph.py` 的 Graph builder。

### Q30：为什么使用 LangGraph，而不是一组 if/else？

核心价值不是“代码更高级”，而是显式 State、条件边、Reducer、checkpoint、节点级 Trace 和可控返工。普通 if/else 也能实现业务，但跨模态并行、局部重试和状态审计会更难维护。

### Q31：Graph State 保存什么？

State 保存问题、Router、Plan、Typed Tasks、Agent Results、Evidence、Critic 决策、预算、重试次数、Trace、错误和最终答案。它是整轮 Agentic 执行的数据总线，而不是对话长期记忆。

**代码锚点**：`app/multi_agent/state.py`。

### Q32：Reducer 做什么？

Reducer 定义并行节点更新怎样合并：列表追加，agent result 字典合并，Evidence 按 `evidence_id` 去重。它避免并行 Specialist 互相覆盖状态。

**边界**：Reducer 是确定性状态函数，不是 Agent，也不判断证据质量。

### Q33：三个 Specialist 分别负责什么？

Text Specialist 找句子级正文证据；Figure Specialist 找目标图片并按问题分析原图；Table Specialist 查找并读取结构化行列，必要时使用图像 fallback。拆分依据是不同工具、完成条件和模型预算，不是按任意步骤拆 Agent。

### Q34：Specialist 可以并行吗？

可以。Supervisor 当前生成的模态任务没有依赖；开启 `MULTI_AGENT_PARALLEL_ENABLED` 且任务多于一个时，Scheduler 使用有限线程池并发。普通应用默认配置关闭并行，但 v4 Dev 最终冻结评测明确启用了并行。

**边界**：并行缩短关键路径但会同时消耗 Provider 和预算，延迟改善不能全部归因于并行。

### Q35：Evidence Critic 做什么？

Critic 在生成答案前检查所需模态是否齐全、Figure/Table Evidence 是否可用、Specialist 是否报告结构化冲突以及是否还有重试预算。它输出 approved、partial、retry 或 refuse，并可只重做缺失任务。

**代码锚点**：`app/multi_agent/agents/critic_agent.py::review`。

### Q36：Evidence Critic 与答案 Judge 有什么区别？

Critic 是在线工作流中的确定性证据验收，决定能否进入答案合成；Judge 是离线评测组件，用 DeepSeek 或 Qwen-VL 比较候选答案与 Gold，计算 Answer Correct。一个影响产品回答，一个只评估结果。

## 五、Function Calling 与异常恢复

### Q37：PaperLens 的 Function Calling 是怎样实现的？

服务端按 Specialist 提供白名单 JSON Schema，DeepSeek 返回 tool call，服务端再做名称、JSON、类型、范围、目标 ID、重复调用和预算校验，执行本地 EvidenceTools，并把统一结果信封作为 tool message 回传模型。最终必须满足模态完成条件。

**代码锚点**：`app/multi_agent/function_calling.py`、`tool_registry.py`。

### Q38：每个 Specialist 能调用哪些工具？

Text：`search_text/read_sentence`；Figure：`search_figures/read_figure/analyze_figure_for_query`；Table：`search_tables/read_table/analyze_table_image_with_vlm`。模型看不到其他工具，也不能自行注册工具。

### Q39：为什么需要严格 JSON Schema 后再做本地二次校验？

Provider 的 Schema 约束减少格式错误，但不能替代服务端信任边界。本地校验还要处理未知工具、额外字段、类型、数值范围和业务语义，避免模型生成的参数直接进入文件、数据库或模型调用。

### Q40：结果信封和完成条件解决什么问题？

结果信封统一记录 status、result IDs、error、latency、cached 和检索信息，让编排层区分成功、空结果、partial 和异常。完成条件进一步要求 Figure 视觉题真正调用原图分析、Table 真正读取内容、Text 得到可引用句子，避免“工具 HTTP 成功”被误认为“研究任务完成”。

### Q41：模型能否读取任意论文、文件或 URL？

不能。Tool Schema 不暴露 `paper_id`、路径和 URL，EvidenceTools 在构造时绑定当前 Paper，只能访问内存中的当前论文对象及其受控图片。模型只能选择白名单工具和业务参数。

### Q42：如何处理重复或无效工具调用？

执行器为工具名和规范化参数生成签名，重复调用会被检测；未知工具、非法参数或候选外目标会产生 rejection 事件。若没有形成有效 Evidence 或完成条件仍未满足，Specialist 进入固定流程 fallback。

### Q43：Function Calling 失败时怎样恢复？

`BaseSpecialistAgent` 捕获 Function Calling 异常、空 Evidence 或流程未完成，记录 fallback 原因，然后执行固定的 search/read/analyze 流程。Provider/tool-completion 连续失败还可打开熔断，在冷却期直接走固定流程。

**冻结事实**：v4 Dev 最终报告有 2 次 Figure `required_visual_analysis_missing` fallback，最终任务仍成功。

### Q44：重试、熔断、fallback 和 Critic repair 有什么区别？

请求重试处理临时网络/HTTP 错误；熔断处理连续 Provider/tool-completion 故障；固定流程 fallback 处理单个 Specialist 的动态调用失败；Critic repair 处理跨 Specialist 证据覆盖不足。它们处在不同层，不能混为一个“自动重试”。

### Q45：当前模型和工具预算是硬上限吗？

它们是明确的受控预算：限制 Function Calling 轮次、工具步数、Qwen 次数和 Graph steps，并在节点前检查超时。但并行任务可能同时开始，正在执行的 Provider 调用不能被强制终止，答案合成还被实现保留，因此不能把它描述成分布式强一致的成本计费器。

## 六、Conversation、SSE 与长期记忆

### Q46：系统如何支持多轮对话？

Conversation、Message 和 Run 持久化在 SQLite。提交消息时用户 Message 与 queued Run 原子创建；后台执行器读取最近上下文，经 Query Resolver 得到独立问题，执行 RAG 后再原子写入助手 Message 和终态 Run。

**代码锚点**：`Repository.create_message_and_run/finish_conversation_run`、`_execute_conversation_run`。

### Q47：怎样防止重复提交和并发 Run？

客户端提供 `client_message_id`，Repository 通过唯一约束复用已有提交；数据库还建立部分唯一索引，限制每个 Conversation 只有一个 queued/running Run。检查和写入在同一事务中完成。

### Q48：短期上下文怎样控制长度？

Resolver 默认读取最近约 6 轮消息，并加入摘要、Conversation memory、论文学习 memory 和最近 Citation ID；超过字符预算时从最旧内容裁剪。当前摘要是确定性消息片段拼接，不是 LLM 摘要器。

### Q49：SSE 是怎样实现的？

`RunManager` 为每个 Run 保存有序事件 deque 和 Condition，支持 heartbeat、terminal event 和 `Last-Event-ID` 续接。Router、Graph Node 和 Tool 事件实时发布；完整答案校验后由 `_publish_answer()` 拆成 `answer.delta`。

**边界**：事件缓冲只在单进程内，不是 Redis Streams；答案不是 Provider Token Streaming。

### Q50：为什么不直接流式发送模型 Token？

未验证 Token 发出后无法撤回，可能先展示无证据或错误内容。PaperLens 选择先完成 Evidence Critic 和 Claim 校验，再分片发送安全答案；它牺牲首字延迟，换取更强的 grounded-answer 边界。

### Q51：取消请求为什么不能立即终止模型？

取消只在 SQLite 写入 `cancel_requested`，执行协程在关键边界检查。核心 RAG 和 Provider 调用在线程中同步运行，Python 无法安全杀死已发出的请求，所以当前是协作式取消，不是抢占式中断。

### Q52：长期记忆记录什么？

它记录用户在某篇论文上的问题、回答摘要、Evidence ID、模态、章节、来源 Run、分析版本、重要性、置信度、过期时间和备注。完成且可回答并有 Citation 的结果进入 active，否则进入 unresolved。

**代码锚点**：`app/services/long_term_memory.py::record_run`。

### Q53：active、unresolved、stale、archived、forgotten 分别是什么？

active 是当前版本已验证学习记录；unresolved 是尚未得到可验证答案的问题；stale 是旧解析版本的 active 记录；archived 是过期或容量回收；forgotten 是用户主动遗忘。置顶可取消 TTL，回填可从历史 Run 生成条目。

**边界**：当前重解析只把 active 改为 stale；隔离键是 `paper_id + local user_scope`，尚无登录鉴权和真实多租户。

## 七、Evaluation、LLMOps 与项目边界

### Q54：项目如何评测 RAG？

评测分开计算 Retrieval、Evidence、Execution Success、Answer Correct 和 Task Success。脚本调用与在线相同的 Router 和执行链，记录逐 Case Citation、Trace、延迟、失败与恢复；可回答题使用 DeepSeek/Qwen-VL Judge，不可回答题检查是否正确拒答。

**代码锚点**：`scripts/evaluate.py`、`app/evaluation/`。

### Q55：v4 Dev 的最终结果是什么？

v4 共 5 篇论文、60 题，Dev 36、Test 24。当前完整冻结的 Dev 36 取得 36/36 Task Success、100% Gold Citation Hit、93.33% Complete Gold Citation Hit、92.54%/96.94% Evidence Precision/Recall；平均延迟从 12.98 秒降到 10.53 秒，P95 从 41.35 秒降到 36.67 秒。

**报告**：`evals/report-dev-v4-final.json`。

### Q56：36/36 是否等于现实准确率 100%？

不等于。它只表示固定的 5 篇论文、36 条 Dev 上满足当前 Task Success 定义；其中 30 条可回答题进入模型裁判体系，6 条不可回答题按拒答逻辑判断。数据规模小、Judge 可能偏差、题目来自封闭论文集合，不能外推到开放世界。

**边界**：v4 的 24 条 Test 尚未正式冻结；历史 v2 Test 8/8 不能与 v4 Dev 合并。

### Q57：Gold Citation Hit 100% 为什么不等于 Evidence 完全正确？

Citation Hit 表示答案至少命中一个 Gold Evidence；Complete Hit 才关注全部预期证据组，最终是 93.33%。Precision/Recall 进一步衡量多引和漏引，因此必须联合解释，不能只报一个 100%。

### Q58：视觉答案为什么需要独立 Visual Judge？

文本 Judge 只读候选答案和文字证据，无法确认颜色、布局、箭头等视觉事实。Qwen-VL Visual Judge 在评测时以独立提示重新查看原图和 Gold/候选答案，避免直接复用在线 Figure Agent 的结论。

**边界**：它仍是模型裁判，不是人工 Gold 的完全替代；在线视觉 verifier 默认未开启。

### Q59：PaperLens 的 LLMOps 链路是什么？

FastAPI 暴露 Prometheus Metrics；`app/observability.py` 为 HTTP、Run、Node、Tool、Model 和 Memory 生成指标与 Span；OTel Collector 把 Trace 转发到 Tempo；Grafana 预置 System、Agents & Tools、Models & Memory 三套 Dashboard。页面 `run_id` 可关联 SQLite Run、应用 Trace 和 Tempo Span。

**边界**：Compose 只运行基础设施，FastAPI 仍由本地 Uvicorn 启动；没有完整告警、SLO 和自动质量门禁。

### Q60：如果让你继续升级，会先做什么？

第一，冻结代码、Prompt 和配置后完成 v4 Test，并加入人工抽检；第二，引入真实身份认证、租户隔离和访问审计；第三，把进程内 SSE/任务执行迁移到可持久化队列或事件流，支持多 Worker；第四，完善告警、SLO、成本预算和质量门禁。升级顺序先补评测可信度和运行可靠性，再追求更多 Agent 或新模型。

**不能声称**：当前已经具备 Kubernetes、分布式任务队列、Qdrant 集群、完整生产 LLMOps、MCP 接入或开放世界泛化能力。

## 使用方法

每次练习随机抽一题，并额外回答三句：

1. “这在代码的哪个类或函数？”
2. “失败时系统怎样恢复、降级或拒答？”
3. “当前有什么不能夸大的边界？”

如果只能复述本页答案而无法顺着函数调用解释，就回到系统掌握手册和数据流文档继续阅读。
