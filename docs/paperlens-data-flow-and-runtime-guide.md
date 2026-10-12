# PaperLens 数据流转与运行机制

> 本文只回答“数据怎样流动、状态怎样变化、失败时怎样处理”。项目定位和概念总览见 `paperlens-system-mastery-guide.md`。

## 1. 阅读约定

每条链路统一按五个问题展开：

1. 输入是什么；
2. 经过哪些函数和组件；
3. 中间状态放在哪里；
4. 输出是什么；
5. 失败时怎样恢复、降级或拒答。

事实优先级：当前代码 > `evals/report-dev-v4-final.json` > 说明文档。

## 2. 进程启动与组件装配

`app/main.py` 在进程内装配：

- `Repository`：SQLite 业务持久化；
- `PaperParser`：Docling/PyMuPDF 解析；
- `HybridRetriever`：BM25、BGE-M3、Qdrant、RRF、Reranker；
- `ReadingService`：DeepSeek 查询改写、生成、校验和 Judge；
- `FigureUnderstandingService`：Qwen-VL 离线/在线图像理解；
- `QueryRouter` 与 Agent Executor；
- `ConversationService`、`QueryResolver`、`RunManager`；
- `LongTermMemoryService`；
- `LLMOps` 指标和 OpenTelemetry Trace。

应用启动时还会把数据库中遗留的 queued/running Run 标为 interrupted。该动作解决进程异常退出后的悬挂状态，但不会自动重放未完成请求。

## 3. 数据存储映射

| 数据 | 存储位置 | 主要生产者 | 主要消费者 |
|---|---|---|---|
| PDF、Figure/Table 图片 | 本地文件系统 | 上传/解析 | Evidence API、前端、Qwen-VL |
| Paper、Chunk、Sentence、Figure | SQLite | Parser/Repository | 检索、Agent、前端 |
| Dense 向量 | Qdrant | HybridRetriever.index | Dense 检索 |
| BM25 候选 | 运行时内存 | HybridRetriever._bm25 | 当前检索请求 |
| Conversation、Message、Run | SQLite | Conversation API | 多轮上下文、状态查询 |
| 应用 Agent Trace | SQLite | Standard/Agentic 执行器 | Trace API、评测 |
| LangGraph checkpoint | SQLite | LangGraphExecutor | 图状态恢复基础 |
| Figure 问题缓存 | SQLite | FigureUnderstandingService | 重复视觉问题 |
| 长期记忆条目/聚合 | SQLite | LongTermMemoryService | Query Resolver、学习记忆 API |
| SSE 事件 | 当前进程内存 | RunManager | EventSource 客户端 |
| Metrics | Prometheus | `/metrics` | Grafana |
| OTel Trace | Tempo | OTel Collector | Grafana/Tempo |

## 4. 链路一：上传新论文

### 4.1 输入

```http
POST /api/papers
Content-Type: multipart/form-data
file=<PDF>
```

入口：`app/main.py::upload`。

### 4.2 临时落盘与安全校验

`stage_pdf(file)`：

1. 分块读取 UploadFile；
2. 写入临时文件；
3. 累计大小并检查上限；
4. 检查扩展名和 `%PDF` 文件头；
5. 计算 SHA-256；
6. 返回临时路径和内容哈希。

无效文件在进入解析器前被拒绝，临时文件随后清理。

### 4.3 文档注册与去重

`Repository.reserve_upload()` 使用 `BEGIN IMMEDIATE`：

- 已有 completed：返回现有 `paper_id`，复用结果；
- 已有 processing：接口返回 202，避免并发重复解析；
- 新哈希：创建 `paper_id`、processing 注册记录和论文目录。

这一层解决的是业务幂等，不是语义去重；两个内容不同但文本相同的 PDF 仍有不同哈希。

### 4.4 解析

`process_paper()` 调用 `PaperParser.parse()`：

```text
Docling：章节、段落、Picture、Table、OCR、表格结构和导出图片
PyMuPDF：页面、文本块、单词、句子、BBox
对齐：同页 + 文本相似度 + BBox 重叠
输出：Paper(chunks, sentences, figures, metadata)
```

Text：

