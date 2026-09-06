# I17 闪测 UI（E1/E4/E5） — GitHub Issue 草稿

**仓库**：`FluentWork/fluentwork-ios`
**优先级**：P0（W4 第 3 日，B22 落地后启动）
**估时**：1.5 dev-day
**关联文档**：
- `docs/30_技术方案/32_FluentWork-iOS App端技术设计文档.md` §八
- `docs/30_技术方案/48_FluentWork_V2_REST接口契约冻结_2026-09-06.md` §1.5
- `docs/40_研发流程与协作/47_D1_D5_开放技术决策备忘录_2026-09-03.md` V1.1 §D-3 + §D-4
- `docs/40_研发流程与协作/58_B22_闪测与A4隐私_Issue_Draft_2026-09-06.md`
- `docs/40_研发流程与协作/60_iOS_W3_W4_代码层启动包_2026-09-06.md`
- 父 Issue：B22（闪测模块，blocked → B22 CLOSED 后启动）

---

## 🎯 目标

实现 PRD §E1/E4/E5 闪测完整 UI：
- **E1 召回闪测**：从调度队列拉 10 题，顺序训练卡流
- **E4 结算页**：本轮正确率 + 错题回放入口 + 「下一轮」按钮
- **E5 推送**：每日 09:00 推送「今日 5 题待复习」（基础本地通知，V1.5 接 APNs）

---

## 🚧 阻塞条件

- **B22（闪测模块 backend）CLOSED**—— `GET /api/v1/drill/round` + `POST /api/v1/drill/judge` 已实施
- iOS 端 `DrillClient` 已存在（如无则从 `AccountAPIClient` 模式抽出 0.3 dev-day）
- D-3 拍板 Ark Mini（判定响应 P90 ≤ 1.5s，iOS 显示「AI 判定中…」loading 不超过 2s）

---

## 📋 实施步骤

### 1. 数据模型（`Sources/Drill/DrillModels.swift`）

```swift
struct DrillRound: Codable {
    let roundID: String
    let items: [DrillItem]
    let startedAt: Date
}

struct DrillItem: Codable, Identifiable {
    var id: String { blockID }
    let blockID: String
    let chunkEn: String
    let intentZh: String
    let audioURL: String
    let timeoutMs: Int
}

struct DrillJudgeResult: Codable {
    let blockID: String
    let semanticMatch: Bool
    let semanticScore: Double
    let pronunciationScore: Double?
    let nextDueAt: Date
    let stateTransition: String
}
```

### 2. 训练卡流 View（`Sources/Drill/DrillTrainView.swift`）

```swift
struct DrillTrainView: View {
    @StateObject var viewModel: DrillTrainViewModel
    @State private var currentIndex = 0
    @State private var userASRText: String = ""
    @State private var isRecording = false
    
    var body: some View {
        VStack {
            // 顶部进度
            ProgressView(value: Double(currentIndex), total: Double(viewModel.round.items.count))
                .padding()
            
            // 当前题目卡
            if let item = viewModel.currentItem {
                DrillCardView(item: item, userASRText: $userASRText)
            }
            
            // AI TTS 预生成播放（用 I15 的 TTSPlayer）
            Button("🔊 听标准发音") {
                AudioPlayer.shared.play(url: viewModel.currentItem?.audioURL)
            }
            
            // 用户录音按钮（HoldToSpeakButton）
            HoldToSpeakButton(
                isRecording: $isRecording,
                onStart: { viewModel.startRecording() },
                onStop: { recordedAudio in
                    Task { await viewModel.judge(audio: recordedAudio, item: viewModel.currentItem!) }
                }
            )
            
            if viewModel.isJudging {
                ProgressView("AI 判定中…")
            }
            
            if let result = viewModel.lastResult {
                ResultToast(result: result)
            }
        }
    }
}
```

### 3. ViewModel（`Sources/Drill/DrillTrainViewModel.swift`）

```swift
@MainActor
final class DrillTrainViewModel: ObservableObject {
    @Published var round: DrillRound
    @Published var currentIndex = 0
    @Published var isJudging = false
    @Published var lastResult: DrillJudgeResult?
    
    var currentItem: DrillItem? {
        round.items.indices.contains(currentIndex) ? round.items[currentIndex] : nil
    }
    
    func startRecording() {
        // 启动 ASR（LiveAudioEngine 已支持）
    }
    
    func judge(audio: Data, item: DrillItem) async {
        isJudging = true
        defer { isJudging = false }
        do {
            let asrText = await ASREngine.shared.recognize(audio: audio)
            let result = try await api.judge(
                roundID: round.roundID,
                blockID: item.blockID,
                audio: audio,
                asrText: asrText
            )
            lastResult = result
            // 2s 后进入下一题
            try? await Task.sleep(nanoseconds: 2_000_000_000)
            lastResult = nil
            currentIndex += 1
        } catch {
            // 错误处理
        }
    }
}
```

