# B14 · Client ASR Relay 架构设计

> 状态：已实现
> 日期：2026-09-02

## 背景

B13 在 iOS 端引入了 `ClientASRTranscriber` 协议，默认实现为 Apple Speech framework。本地 ASR 在部分设备上对中文普通话的识别准确率不足，影响用户体验。

B14 改为：**iOS 不持有任何第三方 API Key，通过现有的 speaking-room WSS 会话，将 ASR 转写工作完全委托给后端 voice gateway，由后端 relay 豆包语音（Volcengine Duplex）的识别结果。**

## 架构决策

| 决策 | 选择 |
|------|------|
| ASR 引擎 | Volcengine Duplex（豆包语音），由 backend voice gateway 持有 credential |
| iOS 是否持有 API Key | **否** — 无需在 iOS 存放任何第三方凭证 |
| ASR 结果传递路径 | 现有 speaking-room WSS 连接 + 新增 `client.asr.transcription` 帧 |
| 回退机制 | 当 voice gateway 未配置 Volcengine 凭证时，iOS 可通过 DI 注册 `AppleSpeechClientASRTranscriber` 覆盖默认实现 |

## 数据流

```
iOS (LiveAudioEngine)
        │ PCM (16kHz, mono, s16le)
        ▼
  [user.speech.start] ──────────── WSS frame
        │
        ▼
  [audio frames: binary PCM]
        │
        ▼
  [user.speech.end] ────────────── WSS frame
        │
        ▼
Backend Voice Gateway
  VolcDuplexProvider
    ├─ PCM → Duplex WSS → 豆包
    └─ Doubao ASR Transcript ◄────────────────┐
        │                                        │
        ▼                                        │
  [client.asr.transcription] ←── WSS frame ─────┘
        │  { type: "client.asr.transcription",
        │    text: "用户说的内容", turn_id: "turn-1" }
        ▼
  SocketTransportEventMapper
        │  .control(.clientASRTranscription(text:turnID:))
        ▼
  SpeechSessionMiddleware
        ├─ dispatch .serverASRReceived(text:turnID:)
        │    → SpeakingRoomState.liveTranscript (UI 显示)
        └─ sendSpeechBoundary(text: serverText)
              → Backend badge hit detection (B12)
```

## WSS 帧设计

### 新增帧类型

#### `client.asr.transcription`（Server → Client）

```json
{
  "type": "client.asr.transcription",
  "text": "用户说的内容",
  "turn_id": "turn-1"
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `type` | string | 帧类型常量：`client.asr.transcription` |
| `text` | string | 豆包返回的完整转写文本 |
| `turn_id` | string? | 对应语音段的 turn 标识，用于与 badge hit 关联 |

## 后端改动

### `internal/voiceproto/frames.go`

- 新增常量 `TypeClientASRTranscription = "client.asr.transcription"`
- 新增结构体 `ClientASRTranscription { Type, Transcript, TurnID }`

### `internal/voicegateway/provider_volc_duplex.go`

`turnToOutbound()` 在有 Doubao ASR 结果时，额外 emit 一条 `ProviderOutbound{Control: ClientASRTranscription{...}}`：

```go
// B14: relay authoritative provider-side ASR transcript back to the client
outbound = append(outbound, ProviderOutbound{
    Control: voiceproto.ClientASRTranscription{
        Type:       voiceproto.TypeClientASRTranscription,
        Transcript: transcript,
        TurnID:     s.activeTurnID,
    },
})
```

## iOS 改动

### `FluentWorkNetworking`

| 文件 | 改动 |
|------|------|
| `WSControlFrame.swift` | 新增 `.clientASRTranscription(text:turnID:)` case 及编解码 |
| `SocketTransportEventMapper.swift` | 新增 `.control(.clientASRTranscription)` → `.serverASRReceived` 映射 |
| `SocketTransport.swift` | 无改动（`.control(WSControlFrame)` 已覆盖新 case） |

### `FluentWorkCore`

| 文件 | 改动 |
|------|------|
| `SpeechSessionEvent.swift` | 新增 `.serverASRReceived(text:turnID:)` 事件 |
| `SpeakingRoomFeature.swift` | 新增 `.serverASRReceived` action 及 reducer 处理 |
| `SpeakingRoomTransportBridge.swift` | 桥接 `SpeakingRoomTransportAction.serverASRReceived` → `SpeakingRoomAction.serverASRReceived` |
| `SpeechSessionMiddleware.swift` | 1. 监听 transport event 中的 `.clientASRTranscription`，dispatch `.serverASRReceived` 并调用 `sendSpeechBoundary(text: serverText)`<br>2. 移除 `speechEnded` 中的本地 Apple Speech ASR 调用 |
| `AppDependencies.swift` | 默认 `clientASRTranscriber` 改为 `ServerRelayASRTranscriber`（no-op，因为 ASR 走 WSS relay） |
| `ServerRelayASRTranscriber.swift` | 新增文件：实现 `ClientASRTranscriber` 协议，直接返回空字符串 |

### `ServerRelayASRTranscriber`（新增）

```swift
public final class ServerRelayASRTranscriber: ClientASRTranscriber {
    public func transcribe(pcm: AsyncStream<Data>) async throws -> String {
        // Drain stream, return empty — ASR is relayed via WSS.
        for await _ in pcm { }
        return ""
    }
}
```

## 回退路径

如 voice gateway 未配置 `VOLC_SPEECH_API_KEY`（Volcengine 凭证），可在 iOS 启动时覆盖 transcriber：

```swift
// 使用本地 Apple Speech
container.clientASRTranscriber.register { AppleSpeechClientASRTranscriber() }
```

## Badge Hit Detection（B12 兼容性）

当 `client.asr.transcription` 帧到达时，`SpeechSessionMiddleware` 立即调用 `sendSpeechBoundary(started: false, turnID: turnID, text: serverText)` 将权威文本发回 backend。

Backend 的 badge emitter 使用此文本进行 B12 phrase block 匹配。此路径不依赖 iOS 本地 ASR 结果。

## 与 B13 的差异

| 方面 | B13（本地 Apple Speech） | B14（服务端 Relay） |
|------|--------------------------|---------------------|
| ASR 准确率 | 依赖设备本地模型，对中文支持一般 | 豆包语音，实时流式，准确率高 |
| iOS API Key | 无 | 无（后端持有） |
| 延迟 | ~300-800ms 本地处理 | 网络 RTT + 豆包处理 |
| 隐私 | 音频留在设备（Apple 管道内） | 音频发往豆包服务器 |
| 网络依赖 | 可离线工作 | 需要网络连接 |
| 回退 | 无（失败则无 transcript） | 可注册 Apple Speech 作为 fallback |
