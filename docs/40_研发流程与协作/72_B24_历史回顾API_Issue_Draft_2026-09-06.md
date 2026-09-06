# B24 历史回顾 API（含会话列表）— GitHub Issue 草稿

**仓库**：`FluentWork/fluentwork-backend`
**优先级**：P2（W3 中期补齐；解锁 I19 历史回顾列表）
**估时**：0.5 dev-day
**关联文档**：
- `docs/30_技术方案/39_FluentWork后端技术架构设计V2_0_2026-09-03.md` §5.3
- `docs/30_技术方案/48_FluentWork_V2_REST接口契约冻结_2026-09-06.md` §1.3.2
- `docs/40_研发流程与协作/55_FluentWork_V2_W3_代码层启动包_2026-09-06.md`
- 父 Issue：无（独立启动）

---

## 🎯 目标

实施 PRD §C4 历史回顾 API：会话列表 + 单会话详情（基础版）：
- `GET /api/v1/sessions` 列表（cursor 分页，默认 size=20，按 started_at DESC）
- `GET /api/v1/sessions/:session_id` 单会话详情（含 utterances + materials 关联）

> **简化范围**：本期不含筛选（since / until / material_id）；V1.5 加 query params

---

## 🚧 阻塞条件

- `practice_sessions` + `utterances` + `materials` 3 张表 schema 已就绪（含 soft delete 字段）
- B22 A4 软删除已 CLOSED（list API 过滤 `deleted_at IS NULL`）

---

## 📋 实施步骤

### 1. 包结构 `internal/session_history/`

```
internal/session_history/
├── service.go    # 列表 + 详情业务逻辑
├── cursor.go     # cursor 编解码（base64）
├── http.go       # RegisterRoutes
└── service_test.go
```

### 2. 数据模型（`internal/session_history/model.go`）

```go
type SessionListItem struct {
    SessionID      string    `json:"session_id"`
    MaterialIDs    []string  `json:"material_ids"`
    ReviewStatus   string    `json:"review_status"`  // "pending" | "ready" | "failed"
    StartedAt      time.Time `json:"started_at"`
    EndedAt        *time.Time `json:"ended_at,omitempty"`
    UtteranceCount int       `json:"utterance_count"`
}

type SessionListPage struct {
    Items      []SessionListItem `json:"items"`
    NextCursor *string           `json:"next_cursor,omitempty"`
}

type SessionDetail struct {
    SessionID   string                `json:"session_id"`
    UserID      string                `json:"user_id"`
    Materials   []MaterialSummary     `json:"materials"`
    Utterances  []UtteranceSummary    `json:"utterances"`
    Review      *ReviewSummary        `json:"review,omitempty"`  // 关联 B18 review
    StartedAt   time.Time             `json:"started_at"`
    EndedAt     *time.Time            `json:"ended_at,omitempty"`
    VoiceID     string                `json:"voice_id"`
}

type MaterialSummary struct {
    ID       string `json:"id"`
    Title    string `json:"title"`
    Kind     string `json:"kind"`
}

type UtteranceSummary struct {
    ID            string  `json:"id"`
    Speaker       string  `json:"speaker"`  // "user" | "ai"
    Text          string  `json:"text"`
    AudioURL      *string `json:"audio_url,omitempty"`
    StartedAtMs   int64   `json:"started_at_ms"`
    EndedAtMs     int64   `json:"ended_at_ms"`
    Eval          *UtteranceEval `json:"eval,omitempty"`  // 关联 B18 eval
}
```

### 3. Service 层（`service.go`）

