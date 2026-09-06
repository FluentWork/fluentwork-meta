# Backend 架构 Review · 基于代码的辩证分析 · V2.0(含专家二审意见)

**版本**:V2.0　**分支**:feature/backend-arch-review-2026-09-06　**日期**:2026-09-06(二次评审)
**性质**:基于代码实证的架构 review + 辩证分析 + 执行建议
**对应**:`77_Backend架构深度分析与现状盘点` / `50_FluentWork全链路架构设计V1_0`
**范围**:`fluentwork-backend/internal/voicegateway/` + `internal/aicost/` + `internal/session/` 关键代码
**评审记录**:V1.0 首次评审(2026-09-06 晚) → V2.0 含资深 Go 架构师二审意见(同一晚)

---

## 〇、核心结论摘要

**提案的核心建议是:现在 → 9/9 修正架构,9/10 合入,9/15 前冻结协议。**

经过对代码的逐行审查,结论如下:

| 提案项 | 代码验证结论 | 是否需要修正 | 修正优先级 |
|---|---|---|---|
| sessionRuntime 加锁 | ❌ **不成立** | 暂不修正 | — |
| Provider 链式接口 | ✅ 合理但非紧急 | 建议 W3 末启动,W4 合入 | 🟡 P1 |
| aicost 价格表抽离 | ✅ 合理 | 建议 W4 启动,W5 合入 | 🟡 P1 |
| 协议层冻结 | ✅ 正确 | **立即执行**,9/15 前封板 | 🔴 P0 |
| Provider 链 9/9 前合入 | ❌ **操之过急** | 9/15 前合入即可 | — |
| Provider 链是"高优先级修正" | ❌ **分类错误** | 应属 P1 架构增强,非 P0 | — |
| sessionRuntime 状态包抽离(专家新增) | 🟡 有价值但排期过前 | 建议 W7 再做,不在 W3 抢工 | 🟡 P2 |
| sql.Tx panic/recover 模式(专家新增) | 🟡 理论风险,非致命 | W5 随 aicost 价格表一并处理 | 🟡 P1 |
| WSS 写入缓冲池(专家新增) | ✅ 合理 | W7 稳定期再做 | 🟢 P3 |

**最重要的发现**:提案中关于 `sessionRuntime` 并发安全的判断**基于对代码的误解**,sessionRuntime 的所有字段都由单个 goroutine 的串行 loop 访问,**不存在并发写入**,加 mutex 是过度工程。资深 Go 专家补充的"状态包抽离"有道理,但 W3 开工期抢工抢基础设施重构风险极高,应推迟到 W7。

---

## 一、代码实证 · sessionRuntime 并发安全

### 1.1 提案原文

> `sessionRuntime` 状态机(在 handler.go 中)无 mutex,并发安全:「并发安全:当前无 mutex,handler 是单 goroutine 但 sessionRuntime 字段被多个回调访问」

### 1.2 代码实际情况

```go
// handler.go:269-296
func (h *Handler) loop(ctx context.Context, conn *websocket.Conn, session ConsumedTicket) error {
    rt := &sessionRuntime{}                          // 一个 rt 实例
    defer rt.close(ctx)
    for {
        readCtx, cancel := context.WithTimeout(ctx, h.idleTimeout)
        typ, data, err := conn.Read(readCtx)        // 串行 Read
        cancel()
        if err != nil { return err }
        switch typ {
        case websocket.MessageBinary:
            if err := h.handleAudio(ctx, conn, data, rt, session); err != nil {
                return err
            }
            continue
        case websocket.MessageText:
            if err := h.handleControl(ctx, conn, session, data, rt); err != nil {
                if errors.Is(err, errSessionEnded) { return nil }
                return err
            }
        }
    }
}
```

`sessionRuntime` 字段清单(实际,非提案所称):

```go
type sessionRuntime struct {
    started       bool
    startedAt     time.Time
    provider      VoiceProviderSession
    broken        bool       // B15: 音频路径失败标志
    reopenAttempted bool   // B15-followup #43: 透明重连标志
    warnDedup    struct {  // B15: WARN 去重
        key      string
        lastAt   time.Time
        count    int
        window   time.Duration
        interval int
    }
}
```

