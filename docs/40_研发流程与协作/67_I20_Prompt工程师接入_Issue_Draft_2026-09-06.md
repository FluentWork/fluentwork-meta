# I20 iOS Prompt 工程师接入（V2.0 收口） — GitHub Issue 草稿

**仓库**：`FluentWork/fluentwork-ios`
**优先级**：P0（W3 第 1 日，**无 backend 阻塞，可立即启动**）
**估时**：0.5 dev-day
**关联文档**：
- `docs/30_技术方案/38_I20收口与全链路架构设计.md`（V1.0 现状）
- `docs/30_技术方案/50_FluentWork全链路架构设计V1_0_2026-09-03.md` §三.5（埋点）
- `docs/40_研发流程与协作/60_iOS_W3_W4_代码层启动包_2026-09-06.md`

---

## 🎯 目标

iOS Prompt 工程师（iOS 端 LLM Prompt 上下文构造）V2.0 收口：

- ✅ **已完成**：基础系统 prompt 注入 + 命中检测构造（V1.x）
- ⏸ **本 Issue 待完成**：
  - **turn 超时兜底**：60s turn 超时后 iOS 主动终止 + 发 `client.turn.abort` 帧给 backend（避免 80+ WARN 级联，见 38_ §1.2）
  - **状态机终态上报**：每次 turn 结束显式上报 `outcome`（`ok` / `timeout` / `user_abandoned`），与 backend `outcome` 字段对齐
  - **Prompt 模板 V2.0**：注入 B7 命中信号后的扩展 prompt 格式（系统 prompt + 最近 8 个 hit block）

---

## 🚧 阻塞条件

- **无 backend 阻塞**——W3 启动首日可立即创建并启动
- 仅依赖 backend WSS 帧协议 V2.0 冻结（53_ §Layer 2 ⏸ 待 W2 末，但本 Issue 仅依赖现有 `client.turn.abort` 帧扩展，本期 backend 已支持）

---

## 📋 实施步骤

### 1. turn 超时兜底（`Sources/SpeechSession/SpeechSessionMachine.swift`）

```swift
// 在 SpeechSessionMachine 添加超时监控
private var turnTimeoutTask: Task<Void, Never>?

func startTurn() {
    state = .listening
    startRecording()
    
    turnTimeoutTask?.cancel()
    turnTimeoutTask = Task { [weak self] in
        try? await Task.sleep(nanoseconds: 60_000_000_000) // 60s
        guard !Task.isCancelled else { return }
        await MainActor.run { self?.handleTurnTimeout() }
    }
}

private func handleTurnTimeout() {
    // 1. 停止录音
    stopRecording()
    
    // 2. 发 abort 帧
    Task {
        try? await webSocketClient.send(frame: ClientTurnAbortFrame(
            sessionID: currentSessionID,
            turnID: currentTurnID,
            outcome: "timeout"
        ))
    }
    
    // 3. 状态机切到 waitingForAIAnswer → 10s 后切 ended
    state = .waitingForAIAnswer
    Task {
        try? await Task.sleep(nanoseconds: 10_000_000_000)
        await MainActor.run {
            if self.state == .waitingForAIAnswer {
                self.state = .ended(outcome: .timeout)
            }
        }
    }
}
```

### 2. outcome 对齐（与 backend `TurnOutcome` 枚举对齐）

```swift
enum TurnOutcome: String, Codable {
    case ok
    case timeout
    case userAbandoned = "user_abandoned"
    case error
}
```

iOS 在以下时机上报 `outcome`：
- 正常结束 → `.ok`
- 超时 → `.timeout`
- 用户主动放弃 → `.userAbandoned`
- 网络断开 → `.error`

### 3. Prompt 模板 V2.0（`Sources/Prompt/SystemPromptBuilder.swift` 新建）

