# B18 review eval 真实 LLM 接入 — GitHub Issue 草稿

**仓库**：`FluentWork/fluentwork-backend`
**优先级**：P0（W3 中期补齐；解锁 I16 完整转录浮层）
**估时**：1 dev-day
**关联文档**：
- `docs/30_技术方案/39_FluentWork后端技术架构设计V2_0_2026-09-03.md` §5.4（review eval）
- `docs/30_技术方案/48_FluentWork_V2_REST接口契约冻结_2026-09-06.md` §1.3.3 + §1.3.7
- `docs/30_技术方案/46_F3_B4_收藏置顶与重播完整规格_2026-09-03.md` V1.2 §B4
- `docs/30_技术方案/33_FluentWork-Prompt工程与语料库设计文档.md` §六
- `docs/40_研发流程与协作/47_D1_D5_开放技术决策备忘录_2026-09-03.md` V1.1 §D-1
- `docs/40_研发流程与协作/55_FluentWork_V2_W3_代码层启动包_2026-09-06.md`
- 父 Issue：无（独立启动；D-1 Ark Mini 已 ✅ 拍板）

---

## 🎯 目标

实施会话结束后 review eval 真实 LLM 接入（D-1 拍板：Ark Mini 异步批处理）：
- `GET /api/v1/sessions/:session_id/review` 返回 `eval` 字段（score / dims / suggestions）
- 会话结束 → 后台异步触发 eval job → 落 `reviews` 表 → 客户端轮询 /review 直到 `status=ready`
- 落地 3 维评分（grammar / fluency / vocabulary）+ 个性化建议

> **范围**：仅 review eval；闪测判定（B22）是另一条独立链路（D-3 Ark Mini 同步调用）

---

## 🚧 阻塞条件

- **D-1 已 ✅ 拍板**（Ark Mini + 深度思考异步）
- **0008 迁移已就绪** ✅（49_ § 0008：alter_utterances_add_llm_eval，已包含 `llm_eval_score` 等字段）
- C-1 凭证：Ark Mini API Key + endpoint（W3 前到位）
- B16 AIOrchestrator 已 CLOSED（提供 `LLMClient` 接口，本 Issue 仅消费）
- staging 已 apply 0008 迁移

---

## 📋 实施步骤

### 1. 包结构 `internal/review/`

```
internal/review/
├── service.go        # Review 业务逻辑
├── evaluator.go      # LLM 调用封装（消费 B16 LLMClient）
├── eval_prompt.go    # Prompt 构造（D-1 拍板模板）
├── http.go           # RegisterRoutes
├── job.go            # 异步任务入队
└── service_test.go
```

### 2. Prompt 模板（`eval_prompt.go`，D-1 拍板）

```yaml
system: |
  You are a strict workplace English speech evaluator.
  Given a TARGET_PHRASE, the USER's actual utterance (with ASR text and confidence), and the
  CONTEXT (scene_tag + function_tag), output strict JSON:
  {"score": 0.0-1.0, "dims": {"grammar": 0.0-1.0, "fluency": 0.0-1.0, "vocabulary": 0.0-1.0},
   "suggestions": ["<15 chars zh>", "<15 chars zh>", ...], "judge_reason": "<20 chars zh>"}
  Strict grading: penalize fillers, broken grammar, off-topic content.
  Output ONLY JSON. No markdown. No commentary.

user: |
  CONTEXT: {{utt.scene_tag}} / {{utt.function_tag}}
  TARGET: {{utt.chunk_en}} ({{utt.intent_zh}})
  USER_ASR: {{utt.asr_text}} (confidence={{utt.asr_confidence}})
  DIFFICULTY: {{utt.difficulty}}
  HISTORY_NEAR_HITS: {{#each hits}}- {{this.chunk_en}} ({{this.intent_zh}}){{/each}}
```

### 3. Evaluator（`evaluator.go`）

