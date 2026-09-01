# 36 FluentWork 可观测性与事件 Schema 设计草案

本文件已迁移到 `fluentwork-infra`，`meta` 不再作为该类 schema 的 source of truth。

当前主文档路径：

- `fluentwork-infra/docs/observability/00_FluentWork可观测性与事件Schema设计.md`
- `fluentwork-infra/docs/observability/01_共享Schema实现设计.md`

迁移原因：

1. `infra` 仓已存在，可承接跨仓 contract / schema 治理
2. 这类定义后续不仅覆盖语音链路，也会继续扩展到购买、订阅、review、worker、网关等事件
3. `meta` 仍保留产品与技术设计文档，但不再持有 schema 主定义

当前约定：

1. 文档说明在 `infra/docs/observability/`
2. 业务/分析 schema 在 `infra/schemas/events/`
3. transport schema 在 `infra/schemas/transport/`
4. 当前已落地 canonical 文件包括：
   - `infra/schemas/transport/wss-control-frames-v1.json`
   - `infra/schemas/events/speech-observability-events-v1.json`

## iOS 端 `source=ios` 约定

跨端事件 join 时按 `source` 区分事件来源。当前 schema（`infra/schemas/events/speech-observability-events-v1.json` 的 `$defs.eventBase.properties.source.enum`）已枚举四个值：

| source | 出处 | 典型事件 |
|---|---|---|
| `ios` | iOS 客户端（SwiftUI app） | `speech_turn_ended`、`speech_interrupt_local`、`speech_audio_first_chunk_received` |
| `backend` | app-server / 后端业务进程 | `review_generation_completed`、`subscription_checkout_started` |
| `voice_gateway` | 实时语音网关（`internal/voicegateway`） | `speech_reconnect_started`、`speech_session_failed`、`feedback_badge_emitted` |
| `worker` | 异步 worker（review / 训练 / 同步） | `corpus_delta_sync_applied`、`review_generation_failed` |

iOS 端埋点约束：

1. 所有由 iOS 发出的 domain / analytics 事件 `source` 字段固定写死 `"ios"`，不得动态拼接，避免回流分析时按 source 切分产生噪音
2. iOS `Tracker` 当前实现（`fluentwork-ios/Shared/FluentWorkCore/Observability/Tracker`）默认即 `"ios"`；新增 tracker 调用点时无需显式传 source
3. 与 backend `voice_gateway` 同名事件（如 `speech_turn_ended`）可按 `(session_id, turn_id, source)` 三键 join：iOS 侧是客户端 VAD 触发点，backend 侧是 ASR 完成点，两个时间戳共同勾勒一次完整 turn
4. `phrase_block_id` 例见 infra 文档新增的 `### phrase_block_id：badge dedupe 跨端关联键` 段
