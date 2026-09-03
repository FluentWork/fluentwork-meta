# FluentWork V2.0 周末作战地图与 Issue 草案集

**版本**：V1.0　**日期**：2026-09-03　**状态**：作战预热
**目标周末**：2026-09-05（周六）/ 2026-09-06（周日）
**作战半径**：MVP 闭环"说 → 读 → 炼化 → 闪测 → 语料库"中仍未落地的 18+ 个 Issue
**作战目标**：48 小时内尽可能多地把"决策 + 凭证 + 骨架 + 一条端到端 demo"落地

---

## 一、作战总览

### 1.1 已完工口径（基线）

```
✅ 已完成（14/15 issues, 第二波主功能面代码）:
  - 说: WSS/ASR/状态机/转录浮层/徽章/I11
  - 读: 回顾页 UI/转录展示/历史列表骨架
  - 炼化: phrase_blocks 表/批量入库/标签/编辑
  - 语料库: I9/F1/F2/F4/本地优先同步

⚠️ 仍 OPEN:
  - B8 真实 LLM 评价（评价内容仍 stub）
  - 第二波遗留: A1-A2/A4/B4/B7/C4/F3/C5 内容生成
  - W7 排期未动: E1-E5/H1-H4
  - V2.0 全新增: LLM 编排/TTS/闪测调度
```

### 1.2 作战科目（按 Epic 分组）

| Epic | 标题 | 周末可完成范围 | 优先级 |
|---|---|---|---|
| **EPIC-V2-A** | LLM 编排与生成层 | B16 AIOrchestrator 骨架 + B26 VAD 接入 + 一条 demo 通 | ⭐⭐⭐⭐⭐ |
| **EPIC-V2-B** | 真实 LLM 评价（B8 收口） | B18 接口契约 + Prompt 模板（实现需凭证） | ⭐⭐⭐⭐⭐ |
| **EPIC-V2-C** | 素材输入模块 | B21 后端 + I14 弹层 UI | ⭐⭐⭐⭐ |
| **EPIC-V2-D** | 即时反馈与 TTS 集成 | B17 TTS Provider + B19 命中注入 + I15 iOS TTS | ⭐⭐⭐⭐ |
| **EPIC-V2-E** | 闪测/话题/补完 | B22 接口契约 + B23 接口契约 + B24/C4 + B25/F3 | ⏸ W6 启动 |

### 1.3 火山云依赖清单

| 服务 | Issue | 用途 | 优先级 |
|---|---|---|---|
| **火山 Ark LLM** | B16, B18, B19, B20, B23 | 对话生成 + 评价 + 内容生成 + 话题生成 | 硬阻塞 C-1 |
| **火山 TTS** | B17 | 流式 TTS 输出 | 硬阻塞 C-2 |
| **火山对象存储 TOS** | B20 | 每日一读音频 / 录音文件 | 阻塞 C-3 |
| **火山语音 SDK（含 VAD）** | B26 | iOS/Android 端侧语音活动检测 | 可选 |
| **火山双工 ASR** | 已接入（B14） | 复用现状 | — |

> **VAD 决策建议**：iOS 端首期用**内置 webrtc VAD + 语音能量检测**双策略，端侧零网络成本；火山语音 SDK 仅在端侧检测失败/复杂场景作为兜底。B26 Issue 已含此决策。

---

## 二、Epic-V2-A：LLM 编排与生成层

### Epic 目标

建立 AIOrchestrator 抽象层，统一调度所有 LLM 调用，提供流式输出、Prompt 模板管理、成本控制、超时降级能力。**这是整条 V2.0 闭环的基础，所有 LLM 调用走此层。**

### Issue 列表

#### ⭐ B16-A: AIOrchestrator 骨架 + LLM Provider 接口（必打第一个）

**范围**：
- 创建 `internal/orchestrator/` 包（已有 `content/` 包可参考）
- 定义 `LLMProvider` interface：`Stream(ctx, prompt, opts) (<-chan Token, error)`
- 实现 `ArkProvider`（火山 Ark LLM）作为 first implementation
- 实现 `MockProvider`（回放固定 fixture）作为 deterministic test provider
- `Orchestrator` 顶层：路由选择 + 重试 + timeout + 埋点