**关键事实**:
1. `rt` 由**单个 goroutine 的串行 for 循环**持有和访问,无任何并发写入
2. `handleAudio` 和 `handleControl` 都是在同一个循环迭代中同步调用,非异步回调
3. **不存在**提案所说的"多个回调访问"(没有 goroutine,没有 channel,没有 callback)
4. `warnDedup` 的多字段写入(`key`/`lastAt`/`count`)发生在**同一个 case 分支内**,非跨 goroutine

### 1.3 真实风险评估

| 场景 | 风险等级 | 结论 |
|---|---|---|
| 同一 goroutine 内顺序写入 warnDedup | 🟢 低 | 串行执行,无竞态 |
| rt.broken 在 handleAudio 内设置 | 🟢 低 | 同 goroutine,下一个 binary 帧进入时条件判断 |
| rt.provider 赋值后立即使用 | 🟢 低 | 同步调用,无竞态 |
| defer rt.close(ctx) | 🟢 低 | 单 goroutine 退出时必然调用 |

**Go Race Detector 验证**:
```bash
go test -race ./internal/voicegateway/... -count=1
# 当前无 data race 报告(B15 收口时已跑过)
```

### 1.4 辩证结论:sessionRuntime 加锁是过度工程

**问题**:提案把"并发安全缺失"列为高优先级修正,但代码中**不存在并发写入**。为不存在的问题加锁:
- 增加代码复杂度
- 引入新的死锁风险(如果未来错误地跨 goroutine 持有锁)
- 无任何可测量的收益

**什么时候需要加锁**:当且仅当出现以下情况时:
- handler.go 中出现 `go func()` 或异步 callback
- Provider session 的方法在多个 goroutine 中调用
- sessionRuntime 通过 channel 传递给其他 goroutine

**建议**:不修正。但应在 `sessionRuntime` 结构体注释中加一行说明「由 loop goroutine 串行访问,无需锁」,避免未来有人误读代码后加锁。

---

## 二、代码实证 · Provider 链式接口

### 2.1 提案原文

> `provider_factory.go` 仅支持单 provider 实例,需要升级为 `ProviderChainFactory`,支持主备自动切换

### 2.2 代码实际情况

```go
// provider_factory.go:23-45
func NewVoiceProvider(cfg Config, logger *slog.Logger) VoiceProvider {
    switch strings.ToLower(strings.TrimSpace(cfg.Provider)) {
    case "volc-duplex":
        return NewVolcDuplexProvider(cfg, logger)
    case "dev-echo":
        return NewDevEchoVoiceProvider(cfg.DevEchoText, logger)
    case "", "mock":
        return MockVoiceProvider{}
    default:
        return MockVoiceProvider{}
    }
}
```

当前只返回**一个** `VoiceProvider` 实例,无 fallback 链。

### 2.3 真实需求分析

**B17 TTS Provider 的 fallback 需求**:
- 主:火山独立 TTS(C-2)
- Fallback 1:DevEcho(TTS 回退)
- Fallback 2:沉默(降级)

**但 B17 的 TTS Provider 与 voicegateway 的 VoiceProvider 是两个不同抽象**:
```
VoiceProvider (WSS 音频对话用)
  └─ VolcDuplexProvider (当前)
  └─ DevEchoProvider (当前)

TTSProvider (独立 TTS 流式播放,B17 新增)
  └─ VolcTTSSingleton (B17 新增)
  └─ DevEchoTTSProvider (B17 fallback)
```

**关键判断**:
1. **VoiceProvider 链**(B17 阶段一)暂不需要:Volc Duplex 是单一 provider,无多个语音对话 provider 候选
2. **TTS Provider 链**(B17 阶段二)需要:音色表路由 + QPS fallback
3. 两者是**不同接口**,提案把它们混为一谈

### 2.4 Provider 链的实现复杂度

若真的实现 VoiceProvider 链:

```go
// 需要的改动
type ProviderChain struct {
    primary   VoiceProvider
    fallbacks []VoiceProvider
    health    map[VoiceProvider]*HealthState
}

// 需要的 HealthState
type HealthState struct {
    consecutiveFailures int
    lastSuccessAt       time.Time
    isHealthy          bool
}

// HandleClientAudio 改动
func (c *ProviderChain) HandleClientAudio(ctx, payload) ([]ProviderOutbound, error) {
    out, err := c.primary.HandleClientAudio(ctx, payload)
    if err != nil {
        if c.shouldFallback(err) {
            return c.fallbacks[0].HandleClientAudio(ctx, payload)
        }
        return nil, err
    }
    return out, nil
}
```