```go
type Service struct {
    store Store
    logger *slog.Logger
}

func (s *Service) List(ctx context.Context, userID string, cursor *string, size int) (*SessionListPage, error) {
    if size <= 0 || size > 100 {
        size = 20
    }
    
    // 解码 cursor
    var lastStartedAt *time.Time
    var lastID *string
    if cursor != nil {
        decoded, err := decodeCursor(*cursor)
        if err != nil { return nil, apierr.BadRequest("invalid_cursor") }
        lastStartedAt = decoded.StartedAt
        lastID = decoded.ID
    }
    
    items, err := s.store.ListSessions(ctx, userID, lastStartedAt, lastID, size+1)
    if err != nil { return nil, err }
    
    page := &SessionListPage{Items: items[:min(len(items), size)]}
    if len(items) > size {
        last := page.Items[len(page.Items)-1]
        page.NextCursor = encodeCursor(Cursor{StartedAt: last.StartedAt, ID: last.SessionID})
    }
    
    return page, nil
}

func (s *Service) GetDetail(ctx context.Context, userID, sessionID string) (*SessionDetail, error) {
    // 1. 校验 session 归属 + 未删除
    sess, err := s.store.GetSession(ctx, sessionID)
    if err != nil { return nil, err }
    if sess.UserID != userID { return nil, apierr.Forbidden("not_owner") }
    if sess.DeletedAt != nil { return nil, apierr.NotFound("session_deleted") }
    
    // 2. 拉取关联 materials + utterances
    materials, err := s.store.GetSessionMaterials(ctx, sessionID)
    if err != nil { return nil, err }
    
    utterances, err := s.store.GetSessionUtterances(ctx, sessionID)
    if err != nil { return nil, err }
    
    // 3. 关联 review（如已 ready）
    review, _ := s.reviewStore.GetReview(ctx, sessionID)  // B18 关联
    
    return &SessionDetail{
        SessionID:  sessionID,
        UserID:     userID,
        Materials:  materials,
        Utterances: utterances,
        Review:     review,
        StartedAt:  sess.StartedAt,
        EndedAt:    sess.EndedAt,
        VoiceID:    sess.VoiceID,
    }, nil
}
```

### 4. Cursor 编解码（`cursor.go`）

```go
type Cursor struct {
    StartedAt time.Time `json:"s"`
    ID        string    `json:"i"`
}

func encodeCursor(c Cursor) string {
    raw, _ := json.Marshal(c)
    return base64.URLEncoding.EncodeToString(raw)
}

func decodeCursor(s string) (*Cursor, error) {
    raw, err := base64.URLEncoding.DecodeString(s)
    if err != nil { return nil, err }
    var c Cursor
    if err := json.Unmarshal(raw, &c); err != nil { return nil, err }
    return &c, nil
}
```

### 5. HTTP 路由（48 §1.3.2）

```go
// internal/session_history/http.go
func RegisterRoutes(rg gin.IRouter, h *Handler) {
    rg.GET("/sessions",         h.accounts.RequireAuth(), h.ListSessions)
    rg.GET("/sessions/:session_id", h.accounts.RequireAuth(), h.GetSessionDetail)
}
```

```go
func (h *Handler) ListSessions(c *gin.Context) {
    cursor := c.Query("cursor")
    sizeStr := c.DefaultQuery("size", "20")
    size, _ := strconv.Atoi(sizeStr)
    
    page, err := h.svc.List(c, h.accounts.UserID(c), nilIfEmpty(cursor), size)
    if err != nil { httpjson.Error(c, err); return }
    httpjson.OK(c, page)
}

func (h *Handler) GetSessionDetail(c *gin.Context) {
    sessionID := c.Param("session_id")
    detail, err := h.svc.GetDetail(c, h.accounts.UserID(c), sessionID)
    if err != nil { httpjson.Error(c, err); return }
    httpjson.OK(c, detail)
}
```

### 6. OpenAPI 同步

```yaml
  /sessions:
    get:
      summary: 会话列表（cursor 分页）
      tags: [sessions]
      parameters:
        - in: query
          name: cursor
          schema: { type: string }
          description: 下一页 cursor（首次不传）
        - in: query
          name: size
          schema: { type: integer, minimum: 1, maximum: 100, default: 20 }
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema: {$ref: '#/components/schemas/SessionListPage'}
        '401': {$ref: '#/components/responses/Unauthorized'}

  /sessions/{session_id}:
    get:
      summary: 会话详情
      tags: [sessions]
      parameters: [{$ref: '#/components/parameters/SessionID'}]
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema: {$ref: '#/components/schemas/SessionDetail'}
        '401': {$ref: '#/components/responses/Unauthorized'}
        '403': {$ref: '#/components/responses/Forbidden'}
        '404': {$ref: '#/components/responses/NotFound'}
```