- Chunk 保留章节与页面；
- Sentence 由词坐标和行坐标形成，可有多个 BBox；
- Table 内容也可生成与表格 BBox 绑定的合成句子。

Figure：

- Picture 保存 Caption、页码、BBox、图片路径、邻近正文和相关句子 ID；
- Docling 坐标会转换成 PDF.js 使用的顶部原点坐标体系。

Table：

- 同样保存在 `Figure` 表中，但 `kind=table`；
- `content` 保存 Markdown，之后可解析成 columns/rows。

Docling 抛出异常时，解析器退回 `_parse_with_pymupdf()`。fallback 主要保证 Text；它只能提取部分嵌入式光栅图，不能等价替代 Docling 表结构。

### 4.5 Figure 离线理解

`FigureUnderstandingService.enrich_paper()` 遍历 Picture：

1. 读取导出原图；
2. 在内存中按需要放大，不修改原文件；
3. 将 Caption 和附近正文作为辅助上下文发送给 Qwen-VL；
4. 保存结构化 description。

输入是“整张图 + 论文局部上下文”，输出是用于以后召回的描述，不是针对某个用户问题的最终视觉证据。

单张图失败只写入 `status=error/unavailable` 和 uncertainty，上传继续。

### 4.6 建索引

`HybridRetriever.index(paper)`：

1. `index_chunks()` 合并 Text、Figure 和 Table 检索表示；
2. BGE-M3 批量生成归一化向量；
3. 为当前论文重建 Qdrant collection；
4. payload 保存 `chunk_id/page/section`；
5. 更新 `vector_indexed` 和 `index_version`。

Figure 检索文本来自 Caption 和离线视觉描述；Table 检索文本来自 Caption 和 Markdown；三者共用 Qdrant collection，但保留各自 ID 和模态。

索引异常返回 `False` 并记录 `last_error`。当前实现仍可保存论文，只是索引状态为不可用；因此“completed”不严格等于“向量索引一定成功”。

### 4.7 阅读卡片与持久化

`ReadingService.create_card()` 使用句子 ID 生成论文摘要卡片。模型失败时返回 unavailable 卡片，不阻止论文保存。

`Repository.save()` 在事务中重写当前论文的 Sentence、Chunk、Figure，并清理过期 Figure Query Cache。最终响应包含论文结构、卡片、解析器、索引状态和是否复用。

### 4.8 整体失败恢复

上传主流程异常时：

- 尝试删除对应 Qdrant collection；
- 释放文档注册占位；
- 删除新建论文目录；
- 返回明确 HTTP 错误。

这里清理的是本次新建资源，不会递归删除工作区或其他论文数据。

## 5. 链路二：论文重新分析

入口：`POST /api/papers/{paper_id}/reanalyze`。

1. SQLite 将分析版本加一并标记 processing；
2. 在临时工作目录重新解析；
3. 将 Figure/Table 图片移动到版本目录；
4. 重新做 Figure 离线理解、索引和阅读卡片；
5. 保存新版本；
6. 将旧版本 `active` 长期记忆标为 `stale`。

失败时 `finish_reanalysis_failure()` 将论文恢复为 completed 并保留原分析数据。当前版本失效只处理 active 记忆，unresolved 不会自动改为 stale。

## 6. 链路三：直接单轮问答

### 6.1 输入

```http
POST /api/papers/{paper_id}/ask
{
  "question": "...",
  "strategy": "hybrid-rerank",
  "limit": 5
}
```

### 6.2 处理

`_ask_paper()`：

1. 读取论文并检查 completed；
2. 若向量索引版本不匹配，尝试重建；
3. `QueryRouter.route(question)` 选择 Standard 或 Agentic；
4. 执行对应链路；
5. 保存 QA History 和应用 Trace；
6. 返回结构化结果。

直接接口没有 Conversation Message/Run，也不会使用多轮 Query Resolver。它适合调试内核和简单单轮调用。

## 7. 链路四：持久化会话问答

### 7.1 提交消息

```http
POST /api/papers/{paper_id}/conversations/{conversation_id}/messages
{
  "content": "它为什么这样设计？",
  "client_message_id": "client-generated-id"
}
```

`Repository.create_message_and_run()` 在一个事务中：

