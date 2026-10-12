# PaperLens 学习笔记：项目主线、解析、检索与规划

> 日期：2026-10-07
> 范围：今天已经学习和讨论的内容。本文用于主动回忆，不替代 `paperlens-system-mastery-guide.md` 和当前代码。

## 1. 今天掌握到哪里

今天完成了以下知识块：

1. PaperLens 的业务目标和两条系统主链；
2. Evidence、Chunk、Sentence 与 Claim 校验；
3. Table 比较问题怎样回答，以及什么时候必须拒答；
4. Docling 与 PyMuPDF 的职责和对齐算法；
5. SQLite、Qdrant、本地文件和 BM25 的存储边界；
6. BM25、Dense、RRF、Reranker 的混合检索链；
7. 三层 Query Rewrite；
8. Router、Planner、Supervisor 的职责；
9. Planner 当前代码怎样生成子任务和检索词。

下一次应从“逻辑子任务与 typed task 的区别”继续，再进入 LangGraph State、Specialist、Evidence Critic 和 Function Calling。

---

## 2. PaperLens 是什么

### 2.1 一句话定位

PaperLens 是一个证据驱动的多模态论文阅读系统：它解析正文、Figure 和 Table，在 Standard RAG 与受约束的 Agentic RAG 之间分流，并把答案绑定到可以回到论文原文核验的页码、BBox 和 Evidence ID。

### 2.2 它解决什么问题

普通 PDF Chat 常见问题：

- 只能处理抽取出的文字，不能可靠查看 Figure/Table；
- 回答没有精确 Citation；
- 很难判断答案中的每个事实是否真的受原文支持；
- 失败时无法知道问题出在解析、检索、模型还是编排。

PaperLens 的核心不是“让模型自由回答”，而是：

> 让系统基于论文中的可验证证据回答，并允许用户回到具体页码和区域核验。

### 2.3 两条系统主链

```text
第一条：论文摄取链
PDF -> 校验/去重 -> 解析 -> Figure 离线理解 -> 索引 -> 持久化

第二条：在线问答链
问题 -> Query Rewrite -> Router -> 检索/Agent -> Evidence 校验
     -> Answer/Citation/Trace
```

必须区分：

- “论文摄取链”和“在线问答链”是两条系统主链；
- Standard RAG 和 Agentic RAG 是在线问答链内部的两种执行路线。

### 2.4 30 秒面试表达

> PaperLens 是一个证据驱动的多模态论文阅读系统，主要解决普通 PDF 问答无法可靠处理正文、Figure、Table，以及答案难以回到原文核验的问题。系统第一条主链负责论文上传、解析和索引，第二条主链负责用户提问、检索、证据校验和答案生成。问答链内部由 Router 在 Standard RAG 和 Agentic RAG 之间分流，最终输出包含可定位到论文页码和 BBox 的 Citation 以及运行 Trace。

相关代码：

- `../app/main.py`
- `../app/domain.py`
- `../app/agent/router.py`

---

## 3. Evidence、Chunk 与 Sentence

### 3.1 EvidenceObject

`app/domain.py::EvidenceObject` 的核心字段：

```text
evidence_id
type: text | figure | table
page
bbox
content
section
metadata
```

作用：把 Text、Figure、Table 统一成 Critic、Answer Agent、Citation API 和前端都能处理的证据格式。

典型流向：

```text
Parser/检索工具
 -> EvidenceObject
 -> Evidence Critic
 -> Answer Agent/Claim 校验
 -> Citation
 -> 页码和 BBox 原文定位
```

### 3.2 为什么检索 Chunk，引用 Sentence

Chunk：

- 保留较完整上下文；
- 适合 BM25、Dense 和 Reranker；
- 能帮助模型理解术语、方法和前后关系。

Sentence：

- 粒度更细；
- 适合检查一条 Claim 是否被原文直接支持；
- 适合生成精确 Citation 和 PDF 高亮。

因此设计是：

```text
Chunk 负责召回
Sentence 负责验证和引用
```

不能直接只检索 Sentence，因为句子太短时语义和上下文可能不足；不能把整个 Chunk 作为最终引用，因为引用范围会过宽。

相关代码：

- `../app/domain.py::Chunk/Sentence/sentence_evidence`
- `../app/tools/evidence_tools.py::search_text`
- `../app/services/reading.py::answer_with_evidence`