---

## ✅ 测试矩阵

| ID | 场景 | 输入 | 预期 |
|---|---|---|---|
| T-B24-1 | 列出最近 20 条会话 | 用户有 50 条 session | 20 条 + next_cursor |
| T-B24-2 | 翻页 | next_cursor 续传 | 接下来 20 条 + 新 cursor |
| T-B24-3 | 最后一页不足 size | 总 25 条, 翻到第 2 页 | 5 条 + next_cursor=null |
| T-B24-4 | 空列表 | 无 session | items=[] + next_cursor=null |
| T-B24-5 | cursor 篡改 | cursor="invalid" | 400 invalid_cursor |
| T-B24-6 | size 超限 | size=200 | 强制 size=20 |
| T-B24-7 | GET /sessions/:id 完整详情 | session 10 句 | 完整字段（含 materials / utterances / review） |
| T-B24-8 | 跨用户权限 | user A GET user B session | 403 forbidden |
| T-B24-9 | A4 删除后查询 | 已删除 session | 404 session_deleted |
| T-B24-10 | A4 撤销后查询 | 撤销后 | 200 完整内容 |
| T-B24-11 | session 无 utterances | 空 session | 200 + utterances=[] |
| T-B24-12 | review 未 ready | review_status=pending | 200 + review=nil |
| T-B24-13 | review failed | review_status=failed | 200 + review={status:"failed", score:0} |
| T-B24-14 | B18 关联 | review_status=ready | 200 + review={score, dims, suggestions} |
| T-B24-15 | 性能 | 50 条 session × 10 utterance | P95 ≤ 200ms |
| T-B24-16 | cursor 稳定性 | 时间戳相同 + ID 字典序 | 不会重复 / 遗漏 |

---

## 📐 估时拆分

| 步骤 | 估时 |
|---|---|
| 1. 数据模型 + Cursor 编解码 | 0.1 dev-day |
| 2. Service 层（List + GetDetail） | 0.2 dev-day |
| 3. HTTP 路由 + OpenAPI 同步 | 0.1 dev-day |
| 4. 测试 + B18 联动 | 0.1 dev-day |
| **合计** | **0.5 dev-day** |

---

## 🎯 DoD

- [ ] `internal/session_history/` 整个包实施
- [ ] `go test ./internal/session_history/...` 16 个测试通过
- [ ] 端到端冒烟：本地真实 session 走通 list + detail
- [ ] 性能：50 句 session P95 ≤ 200ms
- [ ] OpenAPI 同步：`GET /sessions` + `GET /sessions/{id}`
- [ ] A4 软删除 + 撤销删除兼容
- [ ] B18 review 联动（review_status=ready 时嵌入 review 字段）
- [ ] PR 通过 OpenCodeReview + 1 名后端 maintainer review
- [ ] Issue description 链接本文 + 55_ 启动包 + 48 §1.3.2
- [ ] 完成后状态写到 53_ §Layer 2 表格（CLOSED ✅）

---

## 🔗 关联 Issue

- 上游：无
- 关联：B18 review eval（本期通过 `reviewStore.GetReview` 嵌入 review 字段；如 B18 未落地则 `review=nil`）
- 关联：B22 A4 软删除（list 过滤 deleted_at）
- 下游：I19 iOS 历史回顾列表（消费本 Issue 接口）
- 平行：B17 / B18 / B19 / B21 / B22 / B25 / B23（同 W3-W4 批次）

---

## 📝 备注

- **cursor 编码**：使用 base64 JSON（而非 signed cursor）—— 简化客户端实现；如需防篡改可在 V1.5 加 HMAC
- **V1.5 扩展**：支持 query params `since` / `until` / `material_id` 筛选
- **review 字段可选**：B18 未落地时 `review=nil`；客户端应处理 review_status=pending 状态
- **跨用户 403 vs 404**：本期返回 403（明示存在）；V1.5 改 404（更安全的隐私策略）
