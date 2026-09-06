# FluentWork V2.0 REST 接口契约冻结 V1.0

**版本**：V1.0　**日期**：2026-09-06　**性质**：DC-3 接口契约冻结 · 主交付物
**对应**：PRD V1.4 / 后端技术架构设计 V2.0 (`39_`) / 后端技术方案 §四 (`31_`) / B7 (`44_`) / A4 (`45_` V1.2) / F3+B4 (`46_` V1.2) / API 对账 (`47_`) / V2.0 启动前待办 (`53_` DC-3)
**定位**：FluentWork **V1.4 + V2.0** 的完整 REST 接口契约冻结版——以本文为准落地代码、OpenAPI、iOS stub、测试用例。
**变更记录**：
- V1.0 (2026-09-06): 首版冻结；整合 `31_/44_/45_/46_/47_` 全部已拍板契约

> **使用规则**：本文是「写代码时」的事实源。任何 PR 修改 `internal/{*}/http*.go` 路由表，必须同步更新本文相应行（PR checklist 第一条）。**本文与 `47_` 对账文档职责互补**：`47_` 负责对账 + 决策记录；本文负责契约冻结本身。

---

## 〇、契约通用约定

### 0.1 路径前缀

| 类别 | 前缀 | 鉴权 |
|---|---|---|
| 外部 API（用户/客户端可见） | `/api/v1/` | `Authorization: Bearer <jwt>`（游客 `guest_id`、注册 `account_id`） |
| 内部 API（服务间 + ops 工具） | `/internal/v1/` | `X-Internal-Token: <internal_token>` |
| 健康/就绪/发现 | `/healthz` `/readyz` `/` `/openapi.yaml` | 无 |

### 0.2 通用响应

**成功**：
```json
{ "data": <payload>, "request_id": "<uuid>" }
```

**错误**：
```json
{ "code": "<error_code>", "message": "<human_readable>", "request_id": "<uuid>" }
```

### 0.3 通用错误码

| Code | HTTP | 含义 |
|---|---|---|
| `bad_request` | 400 | 请求体格式错误 |
| `unauthorized` | 401 | Token 缺失 / 无效 |
| `forbidden` | 403 | 资源不属于当前用户 |
| `not_found` | 404 | 资源不存在 |
| `conflict` | 409 | 状态冲突（如对已结束会话发消息） |
| `invalid_confirmation_code` | 422 | 删除确认码错误 |
| `rate_limited` | 429 | 限流 |
| `internal` | 500 | 服务端内部错误 |
| `unavailable` | 503 | 依赖服务不可用（DB / 火山 API） |

### 0.4 分页（统一 cursor 制）

```yaml
Query:
  cursor: string        # 上次响应 next_cursor；首次不传
  limit: int            # 默认 20，最大 100
Response:
  items: [...]
  next_cursor: string | null
```

---

## 一、外部 API（用户可见）

### 1.1 账号（account）

#### 1.1.1 发送邮箱验证码
```yaml
POST /api/v1/auth/email-code
Headers: Authorization: Bearer <jwt>
Body: { email: string }
Response 200: { sent_at: timestamp }
Response 429: rate_limited
限流: 同邮箱 1/min
```

#### 1.1.2 发送手机短信验证码
```yaml
POST /api/v1/auth/sms-code
Headers: Authorization: Bearer <jwt>
Body: { phone: string, country_code: string (default "+86") }
Response 200: { sent_at: timestamp }
Response 429: rate_limited
限流: 同号码 1/min
备注: 国内主通道（V1.3 新增）
```

#### 1.1.3 验证码登录
```yaml
POST /api/v1/auth/login
Body: { email?: string, phone?: string, country_code?: string, code: string }
Response 200:
  access_token: string      # 15min
  refresh_token: string     # 30d
  account_id: string
  is_new_account: bool      # true → 首次注册
```

#### 1.1.4 游客登录
```yaml
POST /api/v1/auth/guest
Body: { device_id: string }   # iOS 持久化 UUID
Response 200:
  access_token: string
  refresh_token: string
  guest_id: string
  expires_at: timestamp        # 30d 游客期
备注: V1.4 新增；演示主路径；device_id 绑定
```

#### 1.1.5 游客→注册合并
```yaml
POST /api/v1/account/merge
Headers: Authorization: Bearer <jwt (注册账号)>
Body: { guest_id: string }
Response 200:
  account_id: string
  cascaded:
    materials: int
    sessions: int
    phrase_blocks: int
    drills: int
幂等: 同 guest_id 二次调用返回 cascaded: all_zeros
备注: V1.4 新增；幂等以 device_id 去重
```