---

## 4. Claim 校验

### 4.1 Claim 是什么

Claim 是答案中的一条可以独立判断真假的原子结论。

例如：

> 论文使用对比学习损失，并在 CIFAR-10 上提升了 3.2%。

应拆成：

1. 论文使用了对比学习损失；
2. 论文在 CIFAR-10 上提升了 3.2%。

两条 Claim 必须分别找到直接证据，不能因为第一条有证据就默认第二条也正确。

### 4.2 Standard RAG 中的校验流程

```text
命中 Chunk
 -> 提取候选 Sentence
 -> DeepSeek 生成最多 3 条 Claim
 -> 每条 Claim 绑定 1～2 个 sentence_id
 -> 服务端检查 ID 合法性
 -> 第二次 DeepSeek 请求检查语义支持
 -> 只保留完全受支持的 Claim
```

第一层：`_candidate_claims()` 检查：

- Claim 文本不为空；
- Sentence ID 来自本轮候选；
- Sentence 真实存在；
- 单条 Claim 不引用过多证据。

第二层：`_verify_claims()` 检查：

- Claim 是否与问题直接相关；
- 引用句子是否支持 Claim 的全部细节；s
- 是否包含原文没有给出的数字、比较或推断。

结果：

- 全部通过：`status=ok`；
- 只有部分通过：删除不支持部分，`status=partial`；
- 全部不通过：`status=unsupported_claims`；
- ID 非法：`status=invalid_claim_evidence`；
- 没有句子证据：`status=no_sentence_evidence`。

### 4.3 已学示例

原文：

> Our method achieved 82.4% accuracy on CIFAR-10.

可以通过：

> 该方法在 CIFAR-10 上达到 82.4%。

不能通过：

> 该方法比基线提升了 3.2%。

原因：原文没有给出基线结果和提升幅度。

### 4.4 重要边界

生成与校验是同一套 DeepSeek 服务的两次独立请求和不同角色提示，不是两个完全不同的模型。Claim 校验提高可靠性，但不是形式化证明。

---

## 5. Table 比较问题

### 5.1 什么时候可以回答“比基线提升多少”

必须同时满足：

1. Docling 正确提取了目标 Table；
2. Router/Planner 把问题分配给 Table Specialist；
3. `search_tables()` 命中正确表格；
4. `read_table()` 得到可读的 Markdown、columns 和 rows；
5. 表格同时包含方法值和可比较的基线值；
6. 指标、数据集、split 和实验条件一致。

实际链路：

```text
问题
 -> Router: agentic/table
 -> Planner/Supervisor
 -> Table Specialist
 -> search_tables
 -> read_table
 -> Evidence Critic
 -> Answer Agent
 -> Claim verifier
 -> Table Citation
```

### 5.2 百分点与相对提升率

假设：

```text
Baseline = 79.2%
Ours     = 82.4%
```

绝对提升：

```text
82.4% - 79.2% = 3.2 个百分点
```

相对提升率：

```text
(82.4 - 79.2) / 79.2 ≈ 4.04%
```

面试和回答中必须区分“百分点”和“相对百分比”。

### 5.3 失败与边界

- 只有 Ours 数值、没有 Baseline：拒答提升幅度；
- 表格结构不可读：Table Agent 可尝试 VLM fallback，但默认关闭；
- 问题没有明确写 Table，当前规则不一定总能推断数据在表格；
- 两行实验条件不同：不能直接相减；
- 当前没有独立 Calculator Tool，简单算术由 DeepSeek 完成，再接受 Claim 校验，因此不是确定性算术系统。

相关代码：

- `../app/multi_agent/agents/table_agent.py`
- `../app/tools/evidence_tools.py::search_tables/read_table`
- `../app/services/reading.py::answer_from_evidence`

---

## 6. Docling 与 PyMuPDF

### 6.1 两者职责

Docling 擅长：

- 文档结构和章节；
- Figure、Table；
- OCR；
- 表格 Markdown；
- 图表渲染区域。

PyMuPDF 擅长：

- 稳定读取 PDF 页面和正文；
- 单词与文本行坐标；
- Sentence 切分；
- 句子级 BBox；
- PDF.js 原文高亮。

正确表述：

