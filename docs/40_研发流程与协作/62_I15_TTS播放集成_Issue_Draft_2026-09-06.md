# I15 说的房间 TTS 播放集成 — GitHub Issue 草稿

**仓库**：`FluentWork/fluentwork-ios`
**优先级**：P0（W3 第 3 日，B17 落地后启动）
**估时**：1 dev-day
**关联文档**：
- `docs/30_技术方案/32_FluentWork-iOS App端技术设计文档.md` §六
- `docs/30_技术方案/37_FluentWork-B14_Client_ASR_Relay_Architecture.md`
- `docs/30_技术方案/38_I20收口与全链路架构设计.md` §六
- `docs/30_技术方案/48_FluentWork_V2_REST接口契约冻结_2026-09-06.md` §1.6.1
- `docs/40_研发流程与协作/47_D1_D5_开放技术决策备忘录_2026-09-03.md` V1.1 §D-2
- `docs/40_研发流程与协作/60_iOS_W3_W4_代码层启动包_2026-09-06.md`
- 父 Issue：B17（TTS Provider，blocked → B17 CLOSED 后启动）

---

## 🎯 目标

实现 iOS 端 TTS 音频流接收 + 播放 + 用户打断兼容（PRD F2 增强）：
- 收到 voicegateway WSS 推送的 Opus 帧 → 立即解码播放
- 用户按下 HoldToSpeakButton → 中断 AI 播放 + 立刻进入 listening 状态
- 流式首字 P90 ≤ 400ms（接收第一帧到开始出声）

---

## 🚧 阻塞条件

- **B17（TTS Provider backend）CLOSED**—— voicegateway 已支持流式 Opus TTS 推送
- iOS 端 `LiveAudioEngine` 已支持 PCM 输入/输出（WSS 已 live）

---

## 📋 实施步骤

### 1. TTS 音频接收（`Sources/Audio/TTSPlayer.swift` 新建）

```swift
final class TTSPlayer {
    private let audioEngine = AVAudioEngine()
    private let playerNode = AVAudioPlayerNode()
    private var converter: AVAudioConverter?  // Opus → PCM
    
    init() throws {
        audioEngine.attach(playerNode)
        let format = audioEngine.outputNode.inputFormat(forBus: 0)
        audioEngine.connect(playerNode, to: audioEngine.mainMixerNode, format: format)
        try audioEngine.start()
    }
    
    func enqueueOpusFrame(_ opusData: Data) {
        // Opus 解码 → PCM buffer → playerNode.scheduleBuffer
        guard let pcmBuffer = OpusDecoder.decode(opusData) else { return }
        playerNode.scheduleBuffer(pcmBuffer, at: nil, options: .interrupts, completionHandler: nil)
        if !playerNode.isPlaying {
            playerNode.play()
        }
    }
    
    func interrupt() {
        playerNode.stop()
    }
}
```

### 2. WSS 帧分发增强（`Sources/Networking/WSFrameDispatcher.swift`）

```swift
case let frame as AITTSStartFrame:
    ttsPlayer.interrupt()  // 重新播放前清空旧 buffer
    currentTTSSessionID = frame.sessionID
case let frame as AITTSAudioFrame:
    ttsPlayer.enqueueOpusFrame(frame.opusData)  // 流式推送
case let frame as AITTSEndFrame:
    // 标记 session 结束，不立即停止（等播放器自然排空）
    break
```

### 3. 用户打断兼容（`Sources/SpeechSession/SpeechSessionMachine.swift`）

按住 HoldToSpeakButton 时：
1. 立即调用 `ttsPlayer.interrupt()`
2. 发送 `client.audio.start` 帧给 backend
3. 状态机从 `speaking` 切到 `listening`（**含 I21 的 `waitingForAIAnswer` 中间态降级直接跳 listening**）

### 4. 音色切换 UI（Setting 入口预留）

- Setting → Voice Selection → 4 个音色（D-2 拍板）
- 选完后写入 UserDefaults + 下次创建 session 时通过 `voice_id` 传给 backend
- 本 Issue 仅落 UI + UserDefaults；实际传入 backend 由 B17 后的下一个 sprint 实施

---

## ✅ 测试矩阵

| ID | 场景 | 预期 |
|---|---|---|
| T-I15-1 | 收到首帧 Opus → 出声 | P90 ≤ 400ms |
| T-I15-2 | 流式多帧 → 连续播放 | 无卡顿、无重复 |
| T-I15-3 | 用户按住打断 → AI 静音 | 100ms 内静音 |
| T-I15-4 | 打断后用户说话 → 音频正常 | listening 状态切换正常 |
| T-I15-5 | 网络抖动（丢 1 帧） | 播放器容忍（Opus FEC 内置） |
| T-I15-6 | 后台切换 → 音频自动暂停 | AVAudioSession 通知触发 pause |
| T-I15-7 | 多 session 并发（理论上不会） | 单元测试 mock |
| T-I15-8 | 音色切换 UI | UserDefaults 持久化，下次启动保留 |
| T-I15-9 | 蓝牙耳机断开 | 切回内置扬声器，不卡顿 |
| T-I15-10 | VoiceOver「AI 正在说话」 | 状态改变触发 VoiceOver 提示 |

---

## 📐 估时拆分

| 步骤 | 估时 |
|---|---|
| 1. TTSPlayer + Opus 解码 | 0.4 dev-day |
| 2. WSS 帧分发增强 | 0.2 dev-day |
| 3. 打断兼容 + 状态机联动 | 0.2 dev-day |
| 4. 音色选择 UI + 持久化 | 0.2 dev-day |
| **合计** | **1 dev-day** |

---

## 🎯 DoD

- [ ] `Sources/Audio/TTSPlayer.swift` 提交
- [ ] `WSFrameDispatcher` 新增 `ai.tts.audio` 帧处理
- [ ] 用户打断 ≤ 100ms 静音（性能测试）
- [ ] 4 个音色选择 UI + UserDefaults 持久化
- [ ] Snapshot 测试（音色选择 4 个状态）
- [ ] 端到端冒烟：staging backend + 真实设备走通
- [ ] 蓝牙 / 后台 / VoiceOver 兼容
- [ ] PR 通过 OpenCodeReview + 1 名 iOS maintainer review
- [ ] Issue description 链接本文 + 60_ 启动包 + 32 §六 + 38 §六 + 48 §1.6.1 + 47 §D-2
- [ ] 完成后状态写到 53_ §Layer 2 表格（CLOSED ✅）

---

## 🔗 关联 Issue

- 上游（阻塞）：B17（TTS Provider）
- 下游：I17 闪测 UI（共用 Opus 解码器，复用 `TTSPlayer` 或抽出 `AudioCodec`）
- 平行：I14 / I16 / I17 / I18 / I19（同 W4 批次）
- 测试配套：voicegateway TTS 帧格式需对齐 38_ §六