**接口契约**（草案）：
```go
type LLMProvider interface {
    Stream(ctx context.Context, req LLMRequest) (<-chan LLMEvent, error)
}

type LLMRequest struct {
    SystemPrompt    string
    Messages        []ChatMessage
    Model           string  // "ark-doubao" | "ark-mini" | ...
    Temperature     float64
    MaxTokens       int
    ResponseFormat  string  // "text" | "json"
    StreamMetadata  map[string]any
}

type LLMEvent struct {
    Type     string  // "delta" | "tool_call" | "done" | "error"
    Content  string
    Metadata map[string]any
}
```

**火山云依赖**：Ark LLM `ep-20250620`（深度思考）+ `ep-mini`（快速）双模型，按 `req.Model` 路由。

**测试用例**：
| ID | 场景 | 输入 | 预期输出 |
|---|---|---|---|
| T1.1 | 基本流式返回 | 简单问候 prompt | 至少 3 个 delta 事件 + 1 个 done |
| T1.2 | Mock provider 一致性 | 同 prompt 三次调用 | 三次输出字节一致 |
| T1.3 | 超时降级 | provider hang 30s | ctx cancel 后 5s 内收到 error 事件 |
| T1.4 | 并发安全 | 100 goroutine 同时 Stream | 无 race（`-race` 通过） |
| T1.5 | Token 计数准确性 | 已知长度输出 | 统计 token 与调用方报告数偏差 ≤ 2% |
| T1.6 | Error propagation | provider 返回 5xx | 客户端收到 error event 含 status_code |
| T1.7 | 短路返回 | ctx 已 Done | 立即返回 ctx.Err()，无任何 IO |

**验收标准**：
- `internal/orchestrator/` 包可 import
- 单测覆盖率 ≥ 80%
- 集成测试（mock provider）端到端串联成功
- `cmd/orchestrator-demo/main.go` 跑通，打印流式 token

**估计工作量**：5 dev-days（周末可完成骨架+单测约 1.5 days）

---

#### ⭐ B16-B: Prompt 模板管理与版本控制

**范围**：
- `internal/orchestrator/prompt/` 子包
- YAML 模板文件 `prompts/*.yaml`，支持版本号
- 模板变量注入 + dry-run 输出
- 模板版本在 `ai_cost_logs` 记录

**模板清单**（V2.0 必建）：
- `prompts/dialogue-pm-mid-senior.yaml`（说的房间主对话）
- `prompts/eval-three-layer.yaml`（回顾三层评价）
- `prompts/refine-extract.yaml`（炼化话术块三元组）
- `prompts/dragon-judge.yaml`（闪测语义判定）
- `prompts/daily-read-gen.yaml`（每日一读内容生成）
- `prompts/topic-card-gen.yaml`（话题卡生成）

**测试用例**：
| ID | 场景 | 预期 |
|---|---|---|
| T2.1 | 模板加载 | YAML 解析 0 错误 |
| T2.2 | 变量注入 | `{{user_role}}` 替换为实际值 |
| T2.3 | 缺变量校验 | 必填变量缺失 → 返回 error |
| T2.4 | 版本号解析 | `prompts/eval-v3.yaml` 加载后 `prompt_version="eval-v3"` |
| T2.5 | dry-run 输出 | 控制台打印最终 prompt 但不调用 LLM |

**验收标准**：
- 6 个核心模板全部就位
- dry-run 命令可用：`./bin/orchestrator dry-run prompts/eval-three-layer.yaml --input '{"session_id":"test"}'`
- 单测覆盖率 ≥ 90%

**估计工作量**：2 dev-days

---

#### ⭐ B26: VAD 语音活动检测（端侧 webrtc + 火山 SDK 兜底）

**范围**：
- iOS 端集成 webrtc VAD（开源算法）作为第一层
- 端侧语音能量 RMS 检测作为第二层（防止 webrtc 在背景音乐漏报）
- 火山语音 SDK 仅作云端兜底，标记"上传云端判定"路径

**端侧 VAD 决策点**（V2.0 必拍）：
- 检测到 speech start：开始 PCM 缓冲上送
- 检测到 speech end：发送 `user.speech.end` 帧
- 不依赖 ASR interim 反馈

**测试用例**：
| ID | 场景 | 预期 |
|---|---|---|
| T3.1 | 静默 | 5 秒静音不触发 speech_start |
| T3.2 | 单句 | 说 "Hello" → 1 个 speech_start + 1 个 speech_end |
| T3.3 | 中间停顿 | "Hello... (1s) ...World" → 1 个 speech_start + 1 个 speech_end |
| T3.4 | 长篇独白 | 持续说话 60s | 不应误判为多个 segment |
| T3.5 | 背景音乐 | 播放轻音乐 + 说话 | 仍能检测到说话 |
| T3.6 | 多人说话 | 双声道混音 | 检测为主说话人 |