- 校验 Conversation 属于当前 Paper；
- 检查 `client_message_id` 是否已经处理；
- 阻止同一 Conversation 同时存在两个 queued/running Run；
- 插入用户 Message；
- 插入 queued Run。

接口返回 202，`RunManager.start()` 启动当前进程内的后台协程。

### 7.2 加载上下文

`_execute_conversation_run()` 读取：

- 当前 Conversation 与历史 Message；
- 最近约 6 轮消息；
- Conversation summary 与 memory；
- 当前论文长期记忆聚合；
- 最近 Citation ID。

`QueryResolver` 受最大字符数限制，超限时从最旧消息开始裁剪。

### 7.3 Query Resolver 输出

可能的 `intent`：

- `paper_question`：继续 RAG；
- `conversation_history`：回答“我刚才问过什么”等会话问题；
- `paper_learning_memory`：回答跨会话学习记录问题。

可能的结果：

- `resolved_question`：可独立检索的问题；
- `needs_clarification=true`：信息不足，返回 clarification；
- 直接由会话历史/学习记忆渲染答案，不进入论文 RAG。

会话或学习记忆回答没有论文 Citation，并会明确说明它是历史信息，不是论文证据。

### 7.4 执行和落库

RAG 执行通过 `asyncio.to_thread()` 调用同步内核。结果返回后：

1. 检查是否请求取消；
2. 发布经校验的答案分片；
3. 在一个事务中插入助手 Message 并结束 Run；
4. 更新 Conversation 标题、summary、memory；
5. 写入长期记忆条目；
6. 发布 completed/partial/failed/cancelled 终态事件。

只有结果 `status=ok` 映射为 completed，其他可返回内容通常映射为 partial。

## 8. Router 数据流

输入：解析后的独立问题。

规则特征包括：

- Figure/Table 显式编号；
- 颜色、形状、位置、箭头等视觉词；
- 行、列、指标、数值比较等表格词；
- 需要方法 + 图/表联合解释的复杂表达。

输出至少包含：

- `mode: standard | agentic`；
- `modalities`；
- 是否为纯视觉/纯表格；
- 路由理由。

Router 只决定入口和所需模态，不直接检索、不拆完整任务、不判断最终证据是否充分。

## 9. Standard RAG 数据流

```text
Question
 -> ReadingService.plan_query
 -> HybridRetriever.search_with_trace
 -> top Chunk + candidate Sentence
 -> DeepSeek 生成 atomic claims + sentence IDs
 -> claim verifier
 -> supported claims
 -> Answer + Sentence Citations + retrieval trace
```

### 9.1 混合检索

- BM25：对专有名词、缩写、数字、公式词面更敏感；
- Dense：BGE-M3 编码问题，Qdrant 计算余弦相似度；
- RRF：使用 `1/(60+rank)` 合并两个排序；
- Reranker：CrossEncoder 对候选 query-document 对进行重排。

在线模式中某个后端失败可降级；评测严格模式会抛错，防止指标在不同检索条件下被混在一起。

### 9.2 生成与校验

`answer_with_evidence()` 不直接让模型输出长答案，而是先要求最多若干条原子 Claim，每条绑定候选 Sentence ID。服务端先检查 ID 是否真实存在，再调用独立 verifier 判断支持程度。

无受支持 Claim 时返回结构化拒答；因此输出中的 `answerable/status/citations` 是数据流的一部分，不是仅靠自然语言表达“不知道”。

## 10. Agentic RAG 数据流

### 10.1 Graph State

State 保存：

- 原问题和 resolved question；
- Router 输出；
- Supervisor plan 与 typed tasks；
- `agent_results`；
- 去重后的 Evidence；
- Critic 决策；
- step/model/Qwen/retry/timeout 预算；
- Trace steps、warnings、errors；
- 最终 Answer。

列表字段通过加法 Reducer 合并，Evidence 按 `evidence_id` 去重，字典结果按 Agent/任务键合并。

### 10.2 图节点

```text
START
 -> plan
 -> dispatch_specialists
 -> evidence_critic
      approved/partial -> answer -> END
      retry            -> repair_dispatch -> evidence_critic
      refuse           -> structured_refusal -> END
```