> Docling 提供结构和图表，PyMuPDF 提供正文、Sentence 和精确坐标；PaperLens 将 Docling 的章节语义映射到 PyMuPDF 的定位结果。

### 6.2 调用顺序

`PaperParser._parse_with_docling()` 的核心逻辑：

```python
# PyMuPDF 先生成正文、Sentence 和坐标
text_paper = self._parse_with_pymupdf(..., extract_figures=False)

# Docling 生成结构化 Chunk
structural_chunks = self._docling_chunks(document, paper_id)

# 将 Docling section 映射给 PyMuPDF 结果
self._align_docling_sections(text_paper, structural_chunks)

# 最终普通正文明确使用 PyMuPDF Chunks
chunks = text_paper.chunks
```

最终返回的普通正文和 Sentence 来自 `text_paper`；Docling Figure/Table 另行加入 Paper。

### 6.3 对齐算法

1. Docling BBox 转换成与 PDF.js/PyMuPDF 可比较的顶部原点坐标；
2. 每个 PyMuPDF Chunk 只与同页 Docling Chunk 比较；
3. 文本转小写、去掉非单词字符，最多取前 1200 字符；
4. `SequenceMatcher` 计算文本相似度；
5. BBox 交并比 IoU 计算空间重叠；
6. 合成分数：

```text
alignment_score
= 0.75 × text_similarity
+ 0.25 × bbox_overlap
```

7. 选择同页最高分候选；
8. 分数低于 `0.35` 时放弃映射；
9. 成功时只复制：

```python
pymupdf_chunk.section = docling_chunk.section
pymupdf_sentence.section = docling_chunk.section
```

对齐不是把两套正文合并，而是给 PyMuPDF 正文补充 Docling 章节语义。

### 6.4 为什么保留 PyMuPDF 正文和坐标

PaperLens 最终需要：

```text
sentence_id -> page -> bbox(es) -> PDF.js 高亮
```

PyMuPDF 从单词和文本行坐标生成 Sentence BBox，更适合作为精确 Citation 数据来源。

### 6.5 Docling 结果没有完全丢弃

- `structural_chunks` 还用于识别标题和摘要；
- Figure/Table 对象来自 Docling；
- Table Markdown 会额外生成可检索的 Table Chunk 和 Sentence，并绑定整张表 BBox。

面试重要程度：

- 必须掌握：两种解析器职责、对齐方式、失败降级；
- 重点掌握：Table Markdown 如何成为可检索证据；
- 知道即可：`structural_chunks` 还参与标题与摘要识别。

### 6.6 失败降级

- 单个 Chunk 对齐失败：保留 PyMuPDF 原结果，不强行复制错误章节；
- Docling 整体异常：完整降级为 PyMuPDF；
- fallback 通常能保留正文，但 Figure/Table、矢量图和复杂版面能力减弱。

相关代码：`../app/services/parser.py`。

---

## 7. SQLite、Qdrant、本地文件与 BM25

### 7.1 SQLite：业务事实来源

SQLite 保存：

- Paper、Chunk、Sentence；
- Figure/Table 元数据；
- 页码、BBox、Caption、Table Markdown；
- 文档哈希和处理状态；
- QA History、Agent Trace；
- Conversation、Message、Run；
- Figure Query Cache；
- 长期记忆条目和聚合。

它提供事务、唯一约束和关系状态，例如：

- `client_message_id` 幂等；
- 单会话只允许一个活跃 Run；
- 用户 Message 与 Run 原子创建；
- Assistant Message 与 Run 终态原子保存。

### 7.2 Qdrant：可重建的派生索引

Qdrant 保存：

- BGE-M3 生成的 Dense Vector；
- 少量 payload：`chunk_id/page/section`。

它负责回答：

> 哪些 Chunk 的向量与当前问题最相似？

完整 Evidence 内容仍来自 Paper/SQLite 数据。

### 7.3 本地文件系统

保存：

- 原始 PDF；
- Figure/Table 图片。

### 7.4 BM25

BM25 不持久化，每次检索时在当前论文的 Chunk 上运行时计算。

### 7.5 关键设计结论

```text
SQLite = 业务事实来源
Qdrant = 可从论文数据重新构建的 Dense 索引
```

不能只用 Qdrant，因为它不适合承担 Conversation/Run 事务、幂等约束、Sentence 关系和长期记忆生命周期。

