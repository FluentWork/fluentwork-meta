# I18 话题卡 UI（H1/H2/H3） — GitHub Issue 草稿

**仓库**：`FluentWork/fluentwork-ios`
**优先级**：P1（W4 第 4 日，B23 落地后启动）
**估时**：1 dev-day
**关联文档**：
- `docs/30_技术方案/32_FluentWork-iOS App端技术设计文档.md` §九
- `docs/30_技术方案/48_FluentWork_V2_REST接口契约冻结_2026-09-06.md` §1.7
- `docs/40_研发流程与协作/60_iOS_W3_W4_代码层启动包_2026-09-06.md`
- 父 Issue：B23（话题卡生成，blocked → B23 CLOSED 后启动）

---

## 🎯 目标

实现 PRD §H1/H2/H3 话题卡完整 UI：
- **H1 话题卡展示**：工作台首页顶部「今日话题卡」入口 → 列表展示
- **H2 进入对话**：点击话题卡 → 跳转 `SpeakingRoomView`（复用现有）
- **H3 聊后打卡**：完成会话后显示打卡弹层，记录 streak

---

## 🚧 阻塞条件

- **B23（话题卡生成 backend）CLOSED**—— `GET /api/v1/topic-cards` + `POST /api/v1/topic-cards/:id/checkin` 已实施

---

## 📋 实施步骤

### 1. 数据模型（`Sources/Topic/TopicModels.swift`）

```swift
struct TopicCard: Codable, Identifiable {
    let cardID: String
    let title: String
    let prompt: String
    let sceneTag: String
    let functionTag: String
    let validUntil: Date
    
    var id: String { cardID }
}

struct CheckinResult: Codable {
    let checkinID: String
    let streakDays: Int
}
```

### 2. 工作台首页入口（`Sources/Home/HomeDashboardView.swift`）

```swift
struct HomeDashboardView: View {
    @StateObject var viewModel = HomeDashboardViewModel()
    
    var body: some View {
        ScrollView {
            VStack(alignment: .leading, spacing: 16) {
                // 话题卡入口
                if !viewModel.topicCards.isEmpty {
                    TopicCardCarousel(cards: viewModel.topicCards)
                }
                
                // 其他模块入口（练习 / 闪测 / 每日一读...）
            }
            .padding()
        }
        .task { await viewModel.load() }
    }
}
```

### 3. `TopicCardCarousel`（横向滚动 + 渐变卡片）

```swift
struct TopicCardCarousel: View {
    let cards: [TopicCard]
    
    var body: some View {
        VStack(alignment: .leading) {
            Text("今日话题卡").font(.headline)
            ScrollView(.horizontal, showsIndicators: false) {
                HStack(spacing: 12) {
                    ForEach(cards) { card in
                        NavigationLink {
                            TopicCardDetailView(card: card)
                        } label: {
                            TopicCardItem(card: card)
                        }
                    }
                }
                .padding(.horizontal)
            }
        }
    }
}

struct TopicCardItem: View {
    let card: TopicCard
    var body: some View {
        VStack(alignment: .leading, spacing: 8) {
            Text(card.sceneTag)
                .font(.caption)
                .padding(.horizontal, 8).padding(.vertical, 4)
                .background(Color.blue.opacity(0.15))
                .clipShape(Capsule())
            Text(card.title).font(.title3.bold()).lineLimit(2)
            Text(card.prompt).font(.body).lineLimit(3)
                .foregroundStyle(.secondary)
        }
        .frame(width: 240, height: 140, alignment: .leading)
        .padding()
        .background(
            LinearGradient(colors: [.blue.opacity(0.1), .purple.opacity(0.1)], startPoint: .topLeading, endPoint: .bottomTrailing)
        )
        .clipShape(RoundedRectangle(cornerRadius: 16))
    }
}
```

### 4. `TopicCardDetailView`（点击进入）

```swift
struct TopicCardDetailView: View {
    let card: TopicCard
    @State private var navigateToSession = false
    
    var body: some View {
        ScrollView {
            VStack(alignment: .leading, spacing: 16) {
                Text(card.title).font(.largeTitle.bold())
                Text(card.prompt).font(.title3)
                HStack {
                    Label(card.sceneTag, systemImage: "tag")
                    Label(card.functionTag, systemImage: "function")
                }
                .font(.caption).foregroundStyle(.secondary)
                
                Spacer().frame(height: 32)
                
                Button {
                    navigateToSession = true
                } label: {
                    Label("开始对话练习", systemImage: "play.fill")
                        .frame(maxWidth: .infinity)
                }
                .buttonStyle(.borderedProminent)
            }
            .padding()
        }
        .navigationDestination(isPresented: $navigateToSession) {
            SpeakingRoomView(seedPrompt: card.prompt, cardID: card.cardID)
        }
    }
}
```