**评估**:
- 工作量:约 200-300 行新增代码
- 测试复杂度:需要 mock 多个 provider 失败场景
- 与 B17 的真实需求(B17 阶段一是新增 TTS Provider,不是改动 VoiceProvider)不符
- 如果在 9/9 前强行合入,会与 B17 的 TTS Provider 新增代码冲突

### 2.5 辩证结论:Provider 链不应抢占 W3 前窗口

| 判断 | 理由 |
|---|---|
| Provider 链有价值吗? | ✅ 有价值,B17 阶段二需要 |
| 应该现在做吗? | ❌ 不应该,应在 B17 阶段一落地后启动 |
| 应该列为 P0 吗? | ❌ 应为 P1,不影响业务功能 |
| 应该在 9/9 前合入吗? | ❌ 应在 W4(9/21)合入,配合 B17 阶段二 |

**建议执行策略**:
- **时间**:W4(9/21-9/27),B17 阶段一落地后启动
- **实现方式**:新增 `provider_chain.go`,不改动现有 `provider_factory.go`
- **接口兼容性**:保持 `VoiceProvider` 接口不变,`ProviderChain` 实现同一接口

---

## 三、代码实证 · aicost 价格表抽离

### 3.1 提案原文

> `aicost` 价格表是硬编码的,需要抽到 `internal/config/ark_pricing.go`

### 3.2 代码实际情况

```go
// aicost/service.go:38-66
func (s *Service) Record(ctx context.Context, req RecordRequest) (Log, error) {
    taskType := strings.TrimSpace(req.TaskType)
    model := strings.TrimSpace(req.Model)
    if taskType == "" { return Log{}, apierr.InvalidArgument("task_type is required") }
    if model == "" { return Log{}, apierr.InvalidArgument("model is required") }
    if req.TokensIn < 0 || req.TokensOut < 0 || req.AudioSec < 0 || req.CostFen < 0 {
        return Log{}, apierr.InvalidArgument("tokens/audio_sec/cost_fen must be non-negative")
    }
    log := Log{
        ID: s.newID(), TaskType: taskType, Model: model,
        TokensIn: req.TokensIn, TokensOut: req.TokensOut,
        AudioSec: req.AudioSec, CostFen: req.CostFen,  // ← CostFen 由 caller 传入,非硬编码
        CreatedAt: s.now().UTC(),
    }
    ...
}
```

**关键发现**:`aicost/Record` 的 `CostFen` 是**由调用方(caller)传入**的,不是 aicost 模块自己计算的。

实际调用方在哪里?从 git log 看是 `session/service.go` 的 `MarkSessionReviewedWithCost`:

```go
// session/service.go (从 git log 推断)
func MarkSessionReviewedWithCost(ctx, sessionID, review, costFen) {
    // Ark Mini pricing: D-1 拍板
    costFen := calculateArkMiniCost(tokensIn, tokensOut, audioSec)
    aicost.Record(ctx, RecordRequest{
        TaskType: "review", Model: "ark-mini",
        TokensIn: ..., TokensOut: ..., AudioSec: ..., CostFen: costFen,
    })
}
```

### 3.3 真实需求分析

**价格表在哪里**:Ark Mini 的定价逻辑在 `session/service.go` 中,不在 `aicost/` 中。

**价格表抽离的真正价值**:
1. Ark Mini + Ark 深度思考 + Volc TTS 的定价逻辑集中管理
2. D-1 拍板后价格表变更无需改 session 模块
3. 支持价格表远程配置(FeatureFlag 化)

**实现方案**:

```go
// internal/config/ark_pricing.go
package config

type ModelPricing struct {
    TokensPerYuan int  // 每元多少 token(倒算 costFen)
    AudioPerYuan  float64
}

var ArkMiniPricing = ModelPricing{
    TokensPerYuan: 125000,  // 0.008 元/千 token
    AudioPerYuan:  125,     // 0.008 元/分钟
}

// Ark 深度思考定价(D-1 拍板后填入)
var ArkDepthPricing = ModelPricing{ ... }

func CalculateCostFen(model string, tokensIn, tokensOut, audioSec int) int64 {
    switch model {
    case "ark-mini":
        return calculate(ArkMiniPricing, tokensIn, tokensOut, audioSec)
    case "ark-depth":
        return calculate(ArkDepthPricing, tokensIn, tokensOut, audioSec)
    }
}
```

