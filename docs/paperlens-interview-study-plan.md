# PaperLens AI 应用开发实习面试学习计划

> 目标不是背诵项目介绍，而是能从代码说明一个请求的输入、状态、处理、输出、失败恢复、设计取舍和能力边界。

## 1. 学习材料怎样分工

| 材料 | 用途 | 什么时候看 |
|---|---|---|
| `paperlens-system-mastery-guide.md` | 项目事实总纲 | 每个主题开始前 |
| `paperlens-data-flow-and-runtime-guide.md` | 逐请求追踪数据流 | 阅读代码时对照 |
| `paperlens-ai-application-interview-qa.md` | 练习面试表达 | 理解主题之后 |
| `llmops-and-long-term-memory.md` | 深入观测与记忆 | 学完主链路后 |
| `README.md` | 项目入口、启动和冻结结果 | 最后检查表述一致性 |
| `evals/report-dev-v4-final.json` | 最新冻结评测事实 | 学 Evaluation 时 |

不要先背题库。正确顺序是：看总纲建立地图 → 跟代码走一遍 → 合上文档复述 → 再用题库暴露盲点。

## 2. 统一学习方法

每个主题都完成以下六步：

1. **定位入口**：找到 API、类或函数；
2. **识别输入**：请求体、对象、State 或数据库记录是什么；
3. **画处理链**：按调用顺序写出组件；
4. **识别输出**：返回值、Evidence、数据库状态和事件是什么；
5. **追失败路径**：异常怎样降级、重试、partial 或拒答；
6. **口头复述**：用“实现事实—设计原因—当前边界”三层表达。

每次遇到新术语，先用一句通俗话解释，再补技术定义。例如：

> RRF 就是把 BM25 和向量检索各自的排名合并，而不是直接比较两套不可比的原始分数；技术上它按 `1/(k+rank)` 累加排名分。

## 3. 学习优先级

### P0：面试前必须掌握

- 业务目标与 Evidence Contract；
- 上传与解析主链；
- Standard RAG；
- 三层 Query Rewrite；
- Router、Supervisor、Planner、Specialist、Critic、Answer 的职责；
- Function Calling 约束与回退；
- 冻结评测结果及其边界。

### P1：重点掌握

- Figure/Table 的解析与在线分析；
- Conversation、Run、SSE 与取消；
- 长期记忆生命周期；
- SQLite、Qdrant、Checkpoint 和缓存的数据边界；
- Prometheus、Tempo、Grafana。

### P2：用于深入追问

- 并发、超时和模型预算并非绝对硬限制的原因；
- 解析坐标对齐与 BBox 转换；
- 评测指标实现细节；
- 单机到生产架构的演进方案；
- 当前代码中约束不完全对称的地方。

## 4. 第一天：建立项目地图和 Evidence 思维

### 学习目标

能用 30 秒介绍项目，并解释 PaperLens 为什么不是普通 PDF Chat。

### 文档阅读

1. `README.md` 的系统架构、核心能力和已知边界；
2. `paperlens-system-mastery-guide.md` 第 1～4、18 节；
3. `paperlens-data-flow-and-runtime-guide.md` 第 1～3 节。

### 代码阅读

1. `app/domain.py`
   - `EvidenceObject`
   - `Chunk`
   - `Sentence`
   - `Figure`
   - `Paper`
2. `app/main.py`
   - 全局组件装配；
   - API 路由列表；
   - `_ask_paper()` 的 Standard/Agentic 分支。
3. `app/config.py`
   - 只找与 parser、retrieval、agent、conversation、memory、observability 相关的开关。

### 动手任务

- 画一张“摄取链 + 问答链”图；
- 写出 Text/Figure/Table 进入统一 Evidence 后保留和丢失了什么信息；
- 从 `EvidenceObject` 反推前端如何回到 PDF 高亮。

### 必须能回答

1. PaperLens 的用户价值是什么？
2. 为什么 Evidence 比答案字符串更重要？
3. Chunk、Sentence 和 Evidence 的职责有何不同？
4. 当前系统两条主链是什么？
5. 哪些能力明确没有实现？

### 通过标准

不看文档，用 30 秒、2 分钟两种长度介绍项目；主动说出“没有训练模型、不是开放世界 100% 准确、不是自治 Agent”。

## 5. 第二天：上传、解析、索引与原文定位

### 学习目标

能从一个 PDF 文件追到 SQLite、Qdrant 和原文 BBox。

### 文档阅读

- 总纲第 5、11、14 节；
- 数据流第 4、5、13 节。