Planner 逻辑在 Supervisor plan 节点内部，不是单独的 LLM 节点。

### 10.3 Specialist 调度

Supervisor 把逻辑任务按模态展开。默认正常应用配置是串行；启用并行且任务数大于一时，Scheduler 用最多有限工作线程并发执行。

串行超时只能在一次任务返回后判定；并行超时可以停止等待并取消尚未开始的 Future，但已经运行的 Python 线程和 Provider 请求不能被强制杀死。

### 10.4 Evidence Critic 与定向返工

Critic 从任务要求和结果中计算可用模态，并检查 Specialist 报告的冲突。若只缺 Figure，不会重做 Text/Table，而是选择首个可重试的缺失任务进入 repair。

重试耗尽或关键证据仍不可用时，进入 partial 或 structured refusal。

### 10.5 Answer

Answer Agent 将 State Evidence 转回 `EvidenceObject`，过滤不可用 Figure，再调用 `answer_from_evidence()`。它生成 Claim、绑定真实 Evidence ID、执行 verifier，并按 Router 所需模态检查覆盖率。

Critic 的 partial 状态会传递到最终状态，不能被流畅答案掩盖。

## 11. Function Calling 循环

每个 Specialist 的真实循环：

1. Tool Registry 根据 Agent 返回白名单 Schema；
2. DeepSeek `tool_completion()` 返回一个或多个 tool calls；
3. 检查模型轮次和工具步数预算；
4. 校验工具名、JSON、类型、必填字段、数值范围；
5. 执行模态相关语义约束；
6. 检查重复调用；
7. 执行本地 EvidenceTools；
8. 把统一结果信封作为 tool message 回传模型；
9. 检查模态完成条件；
10. 生成 Specialist result。

白名单：

- Text：`search_text`、`read_sentence`；
- Figure：`search_figures`、`read_figure`、`analyze_figure_for_query`；
- Table：`search_tables`、`read_table`、`analyze_table_image_with_vlm`。

如果循环异常、没有 Evidence、重复调用、超预算或没有满足完成条件，`BaseSpecialistAgent` 记录 `function_calling_fallback`，然后运行固定流程。

目标 ID 约束的真实程度：Figure/Table 在已发现候选集合非空时拒绝集合外 ID；Text 没有完全同等的 discovered-set 约束。所有工具仍绑定当前 Paper 对象，模型不能通过参数跨论文读取。

## 12. 三种模态的在线工具链

### 12.1 Text

```text
search_text(query)
 -> HybridRetriever 排名非 Table Chunk
 -> 每个 Chunk 选择最多有限相关 Sentence
 -> Sentence Evidence(page, bboxes, content)
```

固定 Text 流程通常不需要额外 `read_sentence`，因为 `search_text` 已返回句子证据；Function Calling 模式可按需读取特定句子。

### 12.2 Figure

```text
search_figures(query)
 -> 显式 Figure N 精确匹配或混合检索
 -> read_figure(id)
 -> analyze_figure_for_query(id, original_question)
 -> Qwen-VL 原图分析结果
```

查询缓存键包含 paper、figure、规范化 question 和 cache version。缓存命中可以避免重复视觉调用；跨模态依赖仍会基于当前问题重新整理。

### 12.3 Table

```text
search_tables(query)
 -> 显式 Table N 精确匹配或混合检索
 -> read_table(id)
 -> columns/rows/content
 -> 可选 VLM fallback
```

Table VLM fallback 受配置、Qwen 预算、图片存在性和目标 ID 约束，默认不是主路径。

## 13. Evidence、Citation 与原文定位

答案 Citation 返回 Evidence ID、类型、页码、BBox 和必要内容。前端通过：

```text
GET /api/papers/{paper_id}/evidence/{evidence_id}
GET /api/papers/{paper_id}/pdf
GET /api/papers/{paper_id}/figures/{figure_id}
```

读取原文并高亮对应区域。

Sentence 可能跨多行，因此保存 `bboxes`；兼容接口也保留单个 `bbox`。Figure/Table 通常对应一个 Docling item 区域。

Evidence ID 校验能防止模型引用完全不存在的对象，但不能把 verifier 变成形式化证明；部分支持、解析误差和 Gold 标注不完整仍需评测和人工抽检发现。