#### 1.1.6 一键删除全部数据
```yaml
DELETE /api/v1/account/data
Headers: Authorization: Bearer <jwt>
Body: { confirmation_code: "DELETE-MY-DATA" }
Response 200:
  deleted_at: timestamp
  cascaded:
    materials: int
    sessions: int
    utterances: int
    phrase_blocks: int      # 含 automated/训练中/新
    drill_records: int      # 物理删除
    reviews: int
    topic_cards: int
    daily_reads: int
    block_uses: int
  backup_purge_at: timestamp    # 30d 后物理删除
Response 422: invalid_confirmation_code
备注: ✅ D-API-2 拍板；30 天撤销窗口
```

#### 1.1.7 数据导出
```yaml
POST /api/v1/account/export
Headers: Authorization: Bearer <jwt>
Response 200:
  export_id: string
  email_to: string           # 7 天内收到
  estimated_ready_at: timestamp
备注: 异步任务；邮件含全量 phrase_blocks + materials + sessions JSON
```

### 1.2 素材（materials）

#### 1.2.1 创建素材
```yaml
POST /api/v1/materials
Headers: Authorization: Bearer <jwt>
Body:
  kind: enum["text", "voice_note", "url"]
  content: string           # 文本 / URL / ASR 转录结果
  source_meta?: object      # URL 时填 {title, excerpt}
Response 202:
  material_id: string
  refine_status: enum["queued", "processing"]
备注: 触发异步提炼任务（提炼后状态转 ready）；PRD A1
```

#### 1.2.2 轮询提炼结果
```yaml
GET /api/v1/materials/:id
Headers: Authorization: Bearer <jwt>
Response 200:
  material_id: string
  kind: enum["text", "voice_note", "url"]
  content: string
  refine_status: enum["queued", "processing", "ready", "failed"]
  refine_json?: object      # ready 时返回 {scene_tag, intent_zh, phrases, suggested_actors}
  created_at: timestamp
  updated_at: timestamp
```

#### 1.2.3 删除素材
```yaml
DELETE /api/v1/materials/:id
Headers: Authorization: Bearer <jwt>
Response 204
备注: PRD A4/F4；级联策略同账号级删除——衍生话术块可选级联删除或锚点脱敏保留
```

### 1.3 会话（sessions）

#### 1.3.1 创建会话
```yaml
POST /api/v1/sessions
Headers: Authorization: Bearer <jwt>
Body:
  material_ids: string[]          # 1+ 个素材
  actor_id?: string               # 可选：指定 AI 角色
  voice_id?: string               # 可选：指定 TTS 音色（D-2 拍板音色表）
Response 200:
  session_id: string
  wss_url: string
  ticket: string                  # 60s 有效
  ticket_expires_at: timestamp
备注: 客户端持 ticket 连 voice-gateway；API Key 与火山凭证全程不出服务端
```

#### 1.3.2 历史会话列表
```yaml
GET /api/v1/sessions
Headers: Authorization: Bearer <jwt>
Query:
  cursor?: string
  limit?: int (default 20)
  since?: timestamp
Response 200:
  items:
    - session_id: string
      material_ids: string[]
      review_status: enum["pending", "ready", "failed"]
      started_at: timestamp
      ended_at?: timestamp
  next_cursor: string | null
备注: C4 工作台历史入口
```

#### 1.3.3 回顾数据
```yaml
GET /api/v1/sessions/:id/review
Headers: Authorization: Bearer <jwt>
Response 200:
  session_id: string
  review_status: enum["pending", "ready", "failed"]
  utterances: [{
    utterance_id: string
    speaker: enum["user", "ai"]
    text: string
    audio_url?: string             # B4 重播（见 1.3.7）
    started_at_ms: int
    ended_at_ms: int
  }]
  eval?: object                    # {score, dims, suggestions} (B8 真实评价)
  refine_summary?: string
  started_at: timestamp
  ended_at?: timestamp
```

#### 1.3.4 放弃会话
```yaml
POST /api/v1/sessions/:id/abandon
Headers: Authorization: Bearer <jwt>
Body: { reason?: string }
Response 204
备注: B6 用户主动放弃
```

#### 1.3.5 文本降级消息
```yaml
POST /api/v1/sessions/:id/messages
Headers: Authorization: Bearer <jwt>
Body: { text: string }
Response 200:
  utterance_id: string
  ai_text: string
  created_at_ms: int
Response 409: conflict              # 语音可用时禁止文本降级
备注: V1.4 新增；仅网关判定降级后开放；非流式 ≤3s
```