### 代码阅读顺序

1. `app/main.py::stage_pdf`
2. `app/repository.py::reserve_upload`
3. `app/main.py::process_paper`
4. `app/services/parser.py::parse`
5. `_parse_with_docling`
6. `_parse_with_pymupdf`
7. `app/multimodal/figure_understanding.py::enrich_paper`
8. `app/services/retrieval.py::index/index_chunks/figure_chunks/table_chunks`
9. `app/repository.py::save`

### 动手任务

- 为上传链写一张表：阶段、输入、输出、存储、失败处理；
- 找到 Docling BBox 向 PDF.js 坐标转换的位置；
- 比较 Docling 主路径和 PyMuPDF fallback 能力；
- 解释为什么 Figure 离线描述适合检索，但不能直接回答所有视觉问题。

### 必须能回答

1. 为什么同时使用 Docling 和 PyMuPDF？
2. SHA-256 和文档注册表解决什么问题？
3. Text、Figure、Table 分别怎样形成索引文本？
4. BM25 是否持久化在 Qdrant？
5. Qdrant 失败后论文一定上传失败吗？
6. 重新分析失败怎样保护旧数据？

### 通过标准

选择一篇已解析论文，从一个 Citation ID 反查其 Sentence/Figure/Table、页码、BBox 和原文件资源。

## 6. 第三天：Standard RAG、混合检索与 Query Rewrite

### 学习目标

能解释简单问题为什么不需要多智能体，以及检索和生成怎样分开校验。

### 文档阅读

- 总纲第 6～8 节；
- 数据流第 6～9 节。

### 代码阅读顺序

1. `app/services/query_resolver.py::resolve`
2. `app/services/reading.py::plan_query`
3. `app/agent/router.py::route`
4. `app/services/retrieval.py::search_with_trace`
5. `_bm25/_semantic/_rrf/_rerank`
6. `app/tools/evidence_tools.py::search_text`
7. `app/services/reading.py::answer_with_evidence`
8. Claim verification 相关私有函数

### 三层 Rewrite 对照表

| 层 | 输入问题 | 产物 | 解决的问题 |
|---|---|---|---|
| Query Resolver | “它为什么这样做？” | 独立 resolved question | 多轮指代和省略 |
| Reading plan | 中文论文问题 | semantic + English lexical query | 中英文词面不匹配 |
| Planner | 复杂跨模态问题 | Specialist 子任务 query | 不同模态需要不同搜索词 |

### 动手任务

- 选择一个中文问题，手写 BM25 与 Dense 各自可能召回的内容；
- 用三行伪代码写 RRF；
- 找一条 Standard 答案，区分 retrieved Chunk、candidate Sentence、final Citation；
- 模拟 Dense 失败，说明在线与严格评测模式的不同。

### 必须能回答

1. 为什么不只使用向量检索？
2. 为什么 RRF 合并排名而不是直接相加原始分数？
3. Reranker 与 Embedding Model 有什么不同？
4. 为什么检索 Chunk，最终引用 Sentence？
5. Claim verifier 与 Evidence Critic 的区别是什么？
6. DeepSeek 不可用时 Query Rewrite 怎样降级？

### 通过标准

从 `_ask_paper()` 的 Standard 分支开始，不看指南说出完整函数调用顺序和每步数据类型。

## 7. 第四天：LangGraph、多智能体与 Function Calling

### 学习目标

能解释 Agentic RAG 的必要性、状态图和模型权限边界，而不是只会罗列 Agent 名称。

### 文档阅读

- 总纲第 9、10、17 节；
- 数据流第 10～12 节。

### 代码阅读顺序

1. `app/agent/router.py`
2. `app/agent/planner.py`
3. `app/multi_agent/state.py`
4. `app/multi_agent/supervisor.py`
5. `app/multi_agent/graph.py`
6. `app/multi_agent/agents/base.py`
7. `text_agent.py/figure_agent.py/table_agent.py`
8. `critic_agent.py/answer_agent.py`
9. `app/multi_agent/tool_registry.py`
10. `app/multi_agent/function_calling.py`
11. `app/tools/evidence_tools.py`

### 动手任务

- 画出 Graph 的六个节点和三类条件边；
- 选择 Figure + Table 问题，手写 Supervisor 可能生成的 typed tasks；
- 列出每个 Specialist 的白名单工具和完成条件；
- 跟踪一次 `function_calling_fallback`：触发原因、固定流程、最终结果；
- 找出 Figure/Table target ID 校验与 Text 约束不对称的位置。

### 必须能回答

