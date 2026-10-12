# PaperLens LLMOps 与长期记忆工程指南

> 本文聚焦两项容易被混淆的能力：如何观察一次 AI 请求，以及如何保存用户在某篇论文上的学习进度。两者都已实际实现，但都仍是单机、无鉴权条件下的工程版本。

## 1. 目标与边界

LLMOps 解决：

- 请求是否成功；
- 时间花在哪个节点、工具或模型；
- 是否发生 fallback、rejection、retry 或熔断；
- 模型调用量和 Token 如何变化；
- 某个页面 Run 如何关联到 Trace。

长期记忆解决：

- 用户在同一篇论文中已研究过什么；
- 哪些问题仍未解决；
- 哪些 Figure/Table 已查看；
- 论文重解析、过期、遗忘后如何管理历史记录。

当前没有实现完整的生产告警、SLO、质量门禁、登录鉴权、多租户 RBAC、分布式任务队列或跨 Worker SSE。

## 2. LLMOps 数据流

```text
FastAPI 请求
 -> app/observability.py 记录结构化日志、Metrics、Span/Event
 -> /metrics
    -> Prometheus
    -> Grafana Dashboard
 -> OTLP HTTP
    -> OpenTelemetry Collector
    -> Tempo
    -> Grafana Trace 查询
```

`docker-compose.yml` 只包含：

- Qdrant；
- Prometheus；
- Tempo；
- OpenTelemetry Collector；
- Grafana。

FastAPI 不在 Compose 中，通常由本地 Uvicorn 在 8010 端口启动。Prometheus 配置通过 `host.docker.internal:8010/metrics` 抓取应用。

## 3. Metrics

`app/observability.py::LLMOps` 定义的主要指标类别：

### 3.1 HTTP 与 Run

- HTTP 请求数量、状态和耗时；
- Run 状态计数；
- 活跃 Run；
- Run 端到端延迟。

用途：先判断问题是接口层失败、Run 未完成，还是单纯长尾延迟。

### 3.2 Graph Node 与 Tool

- LangGraph 节点耗时；
- Tool 调用次数、状态和耗时；
- Function Calling outcome；
- `function_calling_fallback`；
- Tool rejection；
- Circuit state。

用途：定位是 Supervisor、Specialist、Critic、Answer，还是某个检索/视觉工具变慢或降级。

### 3.3 Model 与 Token

- Provider/operation 维度的请求数量；
- 模型请求耗时；
- 输入/输出 Token；
- 成功、失败和重试状态。

用途：区分 DeepSeek 生成、Function Calling、Qwen-VL 离线理解、在线视觉分析和 Judge 等调用。

### 3.4 Memory

- 各生命周期状态的记忆数量；
- record/read/edit/forget/backfill/expire 等操作结果。

用途：观察记忆是否持续增长、是否发生过期归档，以及回填/编辑是否失败。

## 4. 三套 Grafana Dashboard

### System Overview

- Runs / 5m；
- Failed HTTP / 5m；
- Active Runs；
- Run P95；
- Run outcomes；
- Model request duration P95。

### Agents & Tools

- Tool calls / 5m by status；
- LangGraph node P95；
- Function fallback 累计值；
- Rejected tool calls 累计值；
- Circuit State。

### Models & Memory

- Tokens / 5m；
- Model requests / 5m；
- Memory items by lifecycle；
- Memory operations / 5m。

Dashboard JSON 位于 `observability/grafana/dashboards/`，实际运行截图位于 `docs/assets/`。这些文件能证明项目做过观测链路验收，但不能证明任意时刻容器都在线。

## 5. Trace 与关联字段

### 5.1 两类 Trace

1. **应用级 Agent Trace**：保存在 SQLite，通过 `/api/papers/{paper_id}/traces/{run_id}` 返回，包含路由、检索、Agent step、fallback、Citation 等业务信息。
2. **OpenTelemetry Trace**：保存在 Tempo，包含 Span 层级、耗时、状态和 Event。

二者相关但不是同一个对象或存储。

### 5.2 关联一次页面回答

Conversation Run 创建 `run_id`。根 Span `paperlens.conversation_run` 绑定安全的：