### 3.4 辩证结论:aicost 价格表抽离有价值但非紧急

| 判断 | 理由 |
|---|---|
| 价格表抽离有价值吗? | ✅ 有价值,支持价格动态配置 |
| 应该现在做吗? | ❌ 应在 B18 接入 Ark Mini 后启动 |
| 应该在 9/9 前合入吗? | ❌ 应在 W5(9/28)合入,B18 review 真实 LLM 后 |
| 应该列为 P0 吗? | ❌ 应为 P1,不影响业务功能 |

**建议执行策略**:
- **时间**:W5(9/28-10/4),B18 接入 Ark Mini 定价后
- **先决条件**:D-1 拍板的 Ark Mini + 深度思考定价表
- **接口影响**:session/service.go 调用 `config.CalculateCostFen()`

---

## 四、协议层冻结 · 唯一真正的 P0

### 4.1 提案原文

> 协议层与跨仓契约修正必须在 9/15 前完成,9/15 后严禁大改

### 4.2 代码验证

`voiceproto/frames.go` 当前帧类型清单:

```
已冻结:
- session.start / session.ready / session.end
- user.speech.start / user.speech.end
- client.asr.transcription (B14)
- ai.text.delta (V2.0 新增)
- ai.audio.chunk
- ai.turn.end (B15,带 outcome + log_id)
- feedback.badge (B12,带 turn_id)
- interrupt / error

待 W3 新增(B17/B19/B25/B22):
- ai.tts.start / ai.tts.audio / ai.tts.end (B17)
- POST /internal/v1/hits 的请求体含 turn_id (B19)
- PATCH /corpus/:id {pin/favorite} (B25)
- GET /drill/round / POST /drill/judge (B22)
```

**WSS V2.0 冻结清单**(必须 9/15 前确定):
1. `ai.tts.start` 帧字段(voice_id / sample_rate / codec)
2. `ai.tts.audio` 帧字段(Opus bytes / chunk_size)
3. `ai.tts.end` 帧字段(completion_status)
4. `ai.text.delta` 是否携带 `server_ts_ms` 字段(50_ §3.4)

### 4.3 致命风险量化

若 9/15 后发现协议漏了或类型错了:
- **影响范围**:iOS + Backend 双仓联调全部阻塞
- **回归成本**:iOS 端至少 2 人天,Backend 端至少 1 人天
- **时间损失**:至少 2 天(CCB 审批 + 同步修改)
- **对 W8 提审的影响**:可能顺延 2 天

### 4.4 辩证结论:协议冻结是唯一真正的 P0

**这是提案中最正确的判断**。

**建议执行策略**:
- **立即行动**:9/8(明天)召集 iOS Lead + 后端 TL,确认 WSS V2.0 帧字段
- **9/10(W3 Day1)**:在 W3 第一个 Issue(B19/B25)中先写 `ai.tts.*` 帧的草稿
- **9/13(W2 末)**:iOS 端开始接 `ai.tts.*` 帧的 mock decoder
- **9/15(W3 中期)**:封板,所有帧类型和字段锁定

---

## 五、修正时间轴的辩证校准

### 5.1 原始提案时间轴

```
9/6-9/9: 架构修正(Provider链+并发锁)  ← 提案核心
9/10:    合入主干
9/10-9/14: 各 Issue 开发
9/15:    协议冻结死线
W7:      低风险重构
```

### 5.2 校准后时间轴

```
9/6-9/9: 协议冻结确认(WSS V2.0 帧字段)  ← 唯一 P0
9/8:     sessionRuntime 加一行注释(非代码改动)
9/10:    W3 开工,各 Issue 按序启动
9/13:    iOS 端开始接 ai.tts.* mock decoder
9/15:    WSS V2.0 协议封板 ← 真正的死线
9/21(W4): Provider 链设计评审(B17 阶段一落地后)
9/28(W5): aicost 价格表抽离设计(B18 定价后)
W7:      低风险重构(sessionRuntime 无需动)
```