**验收标准**：
- iOS `LiveAudioEngine` 集成 `VADProcessor` 中间件
- 后端 `handler.go` 增加 `vad_segment_count` 埋点
- 真实麦克风 + 真实语音跑通 5 种 case 测试

**火山云依赖**：仅 fallback；正常路径走端侧，零网络成本

**估计工作量**：3 dev-days（端侧集成已大半，仅需补 3 路径测试 + 阈值调优）

---

### Epic-V2-A 周末可完成范围

**周六上午 + 周六下午 + 周日凌晨**：
- ✅ B16-A 完成（骨架 + 6 个单测 + Mock provider demo）
- ✅ B16-B 完成（6 个 Prompt 模板 + dry-run 命令）
- 🟡 B26 完成 60%（iOS VADProcessor 集成 + 3 个测试）

---

## 三、Epic-V2-B：真实 LLM 评价（B8 收口）

### Epic 目标

把"回顾页三层评价" + "炼化话术块三元组"从 stub 切真实 LLM 生成。这是 `65_闭环状态收口文档` 识别的**唯一阻断项**。

### Issue 列表

#### ⭐⭐ B18: review eval 真实 LLM 接入（B8 收口）

**范围**：
- 改造 `internal/reviewgen/`，集成 `orchestrator.ArkProvider`
- Prompt：`prompts/eval-three-layer.yaml`（目标达成度 + 问题清单 + 建议）
- Response format：JSON Schema
- 异步任务：从 `cmd/worker/` 拉取 eval task，调用 LLM，写回 `reviews` 表

**接口契约**（草案）：
```go
// POST /api/internal/review/enqueue
// Body: { "session_id": "..." }
// Response: { "review_id": "...", "status": "pending" }

// GET /api/reviews/{review_id}
// Response: {
//   "review_id": "...",
//   "session_id": "...",
//   "goal_attainment": 0.78,
//   "issues": [
//     { "user_utterance_id": "...", "category": "naturalness", "snippet": "...", "suggestion": "..." }
//   ],
//   "suggestions": [ "Try using 'Let's align on...' for soft disagreement." ],
//   "ai_cost_log_id": "...",
//   "created_at": "..."
// }
```

**JSON Schema 响应**（核心契约）：
```json
{
  "type": "object",
  "required": ["goal_attainment", "issues", "suggestions", "refined_blocks"],
  "properties": {
    "goal_attainment": { "type": "number", "minimum": 0, "maximum": 1 },
    "issues": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["utterance_id", "category", "snippet", "suggestion"],
        "properties": {
          "utterance_id": { "type": "string" },
          "category": { "enum": ["grammar", "naturalness", "missing_info"] },
          "snippet": { "type": "string" },
          "suggestion": { "type": "string" }
        }
      }
    },
    "suggestions": { "type": "array", "minItems": 0, "maxItems": 3 },
    "refined_blocks": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["intent_zh", "chunk_en", "anchor_user"],
        "properties": {
          "intent_zh": { "type": "string" },
          "chunk_en": { "type": "string" },
          "anchor_user": { "type": "string" },
          "scene_tag": { "type": "string" },
          "function_tag": { "type": "string" }
        }
      }
    }
  }
}
```

**火山云依赖**：Ark LLM `ep-20250620`（深度思考保证评价质量）

**测试用例**：
| ID | 场景 | 输入 | 预期 |
|---|---|---|---|
| T4.1 | 完整会话评价 | 真实 8 轮对话转录 | 返回完整三层评价 + 3-5 个话术块 |
| T4.2 | JSON Schema 合规性 | Mock provider 跑 schema 校验 | 100% 通过 |
| T4.3 | Stub 标记移除 | 搜索 `internal/reviewgen/` 中"stub" | 0 命中 |
| T4.4 | 超时兜底 | Mock provider hang 60s | 任务标 timeout，写 review row 含 `outcome=timeout` |
| T4.5 | 引用原句断言 | 抽查 5 条 issues | 每条 `snippet` 必出现在原转录中（substring 匹配） |
| T4.6 | 话术块入库 | 评价返回 4 个 refined_blocks | 自动入 `phrase_blocks` 表，`state="new"`, `next_due_at=now+24h` |
| T4.7 | 评价生成 P90 | 100 条测试样本 | P90 ≤ 15s（PRD §九红线） |
| T4.8 | iOS 联调 | 真接口走 demo script | iOS 回顾页从 stub 切真实数据 |