1. Planner 为什么不是独立 LLM Agent？
2. Reducer 为什么不是 Agent？
3. 工具调用返回 success 为什么不等于 Specialist 成功？
4. Evidence Critic 如何定向返工？
5. 模型能决定什么，不能决定什么？
6. 熔断、重试、固定流程 fallback 分别解决什么失败？
7. 并行与模型预算为什么不是绝对强一致限制？
8. checkpoint 已实现到什么程度？

### 通过标准

用一个跨模态请求讲清楚 State 的变化，并至少说明三个失败分支：无 Evidence、视觉分析缺失、关键模态不足。

## 8. 第五天：多模态深挖

### 学习目标

能说明 Figure 与 Table 为什么不能简单当作普通文本，并能定位视觉问题失败在哪一层。

### 文档阅读

- 总纲第 5.2～5.4、11 节；
- 数据流第 4.4～4.6、12、18.2～18.3 节。

### 代码阅读顺序

1. `app/domain.py::Figure.as_evidence`
2. Parser 中 Picture/Table 解析、邻近文本和坐标转换
3. `app/multimodal/qwen_vl_client.py`
4. `app/multimodal/figure_understanding.py`
5. `EvidenceTools.search_figures/read_figure/analyze_figure_for_query`
6. `EvidenceTools.search_tables/read_table/analyze_table_image_with_vlm`
7. Figure/Table Specialist 固定流程

### 动手任务

- 比较 Figure 离线 description、在线 query analysis、Visual Judge 三次 Qwen-VL 调用的目的与输入；
- 解释 Figure Query Cache 的 key 和 version；
- 追踪一个 Table Markdown 如何变成 columns/rows；
- 用“Figure 2 中红色箭头是什么”设计一套排错顺序。

### 必须能回答

1. 为什么不能只用 Caption 回答视觉问题？
2. 为什么要先离线理解，再按问题看原图？
3. Visual Judge 为什么必须独立重新看图？
4. Table 何时使用结构化读取，何时使用 VLM fallback？
5. PyMuPDF fallback 对图表能力有什么损失？

### 通过标准

面对“系统命中了 Figure 2 但答案仍错”这一故障，能按解析、索引、查询、缓存、Qwen、Critic、答案校验逐层定位。

## 9. 第六天：Conversation、SSE、长期记忆与 LLMOps

### 学习目标

能区分业务持久化、运行事件、短期上下文和跨会话记忆，并能从 `run_id` 定位性能问题。

### 文档阅读

- 总纲第 12～16 节；
- 数据流第 7、14～16 节；
- `llmops-and-long-term-memory.md` 全文。

### 代码阅读顺序

1. `app/repository.py` 的 Conversation/Message/Run schema 与事务
2. `app/services/conversation.py`
3. `app/services/query_resolver.py`
4. `app/services/run_manager.py`
5. `app/main.py::_execute_conversation_run/_publish_answer`
6. `app/services/long_term_memory.py`
7. Repository memory 方法
8. `app/observability.py`
9. `docker-compose.yml` 与 `observability/`

### 动手任务

- 画 Message、Run、SSE Event 的生命周期；
- 解释客户端重复提交为何不会重复创建 Run；
- 区分 Conversation summary、Conversation memory、Paper learning memory；
- 对 active/unresolved/stale/archived/forgotten 各写一个真实例子；
- 从 Dashboard 指标写出 Run P95 上升的排查顺序。

### 必须能回答

1. SSE 为什么不是 Token Streaming？
2. 为什么取消不能立即终止 Provider 调用？
3. 服务重启后什么能恢复，什么不能恢复？
4. 长期记忆为什么不是论文事实证据？
5. 论文重解析后哪些记忆会 stale？
6. 当前用户隔离实现到什么程度？
7. 应用 Trace API 与 Tempo Trace 有什么区别？
8. Compose 为什么不等于完整应用部署？

### 通过标准

能用一个 `run_id` 同时解释 SQLite Run、SSE 通道、应用 Trace、OpenTelemetry Span 和长期记忆来源之间的关系。

## 10. 第七天：Evaluation、边界与模拟面试

### 学习目标

能准确解释冻结结果，面对追问时不把指标、数据集和生产能力说大。

### 文档与报告阅读

1. 总纲第 15、18、20 节；
2. 数据流第 17、19 节；
3. `scripts/evaluate.py`；
4. `app/evaluation/`；
5. `evals/questions.v4.jsonl`；
6. `evals/report-dev-v4-final.json` 顶层 configuration/metrics；
7. `evals/report-dev-v4-baseline.json`；
8. `evals/report-test-langgraph-final.json`，只作为历史 v2 对照。