Qdrant 不可用时，在线模式还可以降级到 BM25；SQLite 业务数据丢失时，仅靠向量和少量 payload 无法恢复完整系统。

相关代码：

- `../app/repository.py`
- `../app/services/retrieval.py::index`

---

## 8. 混合检索

### 8.1 完整链路

```text
Question
 -> Query Rewrite
 -> BM25
 -> BGE-M3 + Qdrant Dense Search
 -> RRF
 -> CrossEncoder Reranker
 -> Top-K Chunk
```

### 8.2 BM25 与 Dense

BM25：

- 根据关键词和词频匹配；
- 擅长模型名、缩写、数字、指标和精确术语；
- 容易漏掉同义表达和跨语言问题。

Dense：

- BGE-M3 将问题与 Chunk 编码成向量；
- Qdrant 按余弦相似度召回；
- 擅长语义改写、同义表达和跨语言检索；
- 可能召回语义相似但关键实体不准确的内容。

为什么两者都要用：

> BM25 保住精确词面，Dense 补充语义召回；单独使用任何一方都会产生系统性遗漏。

### 8.3 RRF

BM25 原始分数与 Dense 余弦相似度不在同一量纲，不能稳定地直接相加。

PaperLens 使用：

```text
RRF score = Σ 1 / (60 + rank)
```

含义：

- 只看排名，不比较不可比的原始分数；
- 倒数让前排结果贡献更大；
- 对多个检索列表求和，奖励被多种检索共同找到的 Chunk；
- `+60` 平滑排名差异，避免某一检索器的第一名过度主导。

例子：

```text
A：BM25 第 1，Dense 未命中
A = 1/61 ≈ 0.01639

B：BM25 第 2，Dense 第 1
B = 1/62 + 1/61 ≈ 0.03252
```

B 因为同时被两套检索认可，融合排名高于 A。

边界：RRF 丢失原始分数差距；`60` 是常见平滑常数，不是 PaperLens 训练出的最优参数。

### 8.4 Reranker

BGE-M3/Dense 分别编码问题和文档，速度快，适合大范围召回；CrossEncoder 同时读取 Query 和 Chunk，判断更准确，但每个问题都要重新计算 Query–Chunk 对。

PaperLens 因此采用两阶段检索：

```text
第一阶段：BM25 + Dense + RRF，目标是高召回
第二阶段：CrossEncoder 精排少量候选，目标是高精度
```

当前默认只重排融合排名前面的有限候选，而不是对整篇论文全部 Chunk 运行 CrossEncoder。

### 8.5 失败恢复

- Dense/Qdrant 失败：在线非严格模式可保留 BM25；
- Reranker 失败：可使用 RRF 排名；
- 严格评测模式：直接报告检索后端错误，避免偷偷改变评测条件。

相关代码：`../app/services/retrieval.py`。

---

## 9. 三层 Query Rewrite

### 9.1 第一层：多轮追问消歧

组件：`QueryResolver`。

输入：

- 最近约 6 轮消息；
- Conversation summary；
- Conversation memory；
- 当前论文学习记忆；
- 最近 Citation ID。

输出：

- `resolved_question`；
- `intent`；
- `needs_clarification`。

例子：

```text
上一问：SimCLR 基线取得了什么结果？
当前问：它为什么比基线好？

Resolved：论文中的方法为什么比 SimCLR 基线表现更好？
```

模型不可用时，规则 fallback 通常将上一问题与当前追问拼接。完整 Query Resolver 主要用于 Conversation API；直接 `/ask` 不具备同等的多轮消歧链。

### 9.2 第二层：跨语言检索改写

组件：`ReadingService.plan_query()`。

对于“中文问题 + 英文论文”：

```text
semantic_query：保留中文问题 -> BGE-M3 Dense Search
lexical_query：改写成英文关键词 -> BM25
```

例子：

```text
中文：作者如何避免表示坍塌？
semantic_query：作者如何避免表示坍塌？
lexical_query：prevent representation collapse
```

BM25 本身可以处理中文。当前 `_tokens()` 会把中文连续文本切成二元词；如果论文是中文，中文 BM25 可以生效。但英文论文没有中文词面，因此当前 Standard RAG 为 BM25 生成英文查询。