**验收标准**：
- `internal/reviewgen/stub.go` 文件物理删除
- 端到端：从 iOS 结束对话 → 后台任务队列 → 真实 LLM → iOS 回顾页看到真实评价
- 监控埋点：`ai_cost_logs.review.*` 全部字段写入
- PRD §九指标达成：评价生成 P90 ≤ 15s

**估计工作量**：5 dev-days（周末仅完成契约 + Prompt + 单测，真实接入需凭证）

---

#### ⭐ B19: B7 命中信号注入 LLM 上下文

**范围**：
- `internal/voicegateway/badge_emitter.go` 已有命中检测
- 新增 `hit_block_ids []string` 字段到 `llm_request.metadata`
- 在 `Orchestrator` 构造对话 prompt 时，**注入了命中话术块**作为上下文
- 注入位置：system prompt 末尾追加 `USER_RECENT_HITS` 段落

**注入模板**：
```yaml
- role: system
  content: |
    {{system_prompt}}

    ---
    USER_RECENT_HITS: 用户在最近对话中已用上以下已训练表达，请在后续话轮中
    自然简短确认（不显式提及 block_id）：
    {{range .hit_blocks}}- "{{.chunk_en}}" (intent: {{.intent_zh}})
    {{end}}
```

**火山云依赖**：复用 B16-A ArkProvider

**测试用例**：
| ID | 场景 | 输入 | 预期 |
|---|---|---|---|
| T5.1 | 命中信号注入 | 1 个命中话术块 + 用户下轮对话 | LLM 收到 system prompt 含该话术块 |
| T5.2 | 自然确认触发 | LLM 答复含 "Good catch using 'align on timeline'" | iOS 收到 AI 文本含相关确认 |
| T5.3 | 0 命中无注入 | 用户转录里没命中任何 block | LLM 收到的是原始 system prompt |
| T5.4 | 注入不爆炸 prompt | 100 个命中块 | prompt 总长不超过 8000 token |

**验收标准**：
- iOS 收到 AI 话轮后，肉眼可见"对刚才那句（你用过的）确认"
- 端到端 demo 一句话讲清楚："你的 `let's align on` 用得很地道" 类似输出出现
- 单元测试覆盖 4 种 case

**估计工作量**：2 dev-days

---

### Epic-V2-B 周末可完成范围

**周六下午 + 周日上午**：
- ✅ B18 接口契约冻结（OpenAPI yaml）
- ✅ B18 Prompt 模板就位
- 🟡 B18 单测（仅 Mock provider 部分；真实 LLM 凭证必需）
- ✅ B19 注入模板 + 单测

---

## 四、Epic-V2-C：素材输入模块

### Epic 目标

让用户能粘贴工作素材或一句话描述场景，作为 AI 后续对话/评价/每日一读/话题卡的内容源。**这是 PRD §3 设计原则 2 "用户素材驱动" 的工程落地**。

### Issue 列表

#### ⭐ B21: 素材模块后端（A1/A2）

**范围**：
- 新表 `materials`：id / user_id / raw_text / extracted_topics / extracted_terms / extracted_discussion_points / source_type / created_at / deleted_at
- POST `/api/materials` 异步提炼（提炼期间返回 `material_id` + `status=processing`）
- GET `/api/materials/{id}` 同步查询
- POST `/api/materials/{id}/refine` 启动"自动提炼话术块"

**接口契约**（草案）：
```yaml
POST /api/materials
body:
  raw_text: string  # ≤2000 字
  source_type: enum["paste", "one_liner", "preset"]
response:
  material_id: string
  status: "processing" | "ready"
  estimated_seconds: number  # PRD §A1 要求 5s 内有预期
  extracted:  # status=ready 时填充
    topics: string[]
    terms: string[]
    discussion_points: string[]

GET /api/materials/{id}
response:
  material_id: string
  status: ...
  extracted: ...

POST /api/materials/{id}/refine
# 异步启动话术块提炼，通过 B18 evaluation pipeline
response:
  refine_session_id: string
```