#### 1.3.6 WSS 语音流（voice-gateway 直连，不走本契约）
- 见 `38_` §六 自有协议（Opus 音频帧 + JSON 控制帧）
- B14 Client/ASR/Relay 架构（37_）
- WSS 帧协议 V2.0 冻结版待 DC-3 续作（53_ ⏸）

#### 1.3.7 单条 AI 音频（重播，B4）
```yaml
GET /api/v1/sessions/:session_id/utterances/:utterance_id/audio
Headers:
  Authorization: Bearer <jwt>
  Range: bytes=0-                  # HTTP Range
Response 200:
  Content-Type: audio/opus
  Content-Length: <bytes>
  Accept-Ranges: bytes
Response 206: range 请求成功
备注: ✅ 46 B4.1 拍板；流式 Opus；HTTP Range 断点续传
```

### 1.4 语料库（corpus）

#### 1.4.1 列表
```yaml
GET /api/v1/corpus/blocks
Headers: Authorization: Bearer <jwt>
Query:
  cursor?: string
  limit?: int (default 20)
  pinned?: bool                    # ✅ 46 F3.1 (D-API-1 拍板)
  favorite?: bool                  # ✅ 46 F3.1
  scene_tag?: string
  function_tag?: string
  state?: enum["new", "training", "automated"]
  q?: string                       # 关键词搜索
Response 200:
  items: [Block]                   # 含 pinned / favorited / pinned_at / favorited_at
  next_cursor: string | null
  total: int
排序规则: pinned=true 优先 > favorited=true 其次 > updated_at DESC（服务端强制）
```

#### 1.4.2 编辑（V1.1 启用）
```yaml
PUT /api/v1/corpus/blocks/:id
Headers: Authorization: Bearer <jwt>
Body: { chunk_en?: string, intent_zh?: string, scene_tag?: string, function_tag?: string }
Response 200: { block: Block }
备注: D2
```

#### 1.4.3 置顶 / 取消置顶（✅ D-API-1 拍板方案 A）
```yaml
PATCH /api/v1/corpus/blocks/:id/pin
Headers: Authorization: Bearer <jwt>
Body: { pinned: bool }
Response 200:
  block_id: string
  pinned: bool
  pinned_at: timestamp | null
```

#### 1.4.4 收藏 / 取消收藏（✅ D-API-1 拍板方案 A）
```yaml
PATCH /api/v1/corpus/blocks/:id/favorite
Headers: Authorization: Bearer <jwt>
Body: { favorite: bool }
Response 200:
  block_id: string
  favorite: bool
  favorited_at: timestamp | null
```

#### 1.4.5 删除（V1.2 启用）
```yaml
DELETE /api/v1/corpus/blocks/:id
Headers: Authorization: Bearer <jwt>
Response 204
```

#### 1.4.6 一键全部入库（炼化卡）
```yaml
POST /api/v1/corpus/blocks/batch-accept
Headers: Authorization: Bearer <jwt>
Body: { block_ids: string[] }
Response 200:
  accepted: int
  skipped: int
备注: B10 I8；幂等
```

#### 1.4.7 (Deprecated V1.4) 旧 POST 收藏/置顶 toggle
```yaml
POST /api/v1/corpus/blocks/:id/favorite
Headers: Authorization: Bearer <jwt>
Body: { is_favorite?: bool, is_pinned?: bool }
Response 200: { block_id, is_favorite, is_pinned }
备注: ⚠️ 已在 live 代码；D-API-1 拍板后保留为 deprecated 1 个版本；V1.5 移除
```

### 1.5 闪测（drill）

#### 1.5.1 取本轮题目
```yaml
GET /api/v1/drill/round
Headers: Authorization: Bearer <jwt>
Query:
  size?: int (default 10, max 20)
Response 200:
  round_id: string
  items: [{
    block_id: string
    chunk_en: string
    intent_zh: string
    audio_url: string             # TTS 预生成
    timeout_ms: int               # 默认 8000
  }]
  started_at: timestamp
备注: 调度引擎取数（简化 SM-2，D-4 拍板）
```

#### 1.5.2 提交作答
```yaml
POST /api/v1/drill/judge
Headers: Authorization: Bearer <jwt>
Body:
  round_id: string
  block_id: string
  audio: binary                   # Opus，用户录音
  asr_text?: string               # 客户端 ASR 结果（兜底）
Response 200:
  block_id: string
  semantic_match: bool
  semantic_score: float           # 0-1
  pronunciation_score: float      # 0-100 (V1.1 实施)
  next_due_at: timestamp          # 调度更新
  state_transition: enum["new", "training", "automated"]
备注: 模块化 ASR 链路；语义判定 Ark Mini（D-3 拍板）
```