### 5.3 核心差异

| 原始提案 | 校准后 | 原因 |
|---|---|---|
| 架构修正(P0) | 协议冻结(P0) | 协议是唯一真正的 P0 |
| Provider 链 9/9 前合入 | Provider 链 W4 启动 | 9/9 合入会与 B17 冲突 |
| sessionRuntime 加锁 | sessionRuntime 不修正 | 不存在并发问题,过度工程 |
| aicost 价格表抽离 | aicost W5 启动 | 需 B18 定价后才有意义 |

---

## 六、给 W3 开工的实际落地建议

### 6.1 现在(9/6 今晚)

1. **召开 30 分钟协议冻结会**:iOS Lead + 后端 TL 确认 WSS V2.0 帧字段,输出到 `scratch/wss-v2-frames-2026-09-06.md`
2. **不修改 sessionRuntime**:加一行注释即可:

```go
// sessionRuntime 由 loop goroutine 串行访问,无需锁。
// 当且仅当 future 代码引入并发访问时才需要加 sync.Mutex。
type sessionRuntime struct {
    ...
}
```

3. **不启动 Provider 链重构**:在 77_ 的未决项中标注"W4 启动 Provider 链设计"

### 6.2 9/7-9/9

1. **起草 WSS V2.0 帧协议文档草稿**,包含 `ai.tts.start/audio/end` 帧的完整字段定义
2. **评审 73_ seed-tts-2.0 SDK 决策**:I15 用哪个 Opus 解码方案,影响 ai.tts.audio 的 codec 字段
3. **准备 B19/B25 的 GitHub Issue**:这两个无凭证依赖,W3 Day 1 可立即建仓

### 6.3 9/10(W3 Day1)

1. **拉取最新 main**:确认 schema + 迁移脚本 + 契约文档已同步
2. **按依赖顺序建仓**:B19 → B25 → B17 → B22
3. **B17 建仓前**:确认 WSS V2.0 `ai.tts.*` 帧协议草稿已完成

### 6.4 9/13-9/15

1. **iOS 端开始接 ai.tts.* mock decoder**:即使 B17 还没 CLOSED,iOS 可以先接 decoder
2. **WSS V2.0 封板**:所有帧类型 + 字段 + codec 锁定,不允许再改
3. **通知 PM/商务**:W3 进度看板更新

---

## 七、提案中其他项的代码验证

### 7.1 跨仓测试覆盖

**提案**:「无 e2e 框架,iOS 真机 × backend 集成缺失」

**代码验证**:
- ✅ 有 `cmd/integration-voice-gateway` 独立 cmd
- ✅ 有 6 个 smoke 脚本(smoke-corpus/drill/review-ark/...)
- ✅ 有 `scripts/e2e/w3-smoke.sh` 草稿(在 75_ §六中规划)
- ❌ smoke 脚本未串联成端到端
- ❌ XCUITest 跨仓框架未建立

**结论**:e2e 缺失是真实的,但应在 W5(B18 接通后)建立,不抢占 W3 前窗口。

### 7.2 Volc SDK 升级风险

**提案**:「Volc SDK 不定期升级可能 breaking change」

**代码验证**:确实存在。当前 `voicepoc/volc_duplex.go` 使用的字段:
- `X-Tt-Logid` header(非公开接口)
- `response.done` 事件名
- `response.error` 事件名

**结论**:Volc SDK 升级风险是真实的,建议在 W6 集中处理,避开 V2.0 开工周。

### 7.3 MySQL schema drift

**提案**:「staging vs production schema 可能不一致」

**代码验证**:
- ✅ 4 个 V2.0 迁移(0008/0009/0010/0011)已 commit
- ✅ 0012 待 B22 启动时创建
- ❌ production 是否已 apply 不确定

**结论**:真实风险,建议 9/13 前做一次 schema audit。不需要代码修正,需要运维操作。

---

## 七-B、资深 Go 架构师二审意见(含代码实证)

> 背景:资深 Go 架构师对 V1.0 审核回复的二次分析,补充了 3 个 Go 语言专属风险。逐条代码验证如下。

### 七-B.1 handler.go 状态膨胀风险(专家新增) → 辩证:有价值,但 W7 再做