## 14. SSE、重连与取消数据流

### 14.1 事件发布

`RunManager.publish(run_id, event, data)`：

- 为事件分配递增 ID；
- 放入固定长度 deque；
- 唤醒等待该 Run 的 SSE 客户端；
- terminal 事件后关闭运行通道。

事件包括运行状态、路由、节点开始/结束、工具调用、Critic、答案分片和终态。

### 14.2 重连

客户端携带 `Last-Event-ID` 时，服务端从仍在内存缓冲区中的下一条事件继续发送。超过缓冲窗口、服务重启或连接到另一个 Worker 后，旧事件可能无法恢复，但最终 Run 和 Message 仍可从 SQLite 查询。

### 14.3 答案分片

`_publish_answer()` 接收已经完成校验的完整字符串，按固定字符数发布 `answer.delta`。所以：

- 工作流事件是真实时的；
- 答案正文不是 Provider Token Streaming；
- 未经校验的模型草稿不会暴露给用户。

### 14.4 取消

取消接口写 SQLite `cancel_requested`，并发布取消请求事件。执行协程在进入 RAG 前、阻塞调用返回后和持久化前检查。已发出的同步 Provider 请求仍会运行到返回，这是当前协作式取消的边界。

## 15. 长期记忆数据流

### 15.1 写入

每轮 Conversation Run 结束后，`LongTermMemoryService.record_run()` 根据结果创建条目：

- completed + answerable + Citation -> active；
- 其他可记录结果 -> unresolved。

使用来源 Run 和内容 fingerprint 避免重复回填。用户已标 forgotten 的条目不会因同样内容再次写入而自动复活。

### 15.2 聚合

`refresh_aggregate()` 生成：

- 已学习问题摘要；
- explored sections；
- 已查看 Figure/Table ID；
- unresolved questions；
- 生命周期计数。

Query Resolver 获取的是有长度上限的导航信息，不会把所有历史回答塞进 Prompt。

### 15.3 生命周期

- retention 到期：未置顶条目归档；
- 超过每篇论文容量：优先归档低重要性、未置顶条目；
- pin：取消 TTL；
- unpin：重新设置 TTL；
- reanalyze：旧版本 active -> stale；
- forget：状态设为 forgotten；
- edit：可改问题、摘要、重要性、备注或状态。

当前隔离键为 `paper_id + user_scope`，默认 scope 是 `local`，没有身份认证。

## 16. 可观测数据流

### 16.1 指标

```text
FastAPI /metrics
 -> Prometheus 每 10 秒抓取 host.docker.internal:8010
 -> Grafana Prometheus datasource
 -> System / Agents & Tools / Models & Memory Dashboard
```

指标覆盖 HTTP、Run、活跃 Run、节点耗时、工具调用和拒绝、Function fallback、熔断、模型请求、Token 和记忆操作。

### 16.2 Trace

```text
app/observability.py spans
 -> OTLP HTTP :4318
 -> OpenTelemetry Collector
 -> Tempo
 -> Grafana Tempo datasource
```

Conversation 根 Span 包含 request/run/conversation/paper 等安全关联字段；子 Span 覆盖检索、图节点、工具和模型调用。应用 Trace API 返回的是 SQLite 中保存的业务执行 Trace，与 Tempo 分布式 Trace 相关但不是同一个存储。

### 16.3 日志安全

结构化日志会过滤 Authorization、API Key、完整 Prompt/正文等敏感值，只保留必要 ID、状态、长度和截断预览。生产环境仍需外部密钥管理、访问审计和数据保留策略。

## 17. Evaluation 数据流

`scripts/evaluate.py`：

1. 按 dataset `split` 读取 Case；
2. Retrieval 模式以严格后端条件计算排名指标；
3. RAG 模式调用与应用相同的 Router/Standard/Agent Executor；
4. 收集 Citation、Agent Trace、延迟、失败和恢复信息；
5. 可回答文本题使用 DeepSeek Judge；
6. 视觉题可使用独立 Qwen-VL Visual Judge 重新看原图；
7. 不可回答题按预期拒答判断；
8. 汇总 Retrieval、Evidence、Execution、Answer、Task 指标；
9. 报告写入完整 configuration 和逐 Case 记录。