### 1.6 每日一读（daily-reads）

#### 1.6.1 今日文章
```yaml
GET /api/v1/daily-reads/today
Headers: Authorization: Bearer <jwt>
Response 200:
  read_id: string
  title: string
  content_en: string
  content_zh: string              # 翻译
  audio_url: string               # TTS（音色 D-2 表 en_male_narrator 0.85x）
  estimated_minutes: int
  generated_at: timestamp
  is_fallback: bool               # true → 按需生成（D-5 拍板）
Response 202:
  status: "generating"
  estimated_ready_in_ms: int       # 8000-12000
备注: 不存在则同步触发生成（D-5 批处理 04:00 UTC + 按需兜底）
```

#### 1.6.2 跟读评分提交
```yaml
POST /api/v1/daily-reads/:id/follow-read
Headers: Authorization: Bearer <jwt>
Body:
  sentence_index: int             # 0-based
  audio: binary                   # Opus
Response 200:
  sentence_index: int
  pronunciation_score: float      # 0-100
  fluency_score: float            # 0-100
  suggestions: string[]
备注: I2
```

### 1.7 话题卡（topic-cards）

#### 1.7.1 当前有效话题卡
```yaml
GET /api/v1/topic-cards
Headers: Authorization: Bearer <jwt>
Query:
  cursor?: string
  active_only?: bool (default true)
Response 200:
  items: [{
    card_id: string
    title: string
    prompt: string
    scene_tag: string
    function_tag: string
    valid_until: timestamp
  }]
  next_cursor: string | null
```

#### 1.7.2 聊后打卡
```yaml
POST /api/v1/topic-cards/:id/checkin
Headers: Authorization: Bearer <jwt>
Body:
  session_id?: string             # 关联练习会话
  reflection?: string             # 用户复盘
Response 200:
  checkin_id: string
  streak_days: int
备注: H3
```

### 1.8 订阅（V1.1 启用）

#### 1.8.1 StoreKit 收据校验
```yaml
POST /api/v1/billing/verify
Headers: Authorization: Bearer <jwt>
Body: { receipt: string, product_id: string }
Response 200:
  entitlement: enum["free", "monthly", "yearly"]
  expires_at?: timestamp
  transaction_id: string
```

#### 1.8.2 订阅状态
```yaml
GET /api/v1/billing/status
Headers: Authorization: Bearer <jwt>
Response 200:
  entitlement: enum["free", "monthly", "yearly"]
  expires_at?: timestamp
  features: { [feature]: bool }
备注: MVP 期恒返回 free；接口先行预留
```

---

## 二、内部 API（服务间 + ops）

### 2.1 语料库（gateway 拉取候选）
```yaml
GET /internal/v1/corpus/blocks
Headers: X-Internal-Token: <internal_token>
Query:
  user_id: string
  limit?: int (default 50, max 100)
Response 200:
  items: [Block]
调用方: voicegateway（命中检测候选源）
```

### 2.2 会话票据消费
```yaml
POST /internal/v1/tickets/consume
Headers: X-Internal-Token: <internal_token>
Body: { ticket: string, session_id: string }
Response 200:
  user_id: string
  guest_id?: string
  session_id: string
  consumed_at: timestamp
Response 410: ticket_expired
调用方: voicegateway（WSS 握手）
```

### 2.3 会话激活
```yaml
POST /internal/v1/sessions/activate
Headers: X-Internal-Token: <internal_token>
Body: { session_id: string, wss_connection_id: string }
Response 200:
  session_id: string
  activated_at: timestamp
调用方: voicegateway（WSS 握手成功后）
```

### 2.4 会话结束
```yaml
POST /internal/v1/sessions/end
Headers: X-Internal-Token: <internal_token>
Body:
  session_id: string
  ended_reason: enum["user_abandoned", "review_ready", "timeout"]
Response 200:
  session_id: string
  ended_at: timestamp
调用方: voicegateway
```

### 2.5 AI 成本日志查询（ops + 冒烟）
```yaml
GET /internal/v1/ai-cost-logs
Headers: X-Internal-Token: <internal_token>
Query:
  user_id?: string
  session_id?: string
  since?: timestamp
  cursor?: string
  limit?: int (default 50)
Response 200:
  items: [{
    log_id: string
    user_id: string
    session_id?: string
    provider: enum["ark_mini", "ark_deep", "ark_tts", "volc_duplex"]
    tokens_in: int
    tokens_out: int
    cost_usd: float
    created_at: timestamp
  }]
  next_cursor: string | null
调用方: ops dashboard, smoke harness（B16 已落地）
```

