# 36 FluentWork 可观测性与事件 Schema 设计草案

## 定位

本文件定义 FluentWork 跨仓事件与埋点 schema 的统一入口。

目标不是只覆盖 iOS `I12` 语音链路，而是沉淀一套可扩展的 schema 约束，后续可继续吸纳：

1. 语音链路事件
2. 网络与重连事件
3. review 生成与消费事件
4. 语料库同步事件
5. 购买、转化、订阅等商业埋点
6. infra / worker / 网关可观测性事件

在独立 infra 仓尚未建立前，本文档以 `fluentwork-meta` 为 source of truth。

## 设计原则

1. schema 先行，再写 logger / tracker / analytics adapter
2. 字段命名统一 snake_case
3. 同名字段跨端、跨服务语义必须完全一致
4. error / reason / source 必须先枚举化，禁止自由文本扩散
5. 事件定义与 transport / API contract 解耦，但允许共享字段枚举

## 事件分层

### 1. Domain Event

表达业务事实，不关心具体上报平台。

示例：

- `speech_session_started`
- `speech_interrupt_local`
- `speech_reconnect_started`
- `review_generation_completed`
- `corpus_delta_sync_applied`
- `subscription_checkout_started`

### 2. Transport / Contract Event

表达协议边界事件，驱动状态机或跨端协作。

示例：

- `session.start`
- `user.speech.start`
- `ai.audio.chunk`
- `ai.turn.end`
- `session.end`

### 3. Analytics Event

表达最终投递给具体平台的埋点记录。

示例：

- `ios.analytics.speech_session_started`
- `backend.analytics.review_generation_completed`

## 基础字段草案

所有 domain / analytics 事件优先复用以下基础字段：

- `event_name`
- `event_version`
- `event_time`
- `session_id`
- `user_id`
- `device_id`
- `trace_id`
- `platform`
- `app_version`
- `build_version`
- `source`
- `phase`
- `reason`
- `error_code`
- `error_message`
- `elapsed_ms`

## 语音链路首批字段

### 建议事件

1. `speech_session_started`
2. `speech_session_failed`
3. `speech_session_ended`
4. `speech_interrupt_local`
5. `speech_interrupt_forwarded`
6. `speech_reconnect_started`
7. `speech_reconnect_succeeded`
8. `speech_reconnect_timed_out`
9. `speech_audio_capture_started`
10. `speech_audio_capture_stopped`
11. `speech_audio_first_chunk_received`
12. `speech_turn_ended`
13. `speech_transport_disconnected`
14. `speech_transport_degraded`

### 建议字段

- `network_state`
- `audio_route`
- `is_reconnecting`
- `interrupt_watermark`
- `turn_id`
- `transport_type`

## 当前已冻结的一项 transport 修正

为消除 iOS 端对 `aiAudioEnd` 的 debounce 推断，WSS transport schema 补充显式控制帧：

- `ai.turn.end`

语义：

1. 表示 AI 当前回合结束
2. 不要求该回合一定产生音频
3. 可同时覆盖纯文本回合与音频回合

## 推荐落位

### 当前阶段

source of truth 放在：

- `fluentwork-meta/docs/30_技术方案/36_FluentWork可观测性与事件Schema设计草案.md`

### 后续若建立 infra 仓

可迁移为：

1. `infra/schemas/events/*.json`
2. `infra/schemas/transport/*.json`
3. `infra/docs/observability/*.md`

迁移条件：

1. 至少两个运行时开始共享同一套 schema 文件
2. 需要自动生成校验器、typed client 或埋点 adapter
3. 需要 CI 对 schema 变更做 breaking-change gate

## 验收建议

1. schema 文档进入 meta
2. 关键枚举值冻结
3. iOS / backend 至少一条链路开始按 schema 对齐
4. 后续需要落地 JSON Schema 或 codegen 时，再迁到 infra