**火山云依赖**：B16-A ArkProvider（小模型，提取任务不需深度思考）

**测试用例**：
| ID | 场景 | 输入 | 预期 |
|---|---|---|---|
| T6.1 | 文本粘贴 | 2000 字项目周报 | 5s 内返回 `status=processing`（含 expected ETA） |
| T6.2 | 提炼完成 | 模拟 30s 后查询 | 返回 3-5 个 topics + 5-10 个 terms |
| T6.3 | 一句话模式 | "我想练习 design review 中表达不同意见" | 提炼出 1 个 scenario + 3 个相关 terms |
| T6.4 | 长度校验 | 2001 字 | 422 + 显式错误消息 |
| T6.5 | 级联软删除 | DELETE /api/materials/{id} | material row `deleted_at` 设置，关联 phrase_blocks 级联 |
| T6.6 | 提炼加载文案 | 检查 response.estimated_seconds | 必填且 ≥ 3（PRD §A1: 进度文案 + 预期时长） |

**验收标准**：
- `internal/materials/` 包建立
- 端到端 demo：iOS 弹层提交 → 后端接受 → 5s 内返回 loading ETA → 异步完成 → iOS 可查询
- 单测覆盖率 ≥ 80%

**估计工作量**：3 dev-days

---

#### ⭐ I14: 创建练习弹层（A1/A2 前端）

**范围**：
- iOS 新建 `CreatePracticeSheet.swift`（Sheet 形式）
- 三 Tab：粘贴 / 一句话 / 预置场景
- 默认 Tab = "一句话"（PRD §A2 冷启动摩擦最低路径）
- 提交后展示 loading（PRD §A1 强制要求：有预期文案 + ETA）
- 提炼完成跳转到"说的房间"

**UI 关键状态**：
```
Idle → Submitting (with ETA "约 10-15 秒") → Refining (with progress "正在提取讨论点") → Ready (auto-jump to room)
```

**测试用例**（iOS XCUITest）：
| ID | 场景 | 操作 | 预期 |
|---|---|---|---|
| T7.1 | 默认 Tab | 打开 Sheet | "一句话" Tab 默认选中 |
| T7.2 | 一句话提交 | 输入 "Practice design review" + Submit | 显示 "约 10-15 秒" loading |
| T7.3 | 粘贴模式 | 切到 "粘贴" Tab + 粘贴 1500 字 | 字数实时统计 + Submit 按钮可点 |
| T7.4 | 长度上限 | 输入 2001 字 | 字符统计变红 + Submit 禁用 |
| T7.5 | 提炼完成跳页 | 等待 ETA 倒数 | 跳转 SpeakingRoomView |
| T7.6 | 提炼失败回退 | 网络断开 | 显示错误 toast + 回 Sheet 让用户重试 |
| T7.7 | 预置场景 | 选 Daily Standup | 立即进入（绕过提炼） |

**验收标准**：
- PRD §A1 所有 7 项验收点
- iOS XCUITest 自动化通过
- 视觉走查：loading 文案"约 10-15 秒 + 正在提取讨论点"（PRD 强制）

**估计工作量**：2 dev-days

---

### Epic-V2-C 周末可完成范围

**周六全天 + 周日全天分散**：
- 🟡 B21 完成 70%（表 + 接口 + 单测，真实 LLM 提炼需凭证）
- ✅ I14 完成 80%（UI + 默认 Tab + loading，联调需 B21）

---

## 五、Epic-V2-D：即时反馈与 TTS 集成

### Epic 目标

把"AI 流式生成"送到 iOS，包括 LLM 流式文本 + TTS 流式音频，让对话链路首响 P90 ≤ 1.5s达标。

### Issue 列表

#### ⭐ B17: TTS Provider + 流式集成

**范围**：
- 新建 `internal/tts/` 包
- 实现 `TTSProvider` interface：`Stream(ctx, text, opts) (<-chan AudioChunk, error)`
- 实现 `VolcTTSProvider`（火山 TTS）
- 实现 `MockTTSProvider`（测试用，本地 fixture）

**两种 TTS 模式**：
1. **Streaming 模式**（对话用）：边收 LLM delta，边送 TTS 流式合成
2. **Batch 模式**（每日一读用）：整段文本一次产出，存 TOS

