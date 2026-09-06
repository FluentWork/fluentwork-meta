# B17 TTS Provider + 流式集成 — GitHub Issue 草稿

**仓库**：`FluentWork/fluentwork-backend`
**优先级**：P0（W3 启动首日，依赖 D-2 凭证到位）
**估时**：1.5 dev-day
**关联文档**：
- `docs/30_技术方案/48_FluentWork_V2_REST接口契约冻结_2026-09-06.md` §0
- `docs/40_研发流程与协作/47_D1_D5_开放技术决策备忘录_2026-09-03.md` V1.1 §D-2
- `docs/30_技术方案/34_FluentWork火山引擎选型与开通清单.md`
- `docs/40_研发流程与协作/55_FluentWork_V2_W3_代码层启动包_2026-09-06.md` §一
- 父 Issue：无（首启动）

---

## 🎯 目标

在 `internal/content/tts` 新建 TTS Provider 抽象层，落地火山独立 TTS（主路径）+ 双工内置 TTS（fallback），按 D-2 拍板音色表实施。给后续 B20 每日一读批量生成、iOS I15 流式播放铺平接口。

---

## 🚧 阻塞条件

- **D-2 凭证到位**（C-2：火山 TTS API Key + 流式 TTS endpoint + 音色列表确认）—— 53_ §Layer 1 SLA: W3 前（即 9/10 之前）
- 火山 TTS SDK 选型（go.mod 添加 `volcengine-go-sdk` 或自行 HTTP/2 包装—— 由实施者拍板，issue comment 留痕）

---

## 📋 实施步骤

### 1. Provider 接口定义（`internal/content/tts/provider.go`）

```go
package tts

import "context"

type VoiceConfig struct {
    VoiceID string  // D-2 表音色 ID
    Speed   float64 // 0.85 ~ 1.0
}

type AudioChunk struct {
    Data       []byte // Opus frame
    Seq        int    // 流式序号
    IsFinal    bool   // 最后一片
    DetectedAt int64  // 本片生成时间戳（ms）
}

type Provider interface {
    // Stream 返回的 channel 在 ctx cancel 或服务端断开时关闭
    Stream(ctx context.Context, text string, voice VoiceConfig) (<-chan AudioChunk, error)
    // Ping 健康检查（用于 fallback 切换判断）
    Ping(ctx context.Context) error
    Close() error
}
```

### 2. 火山独立 TTS 实现（`internal/content/tts/volc_streaming.go`）

- 火山 TTS HTTP/2 StreamTTS 接入（参考火山官方 SDK 或 REST API）
- 流式首字目标 P90 ≤ 400ms
- 错误处理：5xx 连续 3 次 → 触发 fallback（计数写到 metrics）

### 3. 双工内置 TTS Fallback（`internal/content/tts/volc_duplex_fallback.go`）

- 火山双工内置 TTS（音色固定，已在 V1.x 接入；本任务仅抽象为 fallback Provider）
- 仅在主 Provider `Ping` 失败 3 次时切换
- 切换后写 metric：`tts_fallback_triggered_total{from="volc_streaming", to="volc_duplex"}`

### 4. 音色表常量（`internal/content/tts/voices.go`）

按 D-2 拍板音色表定：

```go
// D-2 拍板音色表，详见 47_D1D5_备忘录 V1.1 §D-2
var (
    VoiceAIMaleTech         = VoiceConfig{VoiceID: "zh_male_tech_01", Speed: 0.9}    // AI 男性（PM/Tech Lead）
    VoiceAIFemalePro        = VoiceConfig{VoiceID: "en_female_professional", Speed: 1.0} // AI 女性（SRE/Designer）
    VoiceDailyReadNarrator  = VoiceConfig{VoiceID: "en_male_narrator", Speed: 0.85}  // 每日一读
    VoiceDrillCountdown     = VoiceConfig{VoiceID: "en_female_clear", Speed: 1.0}    // 闪测倒计时
)
```

### 5. Handler 注入

- `internal/content/handler.go` 中 `GetToday` / `PostFollowRead` 调用 `tts.Provider.Stream(...)` 而非现有硬编码
- 流式分片写入 HTTP response（`Content-Type: audio/opus`）

---

## ✅ 测试矩阵

| ID | 场景 | 预期 |
|---|---|---|
| T-B17-1 | 火山独立 TTS 流式首字 | P90 ≤ 400ms（端到端从 Stream 调用到第一片） |
| T-B17-2 | 双工内置 TTS fallback 触发 | 5xx 连续 3 次 → 切 fallback + 写 metric |
| T-B17-3 | 音色切换 | 同一 Provider 换 `VoiceConfig{VoiceID: ...}` → 实际音色变更 |
| T-B17-4 | 流式断流 | `ctx cancel()` → client channel 关闭 + 收到 error |
| T-B17-5 | Ping 健康检查 | 正常 200ms 内返回 |
| T-B17-6 | OpenAPI 同步 | `api/openapi-v1.yaml` 加 TTS Provider 配置说明段落 |

---

## 📐 估时拆分

| 步骤 | 估时 |
|---|---|
| 1. Provider 接口 + 单元测试 | 0.3 dev-day |
| 2. 火山独立 TTS 实现 + 集成测试 | 0.6 dev-day |
| 3. 双工内置 TTS fallback | 0.2 dev-day |
| 4. 音色常量 + handler 注入 | 0.2 dev-day |
| 5. 测试 + 文档 + OpenAPI 同步 | 0.2 dev-day |
| **合计** | **1.5 dev-day** |

---

## 🎯 DoD（Definition of Done）

- [ ] `go test ./internal/content/tts/...` 6 个测试通过
- [ ] 集成测试：与真实火山 TTS 端到端 1 次成功（凭证到位后）
- [ ] OpenAPI 同步：手动编辑 `api/openapi-v1.yaml` 加 TTS 配置说明
- [ ] `48_` 契约冻结文档 §1.6.1 确认 TTS 集成路径（无需改动，仅核对）
- [ ] PR 通过 OpenCodeReview 自动 review + 1 名后端 maintainer review
- [ ] Issue description 链接本文 + `55_` 启动包
- [ ] 完成后状态写到 53_ §Layer 2 完成状态表（CLOSED ✅）

---

## 🔗 关联 Issue

- 上游：无（首启动）
- 下游：B20 每日一读内容生成（用 TTS 批量）、I15 iOS TTS 播放集成（用流式 API）
- 平行：B19 / B22 / B25（见 55_ 启动包依赖图）

---

## 📝 备注

- 火山 TTS SDK 选型记录到本 Issue comment（不阻塞实施但需留痕）
- V1.5+ 评估 ElevenLabs 作为多音色扩展时，本抽象层的 `Provider` 接口不变——只需新增实现