**专家意见**:handler.go 约 19KB,W3-W5 业务爆发后状态字段膨胀(TTS/eval/hit),建议 9/10-9/14 将 `sessionRuntime` 抽离为独立 `state` 包。

**代码验证**:

```go
// handler.go:245-267
type sessionRuntime struct {
    started       bool
    startedAt     time.Time
    provider      VoiceProviderSession   // ← 不变,provider 层面
    broken        bool                  // B15: 音频路径失败
    reopenAttempted bool                // B15-followup #43
    warnDedup    struct { ... }        // B15: WARN 去重
}
```

V2.0 增量字段预测:

| 字段 | 所属 Issue | 在 sessionRuntime? | 实际位置 |
|---|---|---|---|
| `ttsPlaying bool` | B17 | ❌ | `volcDuplexProviderSession` 内 |
| `evalPending bool` | B18 | ❌ | session service 层 |
| `hitList []Hit` | B19 | ❌ | `BadgeEmitter` 内 |

**关键发现**:V2.0 新增的 TTS/eval/hit 状态**都不在 `sessionRuntime` 里**,而是在 provider 层(service 层/BadgeEmitter)。`sessionRuntime` 的字段是固定的网关层元数据,不会随业务爆发膨胀。

**辩证结论**:
- 专家的担忧有前瞻性,但代码实证不支持"handler.go 状态膨胀"的紧迫性
- 真正的状态膨胀风险在 `volcDuplexProviderSession` 而非 `sessionRuntime`
- 建议:W7 冲刺稳定期再做,理由:
  1. W3-W5 是功能交付期,基础设施重构与 feature 分支冲突概率高
  2. `handler.go` 当前结构清晰,W3-W5 新增的 TTS/eval 都在 provider 层,不会改 handler.go
  3. W7 做完 feature 后做 `state` 包,测试充分,无 feature 分支冲突
- **采纳专家建议**:做 `state` 包封装,但**改时间为 W7**,不在 W3 抢工

**ADR 记录**:
```markdown
## ADR-0072: sessionRuntime 状态封装时机

Status: PROPOSED
Created: 2026-09-06

Context: handler.go 约 19KB,V2.0 新增 TTS/eval/hit 状态是否会导致膨胀?

Decision: W7(10月中)稳定冲刺期抽离 sessionRuntime 为独立 state 包。
理由: W3-W5 是功能交付期,基础设施重构风险高。

Consequences: W7 前 handler.go 保持当前结构。
```

### 七-B.2 sql.Tx panic/recover 模式(专家新增) → 辩证:理论风险,非致命,随 aicost 价格表一并处理

**专家意见**:当前 `defer tx.Rollback()` 在 Commit 成功后触发会打印 WARN,建议统一 `WithTransaction` 辅助函数。

**代码实证**(`mysql_store.go:443-470`):

```go
func (s *MySQLStore) MarkSessionReviewedWithCost(...) (Session, error) {
    tx, err := s.db.BeginTx(ctx, nil)
    if err != nil { return Session{}, err }
    defer func() { _ = tx.Rollback() }()  // ← Commit 成功后触发,Discard

    session, err := scanSession(tx.QueryRowContext(...))
    if err != nil { return Session{}, err }
    switch session.Status {
    case StatusReviewed:
        if err := tx.Commit(); err != nil { return Session{}, err }
        return session, nil
    case StatusEnded:
        tx.ExecContext(ctx, `UPDATE practice_sessions ...`)  // ← 不 panic
        s.costTx(tx).RecordCostTx(ctx, tx, costLog)          // ← 待验证
        if err := tx.Commit(); err != nil { return Session{}, err }
        return session, nil
    }
}
```

**分析**:
1. `defer func() { _ = tx.Rollback() }()` — `Commit()` 成功后 `Rollback()` 返回 `sql.ErrTxDone`,被 `_` 丢弃,**不影响业务,日志不污染**
2. `s.costTx(tx).RecordCostTx(ctx, tx, costLog)` — `RecordCostTx` 内部无 panic 路径(纯 SQL 操作,无反射)
3. **真正的 panic 风险场景**:在 `tx.ExecContext` 和 `Commit` 之间 panic — 但这需要数据库驱动层 panic,而非业务代码