**接口契约**：
```go
type TTSProvider interface {
    Stream(ctx context.Context, req TTSRequest) (<-chan AudioEvent, error)
}

type TTSRequest struct {
    Text       string
    Voice      string  // "zh_male_en" | "en_female" | ...
    Format     string  // "opus" | "pcm"
    SampleRate int
    Speed      float64
    Mode       string  // "stream" | "batch"
}

type AudioEvent struct {
    Type    string  // "chunk" | "done" | "error"
    Audio   []byte  // opus frame
    Latency time.Duration
}
```

**火山云依赖**：火山 TTS `StreamTTS` 接口（独立服务，非双工内置 TTS）

**火山云决策备注**（必看）：
- 火山双工 API 内置 TTS，**但音色有限且不可换**——只能用作 fallback
- V2.0 主路径用**独立火山 TTS 服务**，可换音色、调整语速
- 凭证 `TTS_API_KEY` + `TTS_APP_ID`

**测试用例**：
| ID | 场景 | 输入 | 预期 |
|---|---|---|---|
| T8.1 | 流式 5s 内首字节 | "Hello world" | 500ms 内收到第一个 chunk |
| T8.2 | 长文本流式 | 500 词段落 | 持续收到 chunk，最后 done |
| T8.3 | Mock 一致性 | 同一文本 3 次 | 3 次 audio 字节完全一致 |
| T8.4 | 超时兜底 | TTS server hang 30s | ctx cancel 5s 内收到 error |
| T8.5 | Voice 不存在 | voice="not_exist" | 返回 422 含可选列表 |
| T8.6 | 批量模式 | TTSRequest.Mode="batch" | 单个 chunk 含完整音频 + 1 个 done |
| T8.7 | 并发安全 | 50 goroutine Stream | 无 race |

**验收标准**：
- `internal/tts/` 包可独立 import
- `cmd/tts-demo/main.go` 跑通，输出 .opus 文件可播放
- 单测覆盖率 ≥ 75%

**估计工作量**：3 dev-days

---

#### ⭐ I15: 说的房间 TTS 播放集成

**范围**：
- iOS `SpeakingRoomView` 接收 `ai.text.delta` 帧 + 新增 `ai.audio.chunk` 帧
- `AudioPlayer` 改造：支持 opus 流式播放 + 队列化管理
- AI 话轮气泡支持长按重播（B4 部分实现）

**UI 关键变更**：
```
[old] 收到 ai.text.complete → 播放整段
[new] 收到 ai.text.delta → 缓存文本；收到 ai.audio.chunk → 入队播放
```

**测试用例**（iOS XCUITest）：
| ID | 场景 | 操作 | 预期 |
|---|---|---|---|
| T9.1 | 流式播放 | AI 说话时 | 气泡内音频进度条实时推进 |
| T9.2 | 重叠播放 | 前一句未说完 + 后一句 | 队列化等待 |
| T9.3 | 打断 | 播放中按麦克风 | 立即停止当前播放 + 录音 |
| T9.4 | 长按重播 | AI 气泡长按 | 重播气泡音频 |
| T9.5 | 网络断了 | 收到 partial chunk 后断网 | 停止播放，不闪退 |

**验收标准**：
- 对话首响 P90 ≤ 1.5s（PRD §九红线）
- iOS AudioQueue 缓冲 200-500ms

**估计工作量**：2 dev-days

---

#### ⭐ B19 (已列): 命中信号注入 LLM 上下文

见 Epic-V2-B。

---

### Epic-V2-D 周末可完成范围

**周日全天**：
- 🟡 B17 完成 60%（Provider 骨架 + Mock 单测，真实 TTS 需凭证）
- 🟡 I15 完成 70%（流式播放骨架，真实联调需 B17）

---

## 六、Epic-V2-E：闪测/话题/补完（W6+ 范围，周末不主攻）

> 这些 Issue 周末仅做**接口契约冻结**和**骨架创建**，实现放 W6+。

### Issue 列表（接口契约级）

#### B22: 闪测模块后端（E1-E3）

- **表**：`drill_records`（id/user_id/block_id/result/expiry/duration_ms/created_at）
- **索引**：`idx_due(user_id, block_id, created_at)` for round-robin query
- **接口**：
  - `GET /api/drill/round?count=10` → 抽 10 张未到期卡
  - `POST /api/drill/judge` → LLM 语义判定
  - `PATCH /api/drill/records/{id}` → 标记完成
- **调度算法**：简化版 SM-2（V2.0 §5.3）
  - 新块：next_due_at = now + 24h
  - 失败：插入本轮尾部 + 1h retry
  - 连续 3 次成功：state="automated" + 7 天/30 天间隔