```go
type Evaluator struct {
    llm     LLMClient  // B16 接口注入
    logger  *slog.Logger
}

type EvalResult struct {
    Score        float64
    Dims         EvalDims
    Suggestions  []string
    JudgeReason  string
}

func (e *Evaluator) Evaluate(ctx context.Context, utt UtteranceForEval, hits []RecordedHit) (*EvalResult, error) {
    prompt := BuildEvalPrompt(utt, hits)
    
    // D-1：Ark Mini 默认同步 1.5s；如超时切换深度思考异步（待 B16 实现）
    raw, err := e.llm.Complete(ctx, LLMRequest{
        Model: "ark-mini",
        SystemPrompt: prompt.System,
        UserPrompt:   prompt.User,
        Temperature:  0.3,
        MaxTokens:    400,
        Timeout:      3 * time.Second,
    })
    if err != nil {
        return nil, err
    }
    
    var eval EvalResult
    if err := json.Unmarshal([]byte(raw), &eval); err != nil {
        return nil, fmt.Errorf("eval_json_parse: %w", err)
    }
    
    return &eval, nil
}
```

### 4. Service 层（`service.go`）

```go
type Service struct {
    store  Store
    eval   Evaluator
    job    JobQueue  // B16 worker 注入
    logger *slog.Logger
}

type Review struct {
    SessionID   string
    Utterances  []UtteranceWithEval  // JOIN utterances + reviews
    Eval        EvalSummary           // 整个会话的聚合
    Status      string                // "pending" | "ready" | "failed"
}

type UtteranceWithEval struct {
    UtteranceID string
    Text        string
    Score       *float64    // nullable
    Dims        *EvalDims
    Suggestions []string
    AudioURL    string
    StartedAtMs int64
    EndedAtMs   int64
}

type EvalSummary struct {
    Score        float64  // 整 session 平均
    Dims         EvalDims
    Suggestions  []string  // top 5 高频建议
    Status       string
}

func (s *Service) EnqueueEval(ctx context.Context, sessionID string) error {
    return s.job.Enqueue(EvalJob{SessionID: sessionID, EnqueuedAt: time.Now()})
}

func (s *Service) GetReview(ctx context.Context, sessionID string) (*Review, error) {
    return s.store.GetReview(sessionID)
}

func (s *Service) RunEvalJob(ctx context.Context, sessionID string) error {
    // 1. 拉取 session 所有 utterances
    utts, err := s.store.GetUtterancesForEval(ctx, sessionID)
    if err != nil { return err }
    
    // 2. 拉取最近命中（通过 B19 的 RecentHits）
    hits, _, err := s.hitsService.RecentHits(ctx, sessionID, 8, 0.7)
    if err != nil { return err }
    
    // 3. 逐条 utterance 调 LLM（限速 1 QPS 防 rate limit）
    sem := make(chan struct{}, 1)
    results := make([]EvalResult, len(utts))
    for i, utt := range utts {
        sem <- struct{}{}
        go func(i int, utt Utterance) {
            defer func() { <-sem }()
            res, _ := s.eval.Evaluate(ctx, utt, hits)
            results[i] = res
        }(i, utt)
        time.Sleep(time.Second)  // 限速
    }
    
    // 4. 落 reviews 表 + 更新 session.review_status=ready
    return s.store.SaveReviews(ctx, sessionID, utts, results)
}
```

### 5. Job 入队触发点

- `internal/session/service.go` 的 `EndSession` 流程末尾：
  ```go
  if err := s.reviewSvc.EnqueueEval(ctx, sessionID); err != nil {
      s.logger.Warn("enqueue eval failed", "session_id", sessionID, "err", err)
  }
  ```
- B16 worker 在 cron / job queue 触发 `RunEvalJob`

### 6. HTTP 路由（48 §1.3.3 + §1.3.7）

```go
// internal/review/http.go
func RegisterRoutes(rg gin.IRouter, h *Handler) {
    rg.GET("/sessions/:session_id/review", h.accounts.RequireAuth(), h.GetReview)
}
```

```go
func (h *Handler) GetReview(c *gin.Context) {
    sessionID := c.Param("session_id")
    review, err := h.svc.GetReview(c, sessionID)
    if err != nil {
        httpjson.Error(c, err)
        return
    }
    httpjson.OK(c, review)
}
```

### 7. OpenAPI 同步

```yaml
  /sessions/{session_id}/review:
    get:
      summary: 会话回顾（带 LLM 评价）
      tags: [sessions]
      parameters: [{$ref: '#/components/parameters/SessionID'}]
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema: {$ref: '#/components/schemas/Review'}
        '202':
          description: 评价生成中
          content:
            application/json:
              schema:
                type: object
                properties:
                  status: { type: string, enum: [pending] }
                  estimated_ready_at: { type: string, format: date-time }
        '401': {$ref: '#/components/responses/Unauthorized'}
        '404': {$ref: '#/components/responses/NotFound'}
```