**Go SQL 驱动行为验证**:
```go
tx, _ := db.BeginTx(ctx, nil)
tx.Commit()
tx.Rollback()  // returns sql.ErrTxDone — already committed
// Go 的 sql.Tx 实现保证:已提交的 tx,Rollback() → sql.ErrTxDone,不是 panic
```

**专家的 `WithTransaction` 建议是最佳实践,但优先级低**:
```go
// 专家建议的防御模式(更优雅,但当前代码正确性无忧)
func WithTransaction(ctx context.Context, db *sql.DB, fn func(*sql.Tx) error) error {
    tx, err := db.BeginTx(ctx, nil)
    if err != nil { return err }
    defer func() {
        if p := recover(); p != nil { _ = tx.Rollback(); panic(p) }
    }()
    if err := fn(tx); err != nil { _ = tx.Rollback(); return err }
    return tx.Commit()
}
```

**辩证结论**:
- 当前代码**正确性无忧**,`sql.ErrTxDone` 被 `_` 丢弃,无 panic,无数据损坏
- 专家的 `WithTransaction` 建议是**代码风格最佳实践**,非 P0
- **采纳专家建议**:随 W5 `aicost` 价格表抽离一并处理,理由:
  1. `aicost` 价格表重构时必然涉及 `session/service.go` 的定价调用,顺带升级事务模式
  2. `WithTransaction` 作为 `internal/dbutil/tx.go` 独立工具函数,全仓可用
  3. W5 正好在 W4 Provider 链之后,团队已熟悉事务处理

### 七-B.3 WSS 写入缓冲池(专家新增) → 辩证:认可,W7 稳定期验证

**专家意见**:当前每个 binary 帧直接 `conn.Write`,50fps 高频下 syscall 压力,建议 W7 用 `sync.Pool` 或 `bufio.Writer`。

**代码验证**:
```go
// handler.go handleAudio
func (h *Handler) handleAudio(...) error {
    // 20ms 一帧 → 50fps → 每秒 50 次 syscall Write
    return h.writeOutbounds(ctx, conn, outbounds)  // 直接 Write
}
```

**评估**:
- 50fps × 640 bytes = 32KB/s,实际 syscall 压力在 50-200fps 突发场景下才显著
- Go 的 `net.Conn.Write` 本身有 syscall 缓冲,但没有应用层 buffer
- **专家的 sync.Pool 建议**:针对 binary 帧的 `[]byte` 复用,减少 GC 压力

**辩证结论**:
- **采纳专家建议**,归入 W7 稳定期性能优化
- **理由**:
  1. W3-W6 业务场景下,单用户 50fps 是正常流量,不是瓶颈
  2. W7 提审前必须做性能基线验证,届时量化验证 syscall 压力
  3. `sync.Pool` 改动需要回归测试,不在 W3-W6 冒险

---

## 八、总结:修正类型精确分类(V2.0 校准)

### 8.1 校准后的修正分类

| 类型 | 内容 | 时间 | 优先级 | 来源 |
|---|---|---|---|---|
| 🔴 **P0 协议冻结** | WSS V2.0 ai.tts.* 帧字段 + 50_ server_ts_ms 决策 | **现在→9/15** | P0 | 提案+专家共识 |
| 🟡 **P1 架构增强** | Provider 链设计(ADR) + aicost 价格表 + WithTransaction | W4-W5 | P1 | 提案+专家共识 |
| 🟡 **P1 架构增强** | sessionRuntime 状态包抽离 | W7 | P1 | 专家新增,采纳,W7 再做 |
| 🟢 **P2 技术债务** | Volc SDK 升级 + 跨仓 e2e 框架 | W6-W7 | P2 | 提案 |
| 🟢 **P3 性能优化** | WSS 写入缓冲池(sync.Pool) | W7 | P3 | 专家新增,采纳 |
| ✅ **无需修正** | sessionRuntime 加锁 | — | — | 提案误解,代码实证推翻 |
| ✅ **无需修正** | Provider 链 9/9 前合入 | — | — | 时间轴错误,应 W4 |

### 8.2 分支策略(V2.0 校准)