#### B23: 话题卡生成后端（H1-H2）

- **触发条件**：`phrase_blocks WHERE user_id=X AND state IN ("new","training") GROUP BY user_id HAVING count(*) >= 20`
- **生成**：调用 LLM，输入该用户 20 块的话术块 + 场景标签，输出话题卡
- **表**：`topic_cards`（id/user_id/topic/related_blocks/opener/sentences[3]/created_at)
- **接口**：
  - `GET /api/topics/weekly` → 返回本周已生成卡
  - `POST /api/topics/{id}/checkin` → 打卡（H3）

#### B24: 历史回顾接口（C4）

- `GET /api/sessions?limit=20&cursor=` → 历史会话分页

#### B25: 收藏置顶接口（F3）

- `PATCH /api/corpus/blocks/{id}/favorite` → `{ favorite: true, pinned: true }`
- `GET /api/corpus/blocks?pinned=true` → 置顶优先排序

---

## 七、GitHub Issue 申报模板

> 周末打完后，到周一统一建仓 issue。每个 issue 用以下模板。

### Template: Backend Issue

```markdown
## 标题
B16-A: AIOrchestrator 骨架 + LLM Provider 接口

## Epic
EPIC-V2-A: LLM 编排与生成层

## 优先级
P0 (V2.0 启动硬前置)

## 范围
- 创建 `internal/orchestrator/` 包
- 定义 `LLMProvider` interface
- 实现 `ArkProvider`（火山 Ark LLM）
- 实现 `MockProvider`
- 顶层 Orchestrator：路由 + 重试 + timeout + 埋点

## 依赖
- 火山 Ark LLM 凭证（C-1）
- 53_V2.0启动前待办清单 D-1 决策

## 接口契约
[链接到 docs/api/v2/orchestrator.md]

## 火山云依赖
Ark LLM `ep-20250620` + `ep-mini`，按 req.Model 路由

## 验收标准
- [ ] internal/orchestrator/ 包可 import
- [ ] 单测覆盖率 ≥ 80%
- [ ] 集成测试通过
- [ ] cmd/orchestrator-demo 跑通

## 测试用例
- [ ] T1.1 基本流式返回
- [ ] T1.2 Mock provider 一致性
- [ ] T1.3 超时降级
- [ ] T1.4 并发安全
- [ ] T1.5 Token 计数
- [ ] T1.6 Error propagation
- [ ] T1.7 短路返回

## 估计工作量
5 dev-days

## 关联文档
- docs/30_技术方案/39_FluentWork后端技术架构设计V2_0_2026-09-03.md §三
- docs/40_研发流程与协作/53_FluentWork_V2_0启动前待办清单_2026-09-03.md §B16
```

### Template: iOS Issue

```markdown
## 标题
I14: 创建练习弹层（A1/A2 前端）

## Epic
EPIC-V2-C: 素材输入模块

## 优先级
P0

## 范围
- 创建 `CreatePracticeSheet.swift`
- 三 Tab：粘贴 / 一句话 / 预置场景
- 默认 Tab = 一句话
- 提交后展示 loading（PRD §A1）

## 依赖
- B21 完成后端接口

## UI 关键状态
Idle → Submitting (with ETA) → Refining (with progress) → Ready

## 测试用例 (XCUITest)
- [ ] T7.1 默认 Tab
- [ ] T7.2 一句话提交
- [ ] T7.3 粘贴模式
- [ ] T7.4 长度上限
- [ ] T7.5 提炼完成跳页
- [ ] T7.6 提炼失败回退
- [ ] T7.7 预置场景

## 验收标准
- PRD §A1 所有 7 项
- XCUITest 自动化
- 视觉走查 loading 文案

## 估计工作量
2 dev-days

## 关联文档
- docs/20_产品设计/20_FluentWork产品需求文档PRD.md §7.1 A1/A2
```

### Template: Epic

```markdown
## 标题
EPIC-V2-A: LLM 编排与生成层

## 范围
建立 AIOrchestrator 抽象层，统一调度所有 LLM 调用

## 子 Issue
- [ ] B16-A: AIOrchestrator 骨架 + Provider 接口
- [ ] B16-B: Prompt 模板管理
- [ ] B26: VAD 语音活动检测
- [ ] I15: 说的房间 TTS 集成（依赖 B17）

## 关键依赖
- 火山 Ark LLM 凭证（C-1）
- D-1 LLM 模型选型

## 验收门槛
- 后端 orchestrator 流式接口联通
- iOS TTS 流式播放首响 P90 ≤ 1.5s

## 关联文档
- 39_V2.0 架构 §三
- 53_待办清单 §Epic-V2-A
```