### 必须记准的冻结事实

- v4：5 篇论文、60 条问题；Dev 36、Test 24；
- 当前完整冻结：v4 Dev 36；
- Task Success 36/36，但不是现实准确率 100%；
- 30 条可回答题进入模型裁判体系，6 条不可回答题按拒答判定；
- Gold Citation Hit 100%；
- Complete Gold Citation Hit 93.33%；
- Evidence Precision/Recall 92.54%/96.94%；
- 平均延迟 12.98s -> 10.53s；
- P95 41.35s -> 36.67s；
- 冻结回归记录为 177 passed；
- v4 Test 24 条未正式冻结；
- 历史 v2 Test 8/8 不能与 v4 Dev 合并。

### 动手任务

- 用自己的话解释六种指标：Recall@K、MRR、Citation Hit、Complete Hit、Precision、Recall；
- 为“执行成功但任务失败”和“答案正确但 Citation 不完整”各举一例；
- 比较 baseline/final，找出不能全部归因于并行的原因；
- 准备五条不能夸大的边界；
- 完成一次 30 分钟模拟面试。

### 通过标准

面试官追问“36/36 是不是 100% 准确”“为什么不跑 Test 调参”“模型 Judge 可信吗”时，能直接纠正问题前提并给出报告证据。

## 11. 三种长度的项目介绍

### 30 秒

目标 + 两条主链 + 一个最重要的可信性设计 + 一组不夸大的冻结结果。

### 2 分钟

按以下顺序：

1. 业务目标；
2. 上传解析与索引；
3. Standard/Agentic 路由；
4. Evidence 与校验；
5. Conversation/Memory/LLMOps；
6. 结果和边界。

### 5 分钟

不要继续堆组件名，选择一个 Figure + Table 真实问题，从 API 输入讲到 Citation 和 Trace，再补失败恢复与评测。

## 12. 回答任何技术问题的统一结构

```text
直接结论
 -> 当前实现在哪里
 -> 输入是什么
 -> 处理过程是什么
 -> 输出是什么
 -> 为什么这样设计
 -> 失败怎样恢复/拒答
 -> 当前没实现什么
```

示例：

> 当前 SSE 不是模型 Token Streaming，而是实时工作流事件加校验后答案分片。Conversation Run 执行时，RunManager 会实时发布路由、节点和工具事件；完整答案经过 Evidence/Claim 校验后才由 `_publish_answer()` 按字符切片。这样避免将不可撤回的未验证内容发给用户，但首字延迟更高，事件缓冲也只在单进程内。

## 13. 最终验收清单

### P0

- [ ] 能画上传链和问答链；
- [ ] 能解释 EvidenceObject 全部关键字段；
- [ ] 能走通 Standard RAG；
- [ ] 能区分三层 Query Rewrite；
- [ ] 能画 LangGraph 节点和条件边；
- [ ] 能解释每个 Agent 的输入和输出；
- [ ] 能解释 Function Calling 约束和固定流程 fallback；
- [ ] 能准确复述冻结指标和边界。

### P1

- [ ] 能解释 Figure/Table 解析、索引和在线工具链；
- [ ] 能解释 Conversation、Run、SSE、取消和重连；
- [ ] 能解释长期记忆五种状态；
- [ ] 能按 run_id 排查一次慢请求；
- [ ] 能说明 SQLite、Qdrant、Tempo、Prometheus 各保存什么。

### P2

- [ ] 能指出至少三处“文档容易说强、代码实际更弱”的地方；
- [ ] 能设计 v4 Test 正式冻结流程；
- [ ] 能提出从单机走向多 Worker/多租户的演进方案；
- [ ] 能讨论模型 Judge 偏差和人工抽检；
- [ ] 能在十分钟内从 API 定位到关键实现函数。

## 14. 明确禁止的面试表述

- “我训练/微调了 DeepSeek 或 Qwen-VL。”
- “36/36 证明系统准确率 100%。”
- “已经完成 v4 Test。”
- “多 Agent 可以自由自治和互相聊天。”
- “SSE 是完整 Token Streaming。”
- “长期记忆已经支持真实多租户。”
- “Docker Compose 部署了整个 FastAPI 应用。”
- “系统已经上 Kubernetes/Qdrant 集群。”
- “PaperLens 主项目使用了 MCP。”
- “Evidence 校验能够形式化证明答案正确。”

掌握项目的标准不是能背出更多技术名词，而是能在被追问时迅速回到当前代码、数据和报告。