### 5. 聊后打卡弹层（`Sources/Topic/CheckinSheetView.swift`）

会话结束后（session review 页返回时）自动弹出：

```swift
struct CheckinSheetView: View {
    let cardID: String
    @State private var reflection: String = ""
    @State private var isSubmitting = false
    @State private var streakDays: Int?
    @StateObject var viewModel = CheckinViewModel()
    @Environment(\.dismiss) var dismiss
    
    var body: some View {
        NavigationStack {
            VStack(spacing: 20) {
                Text("完成打卡 🎉").font(.title.bold())
                
                if let streak = streakDays {
                    Text("已连续打卡 \(streak) 天")
                        .font(.title2)
                        .foregroundStyle(.orange)
                }
                
                TextField("今天的复盘…（可选）", text: $reflection, axis: .vertical)
                    .lineLimit(3...6)
                    .padding()
                    .background(Color(.systemGroupedBackground))
                    .clipShape(RoundedRectangle(cornerRadius: 12))
                
                Button(action: submit) {
                    if isSubmitting {
                        ProgressView()
                    } else {
                        Text("打卡").frame(maxWidth: .infinity)
                    }
                }
                .buttonStyle(.borderedProminent)
            }
            .padding()
            .toolbar {
                ToolbarItem(placement: .cancellationAction) {
                    Button("跳过") { dismiss() }
                }
            }
        }
    }
    
    private func submit() {
        Task {
            isSubmitting = true
            let result = try? await viewModel.checkin(cardID: cardID, reflection: reflection)
            streakDays = result?.streakDays
            isSubmitting = false
            try? await Task.sleep(nanoseconds: 1_500_000_000)
            dismiss()
        }
    }
}
```

---

## ✅ 测试矩阵

| ID | 场景 | 预期 |
|---|---|---|
| T-I18-1 | 首页加载 → 显示话题卡轮播 | 3-5 张卡片横滑 |
| T-I18-2 | 点击话题卡 → 跳转详情 | 显示完整 prompt |
| T-I18-3 | 点击「开始对话」→ 进入 SpeakingRoom | seed prompt 注入 |
| T-I18-4 | 完成会话 → 自动弹打卡弹层 | 显示在 review 页 |
| T-I18-5 | 输入复盘 → 打卡 | streak +1 显示 |
| T-I18-6 | 跳过打卡 | 弹层 dismiss 但 streak 不变 |
| T-I18-7 | 网络异常 → 打卡失败 | 错误提示 + 重试 |
| T-I18-8 | 话题卡过期（validUntil 已过） | 卡片灰显 + 「已过期」标签 |
| T-I18-9 | 多 streak day 颜色变化 | 7/30/100 天显示不同 badge |
| T-I18-10 | 无话题卡场景 | 首页隐藏话题卡模块 |
| T-I18-11 | VoiceOver | 卡片可朗读 + 打卡按钮可操作 |

---

## 📐 估时拆分

| 步骤 | 估时 |
|---|---|
| 1. 数据模型 + API Client | 0.2 dev-day |
| 2. TopicCardCarousel + TopicCardItem | 0.2 dev-day |
| 3. TopicCardDetailView | 0.2 dev-day |
| 4. CheckinSheetView + streak 展示 | 0.3 dev-day |
| 5. Snapshot + 错误态 + i18n | 0.1 dev-day |
| **合计** | **1 dev-day** |

---

## 🎯 DoD

- [ ] `Sources/Topic/` 整个目录新建
- [ ] `TopicCardsClient` API Client 实现（GET + POST checkin）
- [ ] `HomeDashboardView` 集成话题卡入口
- [ ] Snapshot 测试（5 种状态：carousel / detail / checkin success / skip / 网络失败）
- [ ] 端到端冒烟：staging backend + 真实设备走通
- [ ] i18n + Dark Mode + VoiceOver 通过
- [ ] PR 通过 OpenCodeReview + 1 名 iOS maintainer review
- [ ] Issue description 链接本文 + 60_ 启动包 + 48 §1.7 + 32 §九
- [ ] 完成后状态写到 53_ §Layer 2 表格（CLOSED ✅）

---

## 🔗 关联 Issue

- 上游（阻塞）：B23（话题卡生成 backend）
- 依赖：SpeakingRoomView（已 live）
- 下游：iOS Streak Badge（V1.5 启动）
- 平行：I14 / I15 / I16 / I17 / I19（同 W4 批次）