- request ID；
- run ID；
- conversation ID；
- paper ID；
- 截断后的 question preview。

页面获得 `run_id` 后，可以：

1. 查询 SQLite Run 状态；
2. 请求应用 Agent Trace；
3. 在 Tempo 按 Span attribute 查找同一轮；
4. 对照 Prometheus/Grafana 同时间窗口的指标。

### 5.3 慢请求排查顺序

1. System Overview：确认 Run P95 与失败率；
2. 对比 Model P95：判断是否 Provider 长尾；
3. Agents & Tools：定位慢 Node、Tool、fallback 或 Circuit；
4. Tempo：按 `run_id` 查看单次调用树；
5. 区分 BM25、Dense、Reranker、DeepSeek、Qwen-VL、锁等待和冷启动；
6. 回到应用 Trace 核对路由、模态、缓存和 Citation。

不要只看总耗时猜测瓶颈，也不要把所有延迟都归因于多 Agent。

## 6. 结构化日志与隐私

日志和 Span 字段经过 `safe_fields()`/`span_fields()` 处理，过滤或截断：

- API Key；
- Authorization；
- 完整 Prompt；
- 大段论文正文；
- 其他敏感值。

保留的是 ID、状态、类型、长度、耗时和安全预览。生产化仍需要外部密钥管理、租户级访问审计、集中日志权限和数据保留策略。

## 7. 长期记忆的数据模型

长期记忆不是把全部聊天记录重新塞进 Prompt，而是为每篇论文保存有来源、生命周期和容量限制的学习条目。

每条 `paper_memory_item` 包含：

- `user_scope`、`paper_id`；
- 来源 `conversation_id/run_id`；
- `analysis_version`；
- 原问题和 resolved question；
- 回答摘要；
- Evidence ID、Evidence Type、Section；
- answerable/verified/confidence；
- importance；
- status；
- expires_at、pinned、note；
- fingerprint 和时间字段。

SQLite 唯一约束和 fingerprint 用于避免同一来源或同一内容重复写入。

## 8. 写入规则

`LongTermMemoryService.record_run()` 在 Conversation Run 完成后调用：

- `run.status=completed`、`answerable=true` 且存在 Citation：写入 `active`；
- 其他可记录的 partial/refusal/缺证据结果：写入 `unresolved`。

重要性是启发式评分，受问题类型、Evidence 数量、跨模态程度、是否完成和用户编辑影响。它用于容量回收顺序，不是一个训练出的认知重要度模型。

记忆控制类问题本身不会再次写成新的论文学习记忆，避免“查询记忆”不断污染记忆。

## 9. 聚合与读取

`refresh_aggregate()` 从条目生成：

- interactions；
- explored sections；
- viewed Figure/Table ID；
- unresolved questions；
- lifecycle status counts。

Query Resolver 只获取受长度限制的聚合和条目摘要，用来理解“继续上次的问题”“我之前看过哪些图”等意图。

记忆不进入 Citation 的事实来源。即使记忆说“论文证明了 X”，回答论文事实时仍须重新检索当前版本 Evidence。

## 10. 生命周期

### active

当前分析版本中，完成、可回答且带 Citation 的学习记录。

### unresolved

尚未得到可验证答案、缺少关键证据或运行只得到 partial 的问题。它用于提示后续继续研究，不代表错误或失败数据应被删除。

### stale

论文重新分析后，旧版本的 active 记录。原因是 Evidence ID、页面或 BBox 可能变化，旧记录不应继续视为当前可靠导航。

当前 `Repository.mark_memory_stale_for_version()` 只更新 `status='active'` 的旧版本条目；旧 unresolved 不会自动变成 stale。这是必须按代码表述的边界。

### archived

未置顶条目超过 retention，或每篇论文超过容量限制时被归档。归档保留历史，但不进入默认活跃聚合。

### forgotten

用户主动遗忘的条目。相同 fingerprint 后续 upsert 时仍保留 forgotten，避免自动复活。

## 11. 置顶、过期、容量与编辑