### 4. 结算页 View（`Sources/Drill/DrillSummaryView.swift`）

```swift
struct DrillSummaryView: View {
    let results: [DrillJudgeResult]
    
    var accuracy: Double {
        guard !results.isEmpty else { return 0 }
        return Double(results.filter { $0.semanticMatch }.count) / Double(results.count)
    }
    
    var body: some View {
        VStack(spacing: 24) {
            Text("本轮完成 🎉").font(.title)
            Text(String(format: "%.0f%%", accuracy * 100)).font(.system(size: 64))
            Text("\(results.count) 题，正确 \(results.filter { $0.semanticMatch }.count) 题")
            
            NavigationLink("查看错题") {
                DrillWrongListView(results: results.filter { !$0.semanticMatch })
            }
            
            Button("再来一轮") {
                // 触发新一轮 fetch
            }
            .buttonStyle(.borderedProminent)
        }
    }
}
```

### 5. 推送（E5，本地通知）

- `UNUserNotificationCenter` 请求授权
- 每日 09:00 触发本地通知「今日 5 题待复习」（D-4 调度在 backend，iOS 仅本地提醒）
- V1.5 接 APNs（不在本 Issue 范围）

---

## ✅ 测试矩阵

| ID | 场景 | 预期 |
|---|---|---|
| T-I17-1 | 启动闪测 → 拉 10 题 | 训练卡流开始 |
| T-I17-2 | 按住录音 → 松开 → AI 判定 | 2s 内显示 result + 自动下一题 |
| T-I17-3 | 全部 10 题答完 | 跳转结算页 |
| T-I17-4 | 准确率 ≥ 80% 显示鼓励 | 颜色 + 文案 |
| T-I17-5 | 准确率 < 50% 显示再练建议 | 颜色 + 「复习错题」入口 |
| T-I17-6 | 错题列表 | 显示 chunk + 评分 + 改进建议 |
| T-I17-7 | 推送权限请求 | 弹系统授权对话框 |
| T-I17-8 | 推送触发 | 后台状态下通知显示 |
| T-I17-9 | 录音超时（8s） | 自动提交 + 显示「超时」标记 |
| T-I17-10 | 后端 judge 5xx | 重试按钮（最多 2 次） |
| T-I17-11 | 离线状态 | 显示「网络不可用」 |
| T-I17-12 | 中途退出 → 重入 | 进度恢复（state=roundID 持久化到 UserDefaults） |
| T-I17-13 | D-3 判定延迟 ≥ 1.5s | loading 文本「AI 判定中...」+ 不卡顿 |
| T-I17-14 | VoiceOver | 每张卡片可被朗读 + 录音按钮有 hint |

---

## 📐 估时拆分

| 步骤 | 估时 |
|---|---|
| 1. 数据模型 + API Client | 0.2 dev-day |
| 2. DrillTrainView + ViewModel | 0.5 dev-day |
| 3. HoldToSpeakButton 集成 + ASR | 0.2 dev-day |
| 4. DrillSummaryView + 错题列表 | 0.3 dev-day |
| 5. 推送（本地通知） | 0.2 dev-day |
| 6. 测试 + 错误处理 + i18n | 0.1 dev-day |
| **合计** | **1.5 dev-day** |

---

## 🎯 DoD

- [ ] `Sources/Drill/` 整个目录新建（DrillTrainView / SummaryView / ViewModel / Models）
- [ ] `DrillClient` API Client 实现（GET /drill/round + POST /drill/judge）
- [ ] Snapshot 测试（5 种状态：训练中 / 判定中 / 结算页 / 错题列表 / 推送）
- [ ] 端到端冒烟：staging backend + 真实设备走通
- [ ] 推送权限申请时机合理（不抢用户首次启动）
- [ ] D-3 P90 ≤ 1.5s 性能验证
- [ ] i18n + Dark Mode + VoiceOver 通过
- [ ] PR 通过 OpenCodeReview + 1 名 iOS maintainer review
- [ ] Issue description 链接本文 + 60_ 启动包 + 48 §1.5 + 47 §D-3+§D-4 + 58 B22 草稿
- [ ] 完成后状态写到 53_ §Layer 2 表格（CLOSED ✅）

---

## 🔗 关联 Issue

- 上游（阻塞）：B22（闪测模块 backend）
- 依赖：I15 TTSPlayer（共用 Opus 解码器）
- 下游：iOS 推送 V1.5 接 APNs（不在本 Issue 范围）
- 平行：I14 / I15 / I16 / I18 / I19（同 W4 批次）