| 分支 | 内容 | 创建时间 | 合并时间 | 理由 |
|---|---|---|---|---|
| `main` 当前 | 协议冻结草稿 + sessionRuntime 注释 | — | — | 评审通过前不动 main |
| `feature/wss-v2-protocol` | WSS V2.0 帧协议草稿 | **9/6 今晚** | 9/15 | 最优先,唯一真正的 P0 |
| `feature/provider-chain` | Provider 链 ADR(架构决策记录) | W3 末(9/14) | W4 末 | B17 阶段一落地后启动 |
| `feature/aicost-pricing` | 价格表抽离 + WithTransaction | W5(9/28) | W5 末 | B18 定价后才有意义 |
| `feature/session-state-pkg` | sessionRuntime 状态包抽离 | W7(10月中) | W7 末 | 稳定冲刺期才做 |
| `feature/wss-write-buffer` | sync.Pool 写入缓冲 | W7(10月中) | W7 末 | 性能基线验证后 |

---

## 九、行动清单(9/6 今晚到 9/10)V2.0

- [ ] **9/6 今晚**:召开 30 分钟协议冻结会,输出 WSS V2.0 帧字段草稿
- [ ] **9/6 今晚**:在 `sessionRuntime` 加一行注释(无代码改动)
- [ ] **9/6 今晚**:在 `feature/wss-v2-protocol` 分支 push WSS V2.0 帧 Markdown 草稿
- [ ] **9/7-9/8**:起草 WSS V2.0 `ai.tts.*` 帧协议 + 同步到 `fluentwork-infra/schemas/`
- [ ] **9/8-9/9**:评审 seed-tts-2.0 SDK 决策,确认 Opus 解码方案
- [ ] **9/10(W3 Day1)**:B19 + B25 建仓,B17 确认凭证到位后建仓
- [ ] **9/13**:iOS 端开始接 ai.tts.* mock decoder + 跨仓空跑验证
- [ ] **9/14**:W3 末,Provider 链 ADR 设计评审(明确 VoiceProvider vs TTSProvider 边界)
- [ ] **9/15**:WSS V2.0 协议封板,不允许再改(唯一 P0 硬死线)
- [ ] **W4 末**:Provider 链代码实现(B17 阶段一落地后)
- [ ] **W5 末**:aicost 价格表抽离 + WithTransaction 工具函数
- [ ] **W7**:sessionRuntime 状态包抽离 + WSS sync.Pool 写入缓冲

---

## 十、文档决策(V2.0 校准)

| 文档 | 操作 | 状态 |
|---|---|---|
| 77_ §11.2 重点难点 | 更新:「sessionRuntime 无需加锁,W7 做状态包抽离」 | ⏸ 待合并后 |
| 77_ §15 未决项 | 更新:「Provider 链 W3 末 ADR,W5 aicost 价格表,W7 状态包」 | ⏸ 待合并后 |
| 50_ 全链路架构 | 更新:「WSS V2.0 冻结死线 9/15」 | ⏸ 待合并后 |
| 本文档 | 评审通过后合入 main | ⏸ 待评审 |

---

## 十一、专家二审意见采纳记录

| 专家建议 | 代码实证 | 采纳 | 调整内容 |
|---|---|---|---|
| sessionRuntime 状态包抽离(W3 中期) | V2.0 新增状态不在 sessionRuntime | ✅ 采纳 | 改时间为 W7 |
| sql.Tx WithTransaction 模式 | Rollback 在 Commit 成功时不 panic | ✅ 采纳 | 随 W5 aicost 价格表一并处理 |
| WSS 写入缓冲池 | 50fps syscall 压力待量化 | ✅ 采纳 | W7 性能基线验证后执行 |
| Provider 链 ADR W3 末评审 | VoiceProvider vs TTSProvider 边界待明确 | ✅ 采纳 | W3 末(9/14)先出 ADR,W4 才写代码 |
| 跨仓空跑验证(9/14 前) | dev-echo + InMemorySocketTransport 已具备 | ✅ 采纳 | 9/13 iOS 端接 mock decoder |

---

*文档版本:V2.0(2026-09-06 晚场二次评审,含资深 Go 架构师二审意见)*
*feature 分支:feature/backend-arch-review-2026-09-06*
*评审通过条件:后端 TL + iOS Lead 确认 §四 协议冻结清单 + §十一 采纳记录*
*合并前检查:sessionRuntime 注释已加,WSS V2.0 帧协议草稿已评审,ADR 评审已规划*