---

## 八、周末作战时刻表（精确到小时）

### 周六 2026-09-05

| 时段 | 任务 | 产出 |
|---|---|---|
| **09:00-10:00** | **决策会议**：D-1 ~ D-5 拍板 | 5 个开放决策锁定 |
| **10:00-12:00** | **凭证跟进**：联系火山商务核对 Ark LLM + TTS 凭证状态 | 凭证 ETA 明确 |
| **13:00-17:00** | **B16-A 骨架**：orchestrator 包 + 接口 + Mock provider + 7 单测 | 代码 + 单测 |
| **17:00-19:00** | **B16-B 模板**：6 个 YAML 模板 + 加载器 + dry-run 命令 | 模板 + 命令 |
| **20:00-22:00** | **B26 部分**：iOS VADProcessor 集成 + 3 单测 | 代码 + 单测 |

### 周日 2026-09-06

| 时段 | 任务 | 产出 |
|---|---|---|
| **09:00-12:00** | **B17 TTS 骨架**：provider 包 + Mock 单测 | 代码 + 单测 |
| **13:00-17:00** | **B21 素材骨架**：表 + 接口 + 6 单测 | 代码 + 单测 |
| **17:00-18:00** | **I14 弹层 UI**：View + reducer + 4 个 XCUITest | 代码 + 测试 |
| **18:00-19:00** | **I15 TTS 集成骨架**：iOS AudioQueue 改造 | 代码 |
| **20:00-22:00** | **联调 + 端到端 demo**：从 iOS 弹层 → 后端 → LLM mock → 回顾页 stub | demo 录像 |
| **22:00-23:00** | **文档更新**：53_待办清单进度勾选 + 65_收口报告 + 56_收口 | commit |

### 周一 9/7 上午：建仓 issue（按本文 §七 模板）

---

## 九、风险与降级

| 风险 | 触发条件 | 降级方案 |
|---|---|---|
| 火山凭证未到位 | 周六前拿不到 Ark LLM key | 所有 LLM 调用走 Mock provider，留 TODO 切真 |
| B18 评价单测覆盖不到位 | 周末做不完 | 仅契约 + Prompt + 架构 demo；实现推到 W5 |
| I15 TTS 流式联调失败 | iOS AudioQueue 兼容性问题 | 退回整段播放，首响 P90 不达标可接受 |
| Demo 录像失败 | 任意模块崩 | 仅录制已通的 flow，保持 30s 内能讲故事 |

---

## 十、周末打完的标准（出门验收）

**硬产出（必须达成）**：
- [ ] B16-A 骨架 + 7 单测全绿
- [ ] B16-B 6 Prompt 模板就位 + dry-run 工作
- [ ] B18 接口契约冻结（OpenAPI yaml）
- [ ] B19 注入模板 + 4 单测
- [ ] B21 表 + 接口 + 6 单测
- [ ] B17 TTS Provider 骨架 + Mock 单测
- [ ] I14 弹层 UI + 4 XCUITest
- [ ] I15 TTS 集成骨架
- [ ] 至少一段端到端 demo（iOS 弹层 → mock LLM → 跳页）
- [ ] Demo 录像（30s 内）放到共享盘

**软产出（争取达成）**：
- [ ] B26 VAD 集成 60%
- [ ] B22/B23 接口契约冻结
- [ ] GitHub issue 模板在仓内建立（待周一申报）

**未达成范围（明确记账）**：
- 真实 LLM 评价（需凭证）
- 真实 TTS 端到端（需凭证）
- 闪测实现（W6+）
- 话题卡实现（W6+）

---

## 十一、对接关系

| 本文档 | 上游 | 下游 |
|---|---|---|
| 周末作战地图 | PRD V1.5 §14.3 护城河、`39_` V2.0 架构、`53_` 待办清单、`65_` 闭环收口 | GitHub Issues（周一建立）、内测验收清单 |

---

*文档版本：V1.0*
*作战时间：2026-09-05 ~ 2026-09-06*
*下次更新：周一 9/7 建仓 issue 完成后*