### 2.6 AI 命中信号转发（B7，V2.0）
```yaml
POST /internal/v1/voicegateway/hits
Headers:
  X-Internal-Token: <internal_token>
  X-Voicegateway-Service: voicegateway-1
Body:
  user_id: string
  session_id: string
  turn_id: string
  hits: [{ block_id: string, turn_id: string, detected_at_ms: int }]
Response 200: { recorded_count: int }
调用方: voicegateway（44 §2.1）
```

### 2.7 AI 命中聚合查询（B7，V2.0）
```yaml
GET /internal/v1/sessions/:session_id/recent-hits
Headers: X-Internal-Token: <internal_token>
Query:
  lookback_turns?: int (default 8)
  min_score?: float (default 0.7)
Response 200:
  hits: [{ block_id, intent_zh, chunk_en, turn_id, detected_at_ms }]
  ttl_at_ms: int                  # = 最新 hit detected_at_ms + 60000
调用方: voicegateway（44 §2.2）
```

### 2.8 support 撤销删除（A4，V1.4）
```yaml
POST /internal/v1/support/undelete-user
Headers: X-Internal-Token: <internal_token>
Body: { user_id: string, reason: string }
Response 200:
  user_id: string
  restored_counts: { materials: int, sessions: int, ... }
  restored_at: timestamp
调用方: support 内部工具（45 §2.3）；30 天撤销窗口内有效
```

---

## 三、系统路径

| 方法 | 路径 | 用途 |
|---|---|---|
| GET | `/healthz` | 存活探针（liveness） |
| GET | `/readyz` | 就绪探针（readiness，含 DB ping） |
| GET | `/` | 服务发现 JSON（含 `api_prefix: /api/v1`） |
| GET | `/openapi.yaml` | OpenAPI 文档（与 `/openapi/v1.yaml` 同源） |

---

## 四、本文档与上下游文档关系

```
PRD V1.4 (功能契约源头)
    ↓
30_技术方案设计 (V3.4) ← 架构总纲
    ↓
31_后端技术方案 (§四 REST 表) ← V1.4 端点列
    ↓
44/45/46 第二波规格 (B7/A4/F3+B4) ← V1.4 + V2.0 增量
    ↓
47_API契约冻结与对账 ← 对账层（决策记录）
    ↓
48_V2_REST接口契约冻结 (本文) ← 冻结层（代码事实源）
    ↓
OpenAPI yaml (api/openapi-v1.yaml) ← 代码镜像
```

**优先级**：本文 > 47_ > 44/45/46 > 31 §四 > PRD

---

## 五、状态矩阵（截至 2026-09-06）

| 端点 | 状态 | 落地 Issue |
|---|---|---|
| §1.1.1-1.1.5 | Live (B17 实施账号登录流后全 live) | B17 |
| §1.1.6 一键删除 | Planned-V1.4 | B22 |
| §1.1.7 数据导出 | Planned-V1.4 | B22 |
| §1.2.1-1.2.3 素材 | Planned-V1.4 | B21 |
| §1.3.1-1.3.4 会话基础 | Live + B24 | B24 |
| §1.3.5 文本降级 | Live（已挂路由） | — |
| §1.3.7 重播音频 | Planned-V1.4 | B25 |
| §1.4.1-1.4.2 语料库基础 | Live | — |
| §1.4.3-1.4.4 置顶/收藏 PATCH | Planned-V1.4（D-API-1 拍板） | B25 |
| §1.4.5 删除 | Live | — |
| §1.4.6 批量入库 | Live | — |
| §1.4.7 旧 POST toggle | Live（deprecated V1.4） | — |
| §1.5.1-1.5.2 闪测 | Planned-V1.4 | B22 |
| §1.6.1-1.6.2 每日一读 | Live（接口）+ B20（生成） | B20 |
| §1.7.1-1.7.2 话题卡 | Planned-V1.4 | B23 |
| §1.8.1-1.8.2 订阅 | Live（接口）+ V1.1 实数据 | — |
| §2.1-2.5 内部基础 | Live | — |
| §2.6-2.7 B7 注入 | Planned-V2.0 | B19 |
| §2.8 support 撤销 | Planned-V1.4 | B22 |

> **总览**：19 个外部端点（11 live / 8 planned）+ 8 个内部端点（5 live / 3 planned）。代码层 W3 启动后按 Issue 落地。
