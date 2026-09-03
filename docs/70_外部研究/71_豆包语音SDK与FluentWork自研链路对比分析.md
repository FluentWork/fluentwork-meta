# 豆包语音官方 SDK 与 FluentWork 自研语音链路对比分析

> 分支：`analysis/doubao-voice-sdk-vs-custom`  
> 日期：2026-09-03  
> 范围：只做外部能力盘点与架构对比，不涉及代码改动

## 一句话结论

**豆包语音并不是"没有 SDK"**：火山引擎（BytePlus / Volcengine）对 iOS 已经提供了端到端实时语音对话 SDK（Dialog 语音 SDK）、流式语音识别 SDK（`SpeechEngineToB` / `SpeechEngineAsrToB`）以及音频技术 SDK（音高 / VAD / 音量 / 响度检测等）。

FluentWork 之所以"自己完成整条链路"，**不是因为厂商缺 SDK，而是因为现网架构选择了一条与官方客户端 SDK 形态不同的路线**：iOS 端不持有任何第三方凭证，PCM 通过自有的 speaking-room WSS 上行到 voice-gateway，由后端持有 Volc API Key 与豆包 Duplex 直连，再做语料命中 / badge / review 等自有业务。官方 SDK 都是"客户端直连 / 端到端接管录音与播放"的形态，直接套用会破坏这条约束，因此项目选择自研 relay 是架构取舍，不是能力空白。

---

## 1. 问题拆解

"豆包语音没有自己的 SDK 帮助用户完成语音识别、VOD 分析上报等功能吗？"这句话需要先拆成三个独立能力再回答：

1. **语音识别（ASR / 转写）**：把麦克风 PCM 实时转成文字；
2. **VAD / 人声分析**（按语境判断 "VOD" 应为 VAD）：检测"开始说话 / 结束说话 / 静音间隔"，做话轮切分；
3. **音频数据与上报**：拿到采集后的原始音频、音量、状态事件并上抛给业务方做展示 / 埋点 / 反馈。

火山引擎对这三块**都有对应官方 SDK 或能力**，但粒度、持有方与产品形态和 FluentWork 当前的自研链路不一样。

---

## 2. 官方能力盘点（证据）

### 2.1 端到端实时语音对话：Dialog 语音 SDK（iOS）

- 官方文档：端到端 iOS SDK 接口文档（豆包语音）
  <https://docs.volcengine.com/docs/6561/1597646?lang=zh>
- 产品说明原文：Dialog 语音对话 SDK 基于豆包端到端实时语音大模型，**使用设备自身的录音机与播放器**，并把识别 / 对话 / TTS 状态与文本通过 delegate 回调给业务。
- 覆盖的能力：录音采集、播放、每轮"说完了"标志、ASR / 对话状态事件、TTS 文本与原始音频回调（可关闭 SDK 内置播放器、自己拿音频数据）。

这是与 FluentWork speaking-room 形态**最接近**的官方 SDK：别人已经把"录音 → 对话 → 播报"封装好。但它默认是一个**客户端直连、SDK 接管完整会话**的黑盒组件。

### 2.2 纯 ASR 转写：流式语音识别 iOS SDK

- 集成指南：<https://www.volcengine.com/docs/6561/113641>（更名历史说明见 <https://www.volcengine.com/docs/6561/113643>）
- 组件名：`SpeechEngineToB`，2023-09 后统一为 **`SpeechEngineAsrToB`**；iOS 通过 CocoaPods `volcengine-specs` 集成。
- 接口 / 版本说明：<https://docs.volcengine.com/docs/6561/1395846>
- 大模型流式语音识别 iOS SDK 接口文档：<https://docs.volcengine.com/docs/6561/2604753>
- 能力：实时"边说边出文字"，支持部分结果 / 最终结果、句级与词级切分、标点等；适合会议字幕、直播字幕等纯转写场景。

它只解决 ASR，不负责对话编排、TTS 播报或 FluentWork 的语料命中。

### 2.3 离线音频分析（音高 / VAD / 音量 / 响度）：音频技术 SDK

- SDK 概述：<https://www.volcengine.com/docs/6489/1166627>
- 快速入门（含 iOS 静态 / 动态库）：<https://www.volcengine.com/docs/6489/171423>
- 功能面：音频检测能力包含**音高检测、语音活性检测（VAD）、音量检测、响度检测、延迟检测**等。

这可以替换项目里手写的能量阈值 VAD，但引入的是一个偏"本地算法库"形态的二进制 SDK。

### 2.4 同厂商的另一种实时对话路线：RTC + 云端 VAD