DeepSeek 不可用时，fallback 只保留中文问题中的英文模型名、数据集名、缩写、数字和百分比；纯中文概念主要依赖 Dense 跨语言召回。

尚未实现：同时运行中文 lexical query 与英文 lexical query 的双语 BM25 融合。

### 9.3 第三层：Specialist 子任务 Rewrite

组件：`Planner`。

发生在 Agentic RAG 的规划阶段。Planner 根据：

- 原问题；
- Router 识别的 modalities；
- Figure/Table 编号；
- 模型名、数字和关键词；

生成 Text/Figure/Table 子任务检索词。

Planner 是确定性 Python 规则组件，不是独立 LLM Agent，也不是独立 LangGraph 节点。

### 9.4 三层区别

| 层 | 解决的问题 | 输出 |
|---|---|---|
| Query Resolver | 多轮问题缺主语、指代不清 | 独立问题 |
| Reading Query Plan | 中文问题与英文论文词面不匹配 | semantic/lexical query |
| Planner | 复杂问题中不同模态需要不同检索词 | Specialist 子任务 query |

---

## 10. Router、Planner 与 Supervisor

### 10.1 Router

职责：

- 判断 Standard RAG 还是 Agentic RAG；
- 识别 text/figure/table modalities；
- 识别纯视觉、纯表格和复杂问题；
- 给出可解释的 route reason。

Router 当前是规则组件，不是 LLM 分类器。

### 10.2 Planner

职责：

- 生成逻辑子任务；
- 为子任务生成保留术语、编号和数字的检索词。

代码：`../app/agent/planner.py`。

#### `plan()` 的普通分支

未同时命中“主张”和“实验支撑”规则时，按 Router modalities 每种生成一个任务：

```text
text   -> Find method or result statements...
figure -> Find the original scientific figure...
table  -> Find structured table evidence...
```

#### `plan()` 的科研主张分支

问题同时包含：

- CLAIM：贡献、主张、创新、novelty；
- SUPPORT：实验、结果、证据、验证、消融；

则固定拆成：

1. 方法主张与贡献；
2. 支持主张的主要实验结果；
3. 比较或消融证据。

这是启发式规则，不是模型推理。

#### `rewrite()`

1. 合并任务描述和原问题；
2. 优先保留 `Figure 2/Table 3/图 2/表 3` 等连续引用；
3. 提取英文术语、数字、百分比和中文词组；
4. 删除停止词；
5. 不区分大小写去重；
6. 最多保留约 24 个词和 500 字符。

为什么先保留 Figure/Table 引用：如果普通分词把 `Figure 2` 拆散，后续按编号精确匹配会失效。

### 10.3 Supervisor

职责：

- 调用 Planner；
- 把逻辑任务按模态展开成 typed task；
- 映射 Agent：

```text
text   -> text_agent
figure -> figure_agent
table  -> table_agent
```

- 写入 instruction、query、required modalities 和 dependencies；
- 决定 sequential/parallel；
- 分配 steps、retry、model、Qwen-VL 和 timeout 预算；
- Critic 要求修复时，只选择目标任务并追加具体缺口。

代码：`../app/multi_agent/supervisor.py`。

### 10.4 三者不能混为一谈

```text
Router：决定走哪条路、需要哪些模态
Planner：决定要研究哪些逻辑问题、用什么词搜索
Supervisor：决定由哪个 Agent 怎样执行，并给多少预算
```

### 10.5 当前边界

- Planner 不真正理解复杂语义；
- 不能根据检索结果自由动态重规划；
- 科研主张分支重点生成 Text/Table 任务，未必覆盖所有 Figure 需求；
- Repair 主要由 Supervisor 把 Critic 缺口追加到原 Query；
- 多智能体是受规则、State、白名单和预算约束的工作流，不是自由自治。

---

## 11. 今天纠正过的理解

### 误区一

错误：两条主链是 Standard RAG 和 Agentic RAG。

正确：两条主链是论文摄取链和在线问答链；Standard/Agentic 是问答链内部路线。

### 误区二

错误：Chunk 主要用于 PDF 区域定位，Sentence 只用于高亮。

正确：Chunk 主要用于带上下文的召回；Sentence 同时用于 Claim 校验、精确引用和高亮。

### 误区三