- Pin：清除过期时间，防止自动归档；
- Unpin：重新按 retention 设置过期时间；
- Capacity：超过每篇论文上限时，优先归档未置顶、低重要性条目；
- Edit：可编辑问题、摘要、重要性、备注或合法状态；
- Forget：显式设为 forgotten；
- Backfill：从已有 completed/partial Run 补建记忆，跳过记忆控制类问题。

默认配置中的 retention 和容量是单机产品策略，不应表述成通用的长期记忆最优参数。

## 12. 版本失效

论文重新分析成功后，`LongTermMemoryService.invalidate_for_version()` 调用 Repository：

```text
旧 analysis_version + active -> stale
当前 analysis_version         -> 保持现状
unresolved                    -> 当前代码不会自动 stale
```

随后重建聚合。即使旧条目 pinned，也不能继续作为 active 事实导航，因为版本安全优先于用户置顶。

## 13. 隔离与权限边界

当前读写条件始终包含：

```text
paper_id + user_scope
```

默认 `user_scope="local"`。这能防止不同论文的记忆直接混合，也为未来用户字段预留接口，但没有：

- 登录身份；
- Token/Session 鉴权；
- RBAC；
- 真实租户边界；
- 每租户加密和审计。

因此只能说“论文级、本地 scope 隔离”，不能说“已实现多用户长期记忆平台”。

## 14. 相关 API

- `GET /api/papers/{paper_id}/learning-memory`
- `DELETE /api/papers/{paper_id}/learning-memory`
- `GET /api/papers/{paper_id}/learning-memory/items`
- `PATCH /api/papers/{paper_id}/learning-memory/items/{memory_id}`
- `POST /api/papers/{paper_id}/learning-memory/backfill`

Conversation 中询问“我之前研究过什么”会由 Query Resolver 识别为 `paper_learning_memory` intent，直接渲染记忆结果，不进入论文 RAG，并明确说明这不是论文证据。

## 15. 启动与验收

基础设施：

```powershell
docker compose up -d
```

应用仍需本地启动，例如：

```powershell
uvicorn app.main:app --host 0.0.0.0 --port 8010
```

验收顺序：

1. `/api/health` 与 `/api/ready`；
2. `/metrics` 是否包含 `paperlens_` 指标；
3. Prometheus target 是否能抓取 8010；
4. Grafana 三套 Dashboard 是否有新请求数据；
5. 页面创建一个新 Run；
6. 用 `run_id` 查询 SQLite Run、应用 Trace 和 Tempo；
7. 完成一次带 Citation 的会话，检查 active memory；
8. 产生一次无法回答问题，检查 unresolved；
9. 测试 pin/edit/forget/backfill；
10. 重新分析论文，确认旧 active 变 stale。

## 16. 常见故障

| 现象 | 优先检查 |
|---|---|
| Grafana 没数据 | FastAPI 是否在 8010、Prometheus target、`/metrics` |
| Metrics 有数据但 Tempo 无 Trace | OTEL 开关、4318 endpoint、Collector、Tempo datasource |
| 找不到某次 Run Trace | 是否使用当前进程新请求、Span attribute 中的 run_id |
| fallback 数突然增加 | Function Calling provider、Schema rejection、完成条件、Circuit |
| Qwen 调用过多 | 路由、Figure 精确匹配、查询缓存、Qwen budget |
| 长期记忆没有 active | Run 是否 completed、answerable=true、是否有 Citation |
| 重解析后旧记忆仍 unresolved | 当前代码只将 active 标为 stale，属于现有行为 |
| SSE 重连丢历史事件 | 是否重启、跨 Worker、超过进程内 buffer |

## 17. 面试边界总结

可以说：

> 我实现了可运行的 LLMOps 观测闭环，用 Prometheus、OpenTelemetry、Tempo 和 Grafana关联 Run、Agent、Tool 与 Provider 调用；同时实现了按论文隔离、带生命周期和来源追踪的长期记忆。

不能说：

- 已经具备完整生产告警和 SLO；
- 已实现分布式 Trace/任务恢复平台；
- SSE 可跨多 Worker 持久续播；
- 长期记忆等同于向量化的事实知识库；
- 已有真实用户鉴权和多租户隔离；
- stale 逻辑覆盖所有旧版本状态；
- 记忆内容可以直接代替论文 Citation。