- 参考：<https://github.com/volcengine/ai-app-lab/blob/main/demohouse/rtc_conversational_ai/README.md>
- 形态：用火山 RTC 采集音频上行，在**云端**做人声检测与全双工编排，再下发 AI 音频；客户端主要做采集 / 播放与状态展示。

这条路与 FluentWork 的"客户端采集 + 后端中继"思路一致，只是把传输层换成 RTC 而不是自有 WSS。

### 2.5 底层 Duplex API 本身仍是裸 WebSocket JSON

- FluentWork 直连的 `wss://openspeech.bytedance.com/api/v3/duplex/realtime/dialogue` 是**协议层 API**：上行 `session.create`、`input_audio_buffer.append` 等 JSON 帧，下行各类事件。
- 第三方 Go 封装（社区维护，可作协议参考）：<https://github.com/GizClaw/doubao-speech-go/blob/main/docs/realtime_duplex.md>
- 结论：厂商的"API 能力"和"iOS 高层 SDK"是两个层次。**官方高层 SDK 提供的是开箱即用的客户端会话组件，不等于协议本身；只要想自定义会话编排 / 话轮判定 / 混入自有业务逻辑，业界普遍会在协议层自建。**

---

## 3. FluentWork 现状：到底"自己做了什么"

代码证据（以 2026-09-03 分支为准）：

| 环节 | FluentWork 的实现 | 位置 |
| --- | --- | --- |
| 麦克风采集 / PCM | `AVAudioEngine`，16kHz mono s16le，20ms chunk | `fluentwork-ios/Shared/FluentWorkCore/Services/LiveAudioEngine.swift` |
| VAD / 话轮判定 | 能量阈值 `speechThreshold` + 静音保持 `silenceHold` 的自研状态机 | 同上 + `Shared/FluentWorkCore/SpeechSession/` |
| 传输 | `URLSessionWebSocketTask` + 自有帧协议 + 30s ping / 断线处理 | `Shared/FluentWorkNetworking/Socket/URLSessionSocketTransport.swift` |
| 凭证 | iOS 不持有任何第三方 key；只带 `ticket` | `SocketTransportEventMapper.swift` / `AppDependencies.swift` |
| 后端会话 | voice-gateway 持有 Volc API Key，relay 到 Duplex | `fluentwork-backend/internal/voicegateway/provider_volc_duplex.go`、`internal/voicepoc/volc_duplex.go` |
| ASR 文本回流 | `client.asr.transcription` 帧 → 中间件 → 展示 reducer | `internal/voiceproto/frames.go` + iOS `SocketTransportEventMapper.swift` |
| 业务命中 | 服务端 corpus-backed `feedback.badge` 发射 + dedupe | `internal/voicegateway/badge_emitter.go` |

设计依据（既有文档）：

- `fluentwork-meta/docs/30_技术方案/37_FluentWork-B14_Client_ASR_Relay_Architecture.md`
  - 核心决策表：ASR 引擎 = Volcengine Duplex，credential 由 backend voice gateway 持有；**iOS 是否持有 API Key = 否**；ASR 结果经现有 speaking-room WSS 回传。
- `fluentwork-ios/docs/13_ClientASR集成与使用指南.md`
  - 明确记录 B13 → B14 的演进：一开始计划 Apple Speech / Volcengine 客户端 SDK，**B14 后改为服务端中继并移除客户端转写调用**；`ClientASRTranscriber` 协议与实现仅保留供未来复用。
- `VolcengineClientASRTranscriber.swift`（iOS）中仍保留注释：

  ```swift
  /// This is a placeholder implementation that will be completed
  /// once the Volcengine SDK is integrated (tracked in B14).
  ```

  它是历史遗留的"官方 SDK 接入位"，不是当前运行代码。

---

## 4. 三条候选路径对比

| 维度 | A. 现状：自研 WSS relay（默认） | B. 官方 Dialog iOS SDK 客户端直连 | C. 官方 Streaming ASR SDK 客户端直连（仅转写） |
| --- | --- | --- | --- |
| 凭证位置 | 只在后端（iOS 零 key） | 需要把 App 凭证 / Token 放到客户端，或自行做 token 服务 | 需要客户端持有 AppID / Token / Access Token |
| ASR | 后端 relay 豆包 Duplex 的权威转写 | SDK 内部完成，客户端拿结果 / 状态 | 客户端直连流式 ASR |
| 录音 / 播放 | 自研 `LiveAudioEngine`（可控、可插桩、可量化） | SDK 接管设备录音机 / 播放器（省自研，但丢失中间层控制） | 仍需自研采集与播放（SDK 只转写） |
| VAD / 话轮 | 自研能量 + 状态机（当前已可用） | SDK 自带"说完"判定，业务自定义粒度受限 | 无（或需另接音频技术 SDK / RTC VAD） |
| 产品定制（badge 命中 / 语料 / review / daily-read） | 与现有后端业务深度耦合，天然可达 | 需把命中逻辑搬回客户端或再做一层会话代理 | 完全无，只解决文字 |
| 弱网 / 失败恢复 | 全链路自控，已做重开 / ping / drop-gate | 依赖厂商 SDK 内部策略，排障黑盒 | 依赖厂商 SDK 内部策略 |
| 改造成本 | 0（现状） | 中高：加 SDK、迁移协议、重写 transport 层、处理凭证 | 中：只换 ASR 段，仍保留 relay 结构，但失去后端权威转写 / 零 key 约束 |