---

## ✅ 测试矩阵

| ID | 场景 | 输入 | 预期 |
|---|---|---|---|
| T-B18-1 | session 结束 → 触发 eval | session 20 句 | 20 条 reviews 记录 + 评分落库 |
| T-B18-2 | GET /review 在 ready 后 | session | 200 + Review 全字段 |
| T-B18-3 | GET /review 在 pending 中 | session | 202 + status=pending + estimated_ready_at |
| T-B18-4 | 评分维度 | Ark Mini mock 返回 | 3 维均落库（grammar / fluency / vocabulary） |
| T-B18-5 | suggestions 长度 | Ark Mini mock | 长度 ≤ 30 字符（中文 15 字内） |
| T-B18-6 | 异常 ASR (空字符串) | utt.asr_text="" | score=0.0, suggestions=["无可识别内容"], 不报 5xx |
| T-B18-7 | LLM 超时 (3s) | Ark Mini 延迟 5s | 返回 default EvalResult（score=0.5, suggestions=["系统繁忙，请稍后重试"]）+ metric `review_eval_timeout_total` |
| T-B18-8 | LLM 响应非 JSON | mock 返回无效文本 | fallback + metric `review_eval_parse_error_total` |
| T-B18-9 | 跨用户权限 | user A GET user B session review | 403 forbidden |
| T-B18-10 | session 无 utterances | 空 session | EvalSummary.score=0.0, suggestions=["本次会话无内容"], 200 |
| T-B18-11 | 限速 | 20 句并发 | 总耗时 ≥ 20s（1 QPS 节流） |
| T-B18-12 | 撤销删除后 review | A4 删除后撤销 | reviews 表与 sessions 同步复活 |
| T-B18-13 | OpenAPI 字段对齐 | 48 §1.3.3 + §1.3.7 | 字段一致 |
| T-B18-14 | 性能 | session 50 句 eval 任务 | 总耗时 P95 ≤ 60s |

---

## 📐 估时拆分

| 步骤 | 估时 |
|---|---|
| 1. 包结构 + 数据模型 | 0.1 dev-day |
| 2. Prompt 模板 + Evaluator 实现 | 0.2 dev-day |
| 3. Service + Job 入队 | 0.3 dev-day |
| 4. HTTP 路由 + OpenAPI 同步 | 0.1 dev-day |
| 5. 测试 + Mock LLM 集成 | 0.3 dev-day |
| **合计** | **1 dev-day** |

---

## 🎯 DoD

- [ ] `internal/review/` 整个包实施
- [ ] `0008` 迁移已在 staging apply
- [ ] `go test ./internal/review/...` 14 个测试通过
- [ ] 端到端冒烟：本地 mock LLM + 真实 session 走通完整 review
- [ ] 性能：session 50 句 eval 任务 P95 ≤ 60s
- [ ] OpenAPI 同步：`/sessions/{session_id}/review` 含 200/202/401/404
- [ ] metric：`review_eval_timeout_total` + `review_eval_parse_error_total` 上线
- [ ] PR 通过 OpenCodeReview + 1 名后端 maintainer review
- [ ] Issue description 链接本文 + 55_ 启动包 + 33 §六 + 47 §D-1 + 48 §1.3.3+§1.3.7
- [ ] 完成后状态写到 53_ §Layer 2 表格（CLOSED ✅）

---

## 🔗 关联 Issue

- 上游：无（D-1 已 ✅ 拍板）
- 上游依赖：B16 AIOrchestrator（提供 `LLMClient` 接口；不在本 Issue 范围但实施时需要 B16 已 CLOSED）
- 下游：I16 iOS 完整转录浮层（消费 review.eval 字段）
- 平行：B17 / B19 / B22 / B25 / B21 / B23 / B24（同 W3 批次）

---

## 📝 备注

- **D-1 深度思考异步**：本期先用 Ark Mini 同步实现（带 3s 超时 + fallback）；深度思考异步由 B16 worker 后续扩展
- **B19 联动**：本 Service 通过 `hitsService.RecentHits` 拉取最近命中用于 Prompt 上下文；B19 落地后本 Issue 才能端到端跑通
- **LLM 限速 1 QPS**：本期硬编码；V2.1 改为可配置（per-user 配额）
- **撤销删除 (B22 A4) 后 review 复活**：落库逻辑已考虑（reviews 表的 deleted_at 字段随 sessions 级联）