```swift
struct SystemPromptBuilder {
    static func build(
        basePrompt: String,
        recentHits: [RecordedHit],
        userLevel: UserLevel
    ) -> String {
        var prompt = basePrompt
        
        if !recentHits.isEmpty {
            prompt += "\n\n## 最近用户命中过的话术块（用于个性化调整）：\n"
            for hit in recentHits {
                prompt += "- \(hit.intentZh): \(hit.chunkEn)\n"
            }
        }
        
        // V2.0 新增：用户水平调整
        prompt += "\n\n## 用户水平：\(userLevel.rawValue)"
        
        return prompt
    }
}
```

backend 在 WSS 握手成功后通过 `system.prompt.injected` 帧下发基础 prompt（V1.x 已支持）；iOS 在客户端根据 B7 命中 + 用户水平扩展。

### 4. 埋点增强（`Sources/Diagnostics/Tracker.swift`）

```swift
// turn 超时埋点
Tracker.shared.log("turn.timeout", properties: [
    "session_id": sessionID,
    "turn_id": turnID,
    "elapsed_ms": elapsedMs
])

// outcome 分布埋点
Tracker.shared.log("turn.outcome", properties: [
    "outcome": outcome.rawValue,
    "session_id": sessionID
])
```

---

## ✅ 测试矩阵

| ID | 场景 | 预期 |
|---|---|---|
| T-I20-1 | 60s turn 超时 → abort 帧 | backend 收到 `client.turn.abort` + outcome=timeout |
| T-I20-2 | 用户主动放弃 → outcome=user_abandoned | 状态机切 ended，abort 帧带正确 outcome |
| T-I20-3 | 正常结束 → outcome=ok | 与 backend TurnResult 一致 |
| T-I20-4 | 命中块 prompt 注入 | 最近 8 个 hit block 出现在 system prompt |
| T-I20-5 | 用户水平 = advanced | prompt 包含 "用户水平：advanced" |
| T-I20-6 | 用户水平 = beginner | prompt 包含 "用户水平：beginner" |
| T-I20-7 | 10s waitingForAIAnswer 兜底 | 超时后切 ended(outcome=.timeout) |
| T-I20-8 | 超时埋点 | turn.timeout event 上报 |
| T-I20-9 | outcome 分布 | turn.outcome event 4 种 outcome 都触发 |
| T-I20-10 | 与 backend outcome 一致性 | iOS 上报 = backend 记录的 outcome |

---

## 📐 估时拆分

| 步骤 | 估时 |
|---|---|
| 1. turn 超时兜底 | 0.2 dev-day |
| 2. outcome 枚举对齐 + 状态机联动 | 0.1 dev-day |
| 3. Prompt 模板 V2.0 | 0.1 dev-day |
| 4. 埋点 + 单元测试 | 0.1 dev-day |
| **合计** | **0.5 dev-day** |

---

## 🎯 DoD

- [ ] `SpeechSessionMachine.swift` turn 超时兜底逻辑提交
- [ ] `TurnOutcome` 枚举 + 4 种场景下上报
- [ ] `SystemPromptBuilder.swift` 新建 + 最近 8 hits 注入
- [ ] `Tracker.swift` 新增 2 个 event（turn.timeout + turn.outcome）
- [ ] 单元测试：状态机 8 种转换 + outcome 4 种场景
- [ ] 与 backend 集成：发送 `client.turn.abort` 帧 + 收到 `outcome` 字段
- [ ] 38_ §1.2 描述的 4 个缺陷中 #1（`outcome=ok` 误标）修复验证
- [ ] PR 通过 OpenCodeReview + 1 名 iOS maintainer review
- [ ] Issue description 链接本文 + 60_ 启动包 + 38_ §1.2
- [ ] 完成后状态写到 53_ §Layer 2 表格（CLOSED ✅）

---

## 🔗 关联 Issue

- 上游：无（无 backend 阻塞）
- 平行：I21（状态机扩展）同 W3 Day 1 启动
- 下游：所有依赖状态机的 Issue（I14-I19）均受益于本 Issue 的兜底
- 修复合并：38_ §1.2 的 4 个缺陷中 #1 #2（#3 #4 由 backend 处理）