补充选项 D（局部替换）：只把"能量阈值 VAD"换成火山音频技术 SDK，属于低收益可选优化；当前阈值 + 静音保持已经能在真机上工作，且引入二进制 SDK 需要重新走包体 / 合规 / 权限评估。

---

## 5. 结论与建议

1. **对当前已验收的主链路，不切换、不追加官方 SDK。**
   现状（零 key 后端 relay + 自有帧协议 + 后端业务判定）是既有架构决策的产物，链路已跑通；官方 SDK 并不是必须补的"欠账"。

2. **正确表述是"厂商有 SDK，但 SDK 形态与我们的架构约束冲突"，而不是"豆包没有 SDK"。**
   后续对外或写 PRD 时，应避免把"自研链路"表述成"厂商能力缺失"。

3. **保留复用接缝，未来按需再做 direct mode。**
   若未来出现"低时延直连""设备离线 ASR""苹果侧中文识别率不足需换引擎"等真实诉求，可优先利用现有协议层：

   - `ClientASRTranscriber`（iOS 保留文件）是官方 SDK 接入位；
   - 后端已有 `VOICE_CLIENT_ASR_REQUIRED` 灰度 gate（B13 已实现）；
   - 可设计"后端签发短期 token 的 direct mode"，在需要时绕过 relay，而不推翻当前架构。

4. **可选的轻量改进**：如果实测中 VAD 误切 / 漏切成为体验问题（目前真机日志未显示此问题），再评估火山音频技术 SDK 只替换"端点检测"一个环节。

---

## 6. 证据索引

官方 / 外部资料：

- Dialog 端到端 iOS SDK：<https://docs.volcengine.com/docs/6561/1597646?lang=zh>
- 端到端全双工 iOS SDK 文档（豆包语音）：<https://docs.volcengine.com/docs/6561/2556358?lang=en>
- 流式语音识别 SDK 集成：<https://www.volcengine.com/docs/6561/113641>、<https://www.volcengine.com/docs/6561/113643>
- 流式识别组件接口 / 版本：<https://docs.volcengine.com/docs/6561/1395846>
- 大模型流式语音识别 iOS SDK：<https://docs.volcengine.com/docs/6561/2604753>
- 音频技术 SDK 概述 / 快速入门：<https://www.volcengine.com/docs/6489/1166627>、<https://www.volcengine.com/docs/6489/171423>
- RTC 对话式 AI 参考：<https://github.com/volcengine/ai-app-lab/blob/main/demohouse/rtc_conversational_ai/README.md>
- Duplex 协议第三方参考：<https://github.com/GizClaw/doubao-speech-go/blob/main/docs/realtime_duplex.md>

项目内代码 / 文档：

- `fluentwork-meta/docs/30_技术方案/37_FluentWork-B14_Client_ASR_Relay_Architecture.md`
- `fluentwork-ios/docs/13_ClientASR集成与使用指南.md`
- `fluentwork-ios/Shared/FluentWorkCore/Services/LiveAudioEngine.swift`
- `fluentwork-ios/Shared/FluentWorkNetworking/Socket/URLSessionSocketTransport.swift`
- `fluentwork-ios/Shared/FluentWorkCore/Services/VolcengineClientASRTranscriber.swift`（历史接入位）
- `fluentwork-backend/internal/voicepoc/volc_duplex.go`
- `fluentwork-backend/internal/voicegateway/provider_volc_duplex.go`
- `fluentwork-backend/internal/voicegateway/badge_emitter.go`
- `fluentwork-backend/internal/voiceproto/frames.go`

---

## 7. 后续动作建议

- 归档本分析到 `70_外部研究`，作为"为何不接官方 SDK"的可检索结论，避免后续重复调研；
- 若后续要做 direct mode，建议单独开 issue，引用本文件第 5.3 节作为接入约束与接缝；
- 不需要为本次分析改动 backend / iOS 代码。