`Task Success` 不是单纯的语言相似度，它要求执行、可回答性、答案正确性和证据条件共同满足。

最新冻结结果只覆盖 v4 Dev 36 条。不要把历史 v2 Test 8/8 与 v4 Dev 合并，也不要在完成 v4 Test 前声称其结果。

## 18. 三个端到端示例

### 18.1 简单正文事实

输入：“作者使用什么损失函数？”

```text
Resolver: 独立问题
Router: standard/text
Rewrite: 英文 lexical query + 多语言 semantic query
Retriever: BM25 + Dense -> RRF -> Reranker
Reading: Chunk -> Sentence -> Claim -> verifier
输出: answer + sentence_id + page/bbox + retrieval trace
```

失败：没有受支持句子时拒答，不进入 Figure Agent。

### 18.2 Figure 视觉问题

输入：“Figure 2 中红色箭头连接了哪些模块？”

```text
Router: agentic/figure/pure_visual
Planner: 保留 Figure 2 和视觉关系词
Figure Specialist: 精确找 Figure 2 -> 读图 -> Qwen-VL 按问题分析
Critic: 检查 answerable 和 Figure Evidence
Answer: 基于视觉 Evidence 合成并引用 Figure 2
```

失败：Function Calling 未执行视觉分析时触发固定流程；Qwen-VL 仍不可用则 partial/refusal。

### 18.3 Figure + Text + Table 综合问题

输入：“结合方法图和消融表，哪个组件对应最大的性能提升，作者如何解释？”

```text
Router: agentic/text+figure+table
Supervisor: 三个 typed tasks
Scheduler: 按配置串行或并行
Text: 找作者解释
Figure: 找模块关系并看原图
Table: 读取行列和数值
Reducer: 按 evidence_id 合并
Critic: 检查三模态覆盖与冲突
Answer: 跨模态 Claim + Citation
```

失败：缺一个关键模态时只定向返工该任务；仍失败则明确 partial 或拒答。

## 19. 运行边界速查

| 容易误解的说法 | 当前事实 |
|---|---|
| 上传是异步任务队列 | 线程池执行但请求等待；没有分布式队列 |
| 所有 completed 论文都已成功建向量索引 | 索引失败仍可能保存 completed 降级状态 |
| Planner 是 LLM Agent | 当前是确定性规则组件 |
| Graph 在线使用 conversation:turn checkpoint | 执行器支持，但 main 当前未传入，通常用 Run UUID |
| Function Calling ID 全部来自本轮搜索 | Figure/Table 仅在候选非空时严格检查，Text 约束不同 |
| SSE 是模型 Token Streaming | 工作流事件实时；答案在校验后按字符分片 |
| cancel 会立刻终止模型请求 | 只能在协作边界结束 Run |
| 长期记忆可以直接证明论文事实 | 只能辅助导航，答案仍需当前 Evidence |
| Compose 启动完整应用 | Compose 只含基础设施，FastAPI 通常由本地 Uvicorn 启动 |
| 36/36 等于真实准确率 100% | 只是 5 篇论文上的 v4 Dev judge-enabled 结果 |

## 20. 关键代码入口

- API 与总流程：`app/main.py`
- 领域对象：`app/domain.py`
- 配置：`app/config.py`
- SQLite：`app/repository.py`
- 解析：`app/services/parser.py`
- 检索：`app/services/retrieval.py`
- 查询计划、生成与校验：`app/services/reading.py`
- 多轮消歧：`app/services/query_resolver.py`
- SSE：`app/services/run_manager.py`
- 长期记忆：`app/services/long_term_memory.py`
- Router/Planner：`app/agent/`
- LangGraph：`app/multi_agent/graph.py`
- Function Calling：`app/multi_agent/function_calling.py`
- 工具 Schema：`app/multi_agent/tool_registry.py`
- Evidence 工具：`app/tools/evidence_tools.py`
- 多模态：`app/multimodal/`
- 评测：`app/evaluation/`、`scripts/evaluate.py`
- 可观测：`app/observability.py`、`observability/`