错误：Docling 与 PyMuPDF 对齐后合并成一套新正文。

正确：普通正文、Sentence 和坐标保留 PyMuPDF 结果，Docling 主要补充 section；Figure/Table 另行使用 Docling 结果。

### 误区四

错误：SQLite 只保存对话历史。

正确：SQLite 是论文结构、会话、Run、Trace 和记忆的业务事实来源；Qdrant 才是可重建向量索引。

### 误区五

错误：中文关键词不需要 BM25。

正确：中文 BM25 对中文论文有效，当前分词器也支持；只是英文论文需要英文 lexical query 才有词面重合。

---

## 12. 面试速答卡

### 为什么同时使用 Docling 和 PyMuPDF？

> Docling 擅长论文结构、Figure、Table、OCR 和表格内容，PyMuPDF 擅长正文、Sentence 和精确坐标。PaperLens 通过同页文本相似度与 BBox 重叠把 Docling 章节映射到 PyMuPDF 结果，兼顾结构理解和原文定位；Docling 失败时降级到 PyMuPDF，但图表能力会减弱。

### 为什么检索 Chunk、引用 Sentence？

> Chunk 保留完整上下文，适合 BM25、Dense 和 Reranker 做召回；Sentence 粒度更细，适合 Claim 支持检查和精确 BBox 引用，所以采用 Chunk 召回、Sentence 验证与引用的两级设计。

### SQLite 和 Qdrant 怎样分工？

> SQLite 是业务事实来源，保存论文结构、Sentence/BBox、Conversation、Run、Trace 和长期记忆；Qdrant 只保存 BGE-M3 向量及少量 Chunk payload，用于 Dense 检索，是可以重新生成的派生索引。

### 为什么使用 BM25 + Dense？

> BM25 擅长模型名、缩写、数字等精确词面，Dense 擅长同义表达和跨语言语义召回。两者互补，RRF 按排名融合，再由 CrossEncoder 精排少量候选。

### 为什么使用 RRF？

> BM25 与 Dense 原始分数不可比，RRF 只使用排名，以倒数让高排名贡献更大，并通过求和奖励被多种检索共同命中的 Chunk；`+60` 用来平滑排名差异，避免单个检索器第一名过度主导。

### 三层 Query Rewrite 是什么？

> Query Resolver 补全多轮追问；Reading Query Plan 生成跨语言 semantic/lexical query；Planner 为不同 Specialist 生成子任务检索词。三层分别解决上下文省略、语言不匹配和跨模态任务差异。

### Router、Planner、Supervisor 怎样分工？

> Router 决定 Standard/Agentic 和所需模态；Planner 生成逻辑子任务与检索词；Supervisor 把任务展开成绑定 Agent、执行方式和预算的 typed task。

---

## 13. 主动回忆题

合上文档后回答：

1. PaperLens 的两条系统主链是什么？
2. Standard RAG 和 Agentic RAG 位于哪条主链内部？
3. EvidenceObject 为什么是核心数据契约？
4. 为什么检索 Chunk 而引用 Sentence？
5. Claim 的 ID 合法性检查与语义支持检查有什么区别？
6. 一张表满足哪些条件时才能回答“比基线提升多少”？
7. 百分点和相对提升率有什么区别？
8. Docling 与 PyMuPDF 各自负责什么？
9. 两套 Chunk 怎样计算对齐分数？
10. 对齐成功后最终保留谁的正文与坐标？
11. Docling structural chunks 还有哪些用途？
12. SQLite 为什么是事实来源？
13. Qdrant 为什么是可重建索引？
14. BM25 与 Dense 分别容易漏掉什么？
15. 为什么两套检索分数不能直接相加？
16. RRF 中倒数、求和和 `+60` 分别有什么作用？
17. 为什么不能用 CrossEncoder 扫描全部 Chunk？
18. 三层 Query Rewrite 的输入和输出分别是什么？
19. 中文问题为什么保留 semantic query，又生成英文 lexical query？
20. 中文能不能做 BM25？当前实现怎样分词？
21. Planner 为什么不是 LLM Agent？
22. Planner `plan()` 和 `rewrite()` 分别做什么？
23. Router、Planner、Supervisor 为什么不能混为同一个组件？
24. 逻辑子任务和 typed task 有什么区别？

最后一题是下一次学习的起点。
