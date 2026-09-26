# 问题总清单 · PRD 模块轴

**日期**：2026-09-26
**来源**：`docs/20_产品设计/20_FluentWork产品需求文档PRD.md`（V1.6，MVP 范围）
**问题**：PRD 里列为核心业务逻辑的东西，**还有哪些没完成**。
**方法**：先从代码取事实基线，再拿 PRD 的每条功能 ID 去比对 —— **不从文档里找逻辑漏洞**（`design-code-crosscheck` 核心原则：文档的逻辑可以自洽而事实全错）。

**与既有清单的关系**：`问题总清单.md` 是索引，把条目按「产品与缺陷 / iOS 架构 / backend 架构」三条轴拆。**本文件是第四条轴**，判据不同 —— 既有三轴问「哪里会出事故」，本轴问「这条业务逻辑端到端跑得通吗」。

四份既有文件对本轴**几乎零覆盖**【实测】：`话题卡` / `话题建议` / `登录` / `状态灯` / `重播` / `放弃练习` / `auth/refresh` / `迷你` 在四份文件里全部 **0 命中**。

**本文件不动既有编号。** 条目一律用 PRD 自己的功能 ID（`A1` … `H4`），不新造一套。

---

## 0. 怎么读

**判据只有一条**：这条业务逻辑能不能端到端跑通 —— **客户端有入口 → 接口有路由 → 数据有落点 → 用户看得见结果**。四段缺一段就算没完成。

| 标记 | 含义 |
|---|---|
| ✅ | 端到端已实现 |
| 🟡 | 部分 —— 明确写出缺的是哪一段 |
| ❌ | 未实现 |
| ❓ | 待确认 —— 代码里找不到证据，不下结论 |

**确定性**：【实测】= 命令/输出可复现；【读码】= 源码字面；【推断】= 由前两者推出，未直接观测。

---

## 1. 一句话结论

**说 → 回顾 → 入库 这条主链是通的；围绕它的四块业务在客户端是零或半成品。**

后端比客户端领先得非常多：**闪测（E）与话题建议（H）后端整套已就绪并逐条对着 PRD 实现，客户端是占位符**。反过来，「素材驱动」这条被 PRD 称为护城河的主线，**两端各缺一段，且缺的是输入端**。

按「投入产出比」排，最该先动的是 **2.3（闪测 + 话题建议客户端）** —— 后端已付过学费，客户端是从零。按「不修就一直在骗人」排，最该先动的是 **2.1（素材主线）**。

---

## 2. 三条结构性发现（比逐条清单更值得先看）

### 2.1 「素材驱动」这条主线，两端各缺一段，缺的都在输入端

PRD §三 设计原则 2、§14.3 护城河维度三、B2「Prompt 注入素材提炼 + 场景目标 + 程序员话术规范；≥60% AI 话轮引用素材内容」全都压在这一条上。**它现在是空的**，而且不是「接线松了」：

**事实① —— 客户端从来不送素材**【读码】
`Shared/FluentWorkCore/Services/DefaultSpeechSessionClient.swift:91-95` 与 `:107-111`，**两个** `createSession` 调用点都硬编码 `materialID: nil`：

```swift
created = try await api.createSession(
    accessToken: accessToken,
    materialID: nil,          // ← 两处都是 nil
    sceneType: "standup"      // ← 场景也是硬编码
)
```

**事实② —— 客户端没有创建素材的入口**【读码】
`Shared/FluentWorkNetworking/API/FluentWorkAPI.swift:5-49` 共 15 个 case，**没有 `/materials`**。也没有创建练习弹层（PRD §六 要求它是「从首页到开始说话不超过两次点击」的默认入口）—— 全仓搜 `创建练习` / `CreatePractice` / `2000`（字数上限）/ `迷你` 均无 UI 命中。

**事实③ —— 就算 `material_id` 有值，送到模型的也只是「编号」不是「内容」**【读码】
`internal/voicegateway/provider_volc_duplex.go:1128-1140`：

```go
if material := strings.TrimSpace(start.MaterialID); material != "" {
    parts = append(parts, "素材编号："+material+"。")   // ← 拼的是 UUID
}
```

而素材正文 `Material.Content` 在全仓的消费者**只有提炼 prompt 一个**（`internal/materials/refiner.go:62`：`RefinePrompt(m.Kind, m.Content)`）【读码】。**没有任何代码把素材的提炼结果（主题/术语/讨论点）放进对话的 system prompt。**

**⇒ B2 在生产上不可能达标。** 这一条是设计原则 2 与护城河维度三的**唯一实现路径**，而它两头都断。

**与既有记录的关系**：`问题总清单-产品与缺陷.md` 的 **P1-25 已修**（`session.start` 两端字段名从「零重叠」对齐到网关的名字）。那修的是**中段一个字段名**，修完之后链路的形状是对的 —— 但**没有东西流过去**。这是本仓反复撞到的形状的一个新子型：**判据被修好了，被它驱动的那件事从来没有生产者。**

### 2.2 `/auth/refresh`：客户端在调，后端没有这条路由

**事实① —— 客户端整条链是接上的**【读码】
- `Shared/FluentWorkNetworking/API/FluentWorkAPI.swift:62-63`：`case .refreshToken` → `path = "/auth/refresh"`；
- `Shared/FluentWorkNetworking/SessionAPIClient.swift:90-93`：`refreshToken(_:)` → 上面那个 case；
- `Shared/FluentWorkCore/Services/TokenRefreshCoordinator.swift:122`：401 时调 `sessionAPI.refreshToken(...)`；
- `Shared/FluentWorkCore/Services/AuthenticatedNetworkClient.swift:72`：`handle401Error()` —— **生产调用点**；
- `Shared/FluentWorkCore/Dependencies/AppDependencies.swift:526-533` 的 `networkClient` 构造它，`:544-570` 交给 **corpus / dailyRead / sessionHistory 三个 API 客户端**。

**事实② —— 后端从来没有这条路由**【实测】
`internal/account/http.go:30-36` 只注册四条：`/auth/guest`、`/account/merge`、`/account/data`、`/account/export`。

```
$ grep -rn "auth/refresh" fluentwork-backend/
（零命中，含 api/openapi-v1.yaml）
```

**后果**：访客 access token TTL = **2 小时**（`internal/config/config.go:20` `defaultAccessTokenTTL = 2 * time.Hour`）。超时后：
- **说的房间那条路没事** —— `ensureAccessToken`（`DefaultSpeechSessionClient.swift:294-310`）在过期时**重新签发访客令牌**，压根不走 refresh；
- **语料库 / 每日一读 / 历史三条路会坏** —— 它们走 `AuthenticatedNetworkClient`，401 → 刷新 → **404** → `TokenError.refreshFailed`。

**这不是「刷新逻辑没写」，是两端各写了一半且方向相反**：客户端假设后端有，后端从没打算有。修法有两条 —— 补后端路由，或把客户端这条链摘掉改成重发访客令牌。**这是产品决定**，因为 G1 的注册登录本来就没做，刷新要服务的对象还不存在（见 §3-G1）。

### 2.3 闪测与话题建议：后端整套已就绪，客户端是零

**闪测（模块 E）** 后端逐条对着 PRD 实现了，连 PRD V1.6 才补的两条都在：
- `internal/drill/http.go:31-33`：`GET /drill/round`（E1）、`POST /drill/judge`（E2）、`POST /drill/appeal`（E2 一键申诉）；
- `internal/drill/types.go:106-110` `ASRText` 注释直接写着「the client shows it next to the verdict (E2: 判定前展示 ASR 识别文本)」；
- `:126-129` `Promoted` 注释写着「E4's 已自动化 变化」；
- `internal/drill/service.go` + `config.go`：E3「调度参数服务端可配置」。

**客户端**：`App/FluentWorkHost/HostRootView.swift:17-20`

```swift
flashRoot: {
    Text("闪测（占位）")
        .foregroundStyle(.secondary)
},
```

`FluentWorkAPI` 无 `/drill/*`；`Shared/FluentWorkCore/Navigation/AppNavigation.swift:20-28` 的 `AppRoute` 五个目的地里没有闪测。

**话题建议（模块 H）** 后端同样整套在（`internal/topic/http.go:29-32` + `model.go:53-78`，含 H2 的来源标注 `SourceNote`、H3 的打卡 `CheckedInAt`）；**客户端无 TopicAPIClient、无页面**。

**这是投入产出比最高的一类**：后端已经付过学费，客户端是纯粹的从零，且不需要跨仓契约变更。

---

## 3. 按模块的逐条清单

### 模块 A：素材输入

| ID | PRD 要求 | 现状 | 缺的是哪一段 | 确定性 |
|---|---|---|---|---|
| **A1** | 粘贴 **≤2000 字**；AI 提炼；**5 秒内展示供确认 + 有预期的 loading** | 🟡 | **客户端零**。后端 `POST /materials`（`internal/materials/http.go:27`）在，`kind` 支持 `paste`（`internal/materials/service.go:53-56`）。iOS 无入口、无 `MaterialsAPIClient`。**上限口径已部分收口**（backend `686b74e`）：**单位**修好了（`service.go:69` 改 `utf8.RuneCountInString`；此前 `len()` 是字节 ⇒ 中文 5000 字节 ≈ 1666 字，比 PRD 更严、会 400 掉合法的 2000 字输入）；**值**仍是 **5000 字符**而 PRD 写 **2000 字** —— UI 文档要的是「超限**截断**」不是拒绝，故 2000 是**客户端 UX 上限**，服务端 5000 字符是安全网。⇒ **2000 这个数今天没有任何执行点**（服务端 5000、客户端零）。见工单 T5-a / T5-c | 【读码】 |
| **A2** | 一句话场景描述（≥10 字，≤1 秒不阻塞）；**创建弹层的默认入口** | 🟡 | 后端 `kind=sentence` 支持（同上）；**客户端零**。PRD 称它是「冷启动摩擦最低的路径」，而这条路径不存在 | 【读码】 |
| **A3** | 预置场景卡 Daily Standup，一键进入无需输入 | 🟡 | 场景在客户端**硬编码为字符串** `"standup"`（`DefaultSpeechSessionClient.swift:94` / `:110` / `:129`），**没有卡片 UI，也没有「一键进入」的入口** | 【读码】 |
| **A4** | 输入框常驻隐私声明；设置页「删除我的全部素材」二次确认即时生效 | 🟡 | 后端两条都在（`internal/account/http.go:34-35` `DELETE /account/data` + `POST /account/export`，`internal/account/privacy_service.go` 有级联 wiper）。**客户端：无输入框、设置页无删除入口** | 【读码】 |
| **A5** | **演示素材一键体验完整链路；首次用户的主引导路径** | ❌ | **两端都没有。** 后端唯一的相近物是 `internal/corpus/starter.go` 的起步语料，而它**只在 development 生效**（`cmd/app-server/main.go:132-135` `if cfg.IsDevelopment()`），且它 seed 的是话术块、不是「演示素材 → 完整闭环」 | 【读码】 |

### 模块 B：说（Practice Room）

| ID | PRD 要求 | 现状 | 缺的是哪一段 | 确定性 |
|---|---|---|---|---|
| **B1** | 语音对话；首响 ≤1.5s（P90）；**8-12 回合 + 迷你会话（3-5 回合）** | 🟡 | 语音链路已通。**迷你会话两端都不存在** —— 后端 `CreateRequest` 只有 `material_id` / `scene_type`（`internal/session/types.go:252-255`），无回合数；客户端搜 `迷你` / `mini` / `3-5` / `8-12` 零命中。首响 P90 **未量化验证**（真机没跑过） | 【读码】 |
| **B2** | Prompt 注入素材提炼 + 场景目标；**≥60% AI 话轮引用素材内容** | ❌ | 见 **§2.1**。输入端（无素材）与中间段（只注入编号不注入内容）**两处都缺** ⇒ 生产上不可能达标 | 【读码】 |
| **B3** | 实时转录**浮层**；延迟 ≤1 秒 | 🟡 | 有实时转录，但是**内联视图不是浮层**（`Shared/FluentWorkUI/SpeakingRoom/SpeakingRoomView.swift:650-664`）。延迟无量化 | 【读码】 |
| **B4** | AI 气泡支持**重播音频**、展开文本 | ❌ | 气泡上只有命中徽章按钮（`SpeakingRoomView.swift:600-643`）。全仓搜 `重播` / `replay` 在 UI 层零命中 | 【读码】 |
| **B5** | 完整转录（区分说话人）+ 异步分析；3 秒内可查看 | ✅ | 后端 review worker + `GET /sessions/:id/review`；iOS `ReviewRootView` + `SessionDetailView` | 【读码】 |
| **B6** | **放弃练习**，不进回顾、不入库 | ❌ | 关闭房间一律派发 `endTap`（`App/FluentWorkHost/HostRootView.swift:866-872`），没有第二条路。`SpeakingRoomAction` 里无「放弃会话」动作（`Shared/FluentWorkCore/Architecture/Features/SpeakingRoomFeature.swift:167-219`）；`userAbandoned` 只用于单轮 abort | 【读码】 |
| **B7** | 命中即给轻量徽章 + AI 下一话轮自然确认；**阈值保守化** | ✅ | `internal/voicegateway/badge_emitter.go` + `Shared/FluentWorkUI/BadgeFeedback/BadgeFeedbackOverlay.swift` | 【读码】 |
| **B8** | 卡壳救援：沉默 ≥3s / 未完成句 / **用户点「给我点儿提示」** | 🟡 | 链路在（后端 `internal/conversation/rescue_generator.go`，客户端 `SpeakingRoomView.swift:555-572`）。⚠️ **但 `RescueEnabled` 默认只在 development 开 ⇒ 生产上点按钮是一条静默被忽略的请求**（既有清单已记，需 Tango 拍板） | 【读码】 |

### 模块 C：读（回顾页与每日一读）

| ID | PRD 要求 | 现状 | 缺的是哪一段 | 确定性 |
|---|---|---|---|---|
| **C1** | 转录按话轮展示；**用户话轮可回听录音** | 🟡 | 转录有；**回听没有** —— `ReviewRootView` / `SessionDetailView` 里搜 `回听` / `playback` / `replay` 零命中 | 【读码】 |
| **C2** | **三层评价卡**：目标达成度 / 问题清单（引用原句，按语法/地道度/信息缺失分类）/ 提高建议（≤3 条） | 🟡 | 后端 `internal/review/` 产出 issues / suggestions / comparisons。**客户端取到了却不渲染** —— `Shared/FluentWorkNetworking/API/APIModels.swift:253-254` 明确解码 `issues: [ReviewIssue]` 与 `suggestions: [SuggestionItem]`，但 `ReviewRootView` 的 ViewModel 只留了两个计数（`Shared/FluentWorkUI/Review/ReviewRootView.swift:13-20`），页面渲染的是 `LabeledContent("Issues", value: "\(count)")`（`:155-162`）。**⇒ 数组在 ViewModel 映射那一步被丢掉，原句与分类一个字都没到屏幕上** | 【读码】 |
| **C3** | 双栏对照，差异高亮；3-8 条 | ✅ | `ReviewRootView.swift:175-183`（`You:` / `Better:`）。差异高亮的粒度未核 | 【读码】 |
| **C4** | 历史列表可打开任意回顾页 | ✅ | `SessionHistoryRootView` + `SessionDetailView`（只读重现） | 【读码】 |
| **C5** | 每日一读：150-250 词、AI 朗读、跟读模式、历史归档 | ✅ | 后端 `internal/content/service.go:57,70`（`GetToday` → `ensureTodayRead` 懒生成）；iOS `DailyReadRootView`。跟读评分按 PRD 随 V1.1 后置，不算缺口 | 【读码】 |

### 模块 D：话术块炼化

| ID | PRD 要求 | 现状 | 缺的是哪一段 | 确定性 |
|---|---|---|---|---|
| **D1** | 自动提炼 3-5 个话术块；**优先选自卡壳点** | ✅ | `internal/reviewgen/reviewgen.go` 的 refine；卡壳点经 `StuckEvent` 进输入（`reviewgen.go:29-46`） | 【读码】 |
| **D2** | 卡片**可编辑、可丢弃**、单独入库；一键全部入库 | 🟡 | iOS 回顾页**只有「加入语料库」**（`ReviewFeature.swift:49-52` 的动作集里没有 edit / discard；按钮文案见 `ReviewRootView.swift:219-227`）。语料库侧 `CorpusAPIClient.updateBlock` **存在但零调用点**（`CorpusAction` 里无编辑动作，`CorpusFeature.swift:96-119`）。后端 `batch-accept` 在，但客户端是逐张点击 | 【读码】 |
| **D3** | 自动打场景 / 功能标签；标签体系预设 | ✅ | refine prompt 产出 `scene_tag` / `function_tag`（`internal/materials/refiner_prompt.go`，枚举是闭集） | 【读码】 |

### 模块 E：闪测（提取训练）

| ID | PRD 要求 | 现状 | 缺的是哪一段 | 确定性 |
|---|---|---|---|---|
| **E1** | 中文意图 → 口头作答 → 判定对照；单题限时 5 秒；一轮 10 题 ≤5 分钟 | ❌ | **客户端零**（占位符）。后端 `GET /drill/round` 在 | 【读码】 |
| **E2** | 语义等价判定；**判定前展示 ASR 识别文本 + 一键申诉** | ❌ | **客户端零。** 后端两条都已实现（`JudgeResponse.ASRText` + `POST /drill/appeal`，`internal/drill/types.go:106-110` / `:134-140`） | 【读码】 |
| **E3** | 简化间隔重复；调度参数**服务端可配置** | 🟡 | 后端 ✅（`internal/drill/config.go` + `corpus.Schedule`）。客户端零 ⇒ 端到端不成立 | 【读码】 |
| **E4** | 结算页展示本轮成功率与「已自动化」变化 | ❌ | 后端给了 `Promoted` 字段（`types.go:126-129`）。**无客户端** | 【读码】 |
| **E5** | 常驻底部 Tab；推送提醒可关闭 | 🟡 | Tab 位存在（`AppTab.flashTest`，`AppNavigation.swift:6-18`），**内容是占位**；推送见 §3-推送 | 【读码】 |

### 模块 F：语料库

| ID | PRD 要求 | 现状 | 缺的是哪一段 | 确定性 |
|---|---|---|---|---|
| **F1** | **场景 / 功能双维度筛选** + 关键词搜索 | 🟡 | 关键词搜索 ✅（`CorpusRootView.swift:112-120`）。**双维度筛选无 UI** —— 后端支持（`FluentWorkAPI.swift:144-172` 发 `scene` / `func`），但 `CorpusAction` 里没有筛选动作，`sceneTag` / `functionTag` 只当文本显示 | 【读码】 |
| **F2** | **状态灯**：新入库（灰）/ 训练中（黄）/ 已自动化（绿） | ❌ | 后端字段在（`internal/corpus/types.go:172` `PhraseBlockView.State`）。**iOS 完全不渲染** —— 搜 `状态灯` / `新入库` / `训练中` / `已自动化` 零命中 | 【读码】 |
| **F3** | 收藏与**置顶**筛选 | 🟡 | 收藏 ✅（`CorpusRootView.swift:195-198`）。**置顶没有独立 UI**：`pinned` 参数存在，但 Host 直接把它写成 `isFavorite`（`App/FluentWorkHost/HostRootView.swift:33-35`）⇒ 两个语义被合成一个 | 【读码】 |
| **F4** | 单条删除；**删素材可级联删除衍生话术块** | 🟡 | 单条删除 ✅。级联后端 ✅（`internal/corpus/privacy_wiper.go`）。**客户端没有「删素材」的入口**（与 A4 同一条缺口） | 【读码】 |

### 模块 G：账号与基础

| ID | PRD 要求 | 现状 | 缺的是哪一段 | 确定性 |
|---|---|---|---|---|
| **G1** | 邮箱验证码 + 手机号短信**双通道注册/登录**；**Token 刷新** | ❌ | **注册/登录两端全无**：后端只注册了 `/auth/guest`（`internal/account/http.go:30-36`），`internal/account/service.go:199` 的 `IssueSession` 注释自己写着「Tests and **later login** use this」⇒ 登录是「以后」。iOS 无登录页、无验证码 UI。**Token 刷新见 §2.2** | 【读码】+【实测】 |
| **G2** | 语音偏好（**含 AI 语速 0.8x / 1.0x**）、通知开关、隐私删除入口 | ❌ | iOS 设置页实际只有「关于」+ DEBUG「开发者」两区（`Shared/FluentWorkUI/Settings/SettingsRootView.swift:68-75`）。搜 `语速` / `speechRate` / `0.8x` / `通知开关` 零命中。PRD V1.6 专门为语速记了 D6，**至今没有落点** | 【读码】 |
| **G3** | 演示素材一键体验（与 A5 同一能力，首次用户默认走此路径） | ❌ | 同 **A5** | 【读码】 |
| **G4** | 游客设备级身份；**首次「保存语料」时触发注册**；游客数据**幂等归并** | 🟡 | 后端 ✅（`/auth/guest` + `/account/merge`，`account/service.go:109` `Merge`）。iOS 服务层在（`DefaultSpeechSessionClient.swift:265-268`），reducer 在（`AuthFeature.swift:27,37`），**但没有任何 View 派发它**（仅测试触发）⇒ 归并永远不会发生。且「首次保存语料触发注册」在注册不存在时无从触发 | 【读码】 |

### 模块 H：话题建议

| ID | PRD 要求 | 现状 | 缺的是哪一段 | 确定性 |
|---|---|---|---|---|
| **H1** | 每周 3-5 张话题卡：话题 + 关联场景 + 可调用话术块清单 + 2-3 句开场句式 | ❌ | **客户端零。** 后端 `GET /topic-cards` + `internal/topic/generator.go` 在，`Card.BlockIDs` 就是「可调用话术块清单」 | 【读码】 |
| **H2** | 仅从素材主题与语料标签派生；**每张卡标注来源**；禁止泛话题 | ❌ | 后端 ✅（`Card.SourceNote` + `internal/topic/grounding.go`）。**无客户端** | 【读码】 |
| **H3** | 聊后打卡 + 可选自评顺畅度；数据回传 | ❌ | 后端 ✅（`POST /topic-cards/:id/checkin` + `/stats`）。**无客户端** | 【读码】 |
| **H4** | 典型会议时段前推送当日话题；可关闭 | ❌ | **两端全无**（见下「推送」） | 【读码】 |

### 模块 I：语音评测

**整体后置到 V1.1**（PRD §7.9，V1.3 节奏决策）。**不算 MVP 缺口**，本清单不列。

### 非模块（PRD §六 信息架构 / §9.2 客户端专项）

| 项 | PRD 要求 | 现状 | 证据 | 确定性 |
|---|---|---|---|---|
| **工作台 Tab 1** | 今日入口卡（开始新练习 / 继续上次）+ 每日一读卡 + 话题建议卡 + 练习历史列表 + **轻量数据条**（本周练习次数 / 库存话术块数 / 已自动化数） | 🟡 | 现在是「功能入口」模块列表，`Kind` 只有 5 种：`speakingRoom` / `review` / `dailyRead` / `sessionHistory` / `unsupported`（`Shared/FluentWorkUI/Workbench/WorkbenchHomeView.swift:12-19`）。**今日入口卡、话题建议卡、数据条三样都没有** | 【读码】 |
| **底部 Tab 数** | **3 个主 Tab**（工作台 / 闪测 / 语料库） | 🟡 | 实际 4 个 —— 多一个 `settings`（`AppNavigation.swift:6-18`），且同文件 `:5` 的注释还写着「工作台｜闪测｜语料库」⇒ **注释与代码不一致** | 【读码】 |
| **推送通知** | §9.2：闪测复习 / 每日一读更新 / 话题建议提醒，均可单独关闭 | ❌ | iOS 全仓无 `UNUserNotificationCenter` / `registerForRemoteNotifications` / `aps-environment`；`project.yml:42-50` 只有麦克风与语音识别用途。**三类提醒一条都没有** | 【读码】 |
| **性能红线** | 首响 P90 ≤1.5s；评价 ≤15s；冷启动 ≤2.5s | ❓ | 无量化基线。真机一次端到端都没跑过（与既有清单「h8 端到端一次没跑」同源） | 【推断】 |

---

## 4. 汇总

### 4.1 计数

| 判定 | 条数 | 条目 |
|---|---|---|
| ✅ 已实现 | 7 | B5 · B7 · C3 · C4 · C5 · D1 · D3 |
| 🟡 部分 | 14 | A1 A2 A3 A4 · B1 B3 B8 · C1 C2 · D2 · E3 E5 · F1 F3 F4 · G4 · 工作台 · Tab 数 |
| ❌ 未实现 | 13 | A5 · B2 B4 B6 · E1 E2 E4 · F2 · G1 G2 G3 · H1 H2 H3 H4 · 推送 |

（部分条目跨格，计数按主判定）

### 4.2 按「能不能立刻动手」排

**① 不需要新决定、可以立刻做**（纯客户端补件，后端已就绪）

- **E1 / E2 / E4 + E5 内容**：闪测客户端整块。后端 round / judge / appeal / promoted 全在。
- **H1 / H2 / H3**：话题建议客户端整块。后端生成、来源标注、打卡、统计全在。
- **F1 双维度筛选**、**F2 状态灯**、**F3 置顶独立化**：三个小件，接口参数与字段都已存在。
- **C2 三层评价渲染**：后端数据在，客户端只显示计数。
- **B4 重播**、**B6 放弃练习**：两个独立小件。
- **D2 编辑 / 丢弃**：`CorpusAPIClient.updateBlock` 已在，缺的是动作与 UI。
- **Tab 注释与代码不一致**（`AppNavigation.swift:5`）：一行的事。
- **A5 / G3 演示素材**：需要先定「演示素材长什么样」，但那是内容决定不是架构决定。

**② 需要 Tango 拍板**

- **§2.1 素材主线**：先做哪一半？三种形状 —— (a) 客户端补创建入口 + 后端把素材**内容**注入 prompt；(b) 只补客户端入口，先让 `material_id` 有值（链路形状对，模型仍只看编号）；(c) 先承认这条线 MVP 不做，把 B2 的验收标准从 PRD 里划掉。**我倾向 (a)**：只做 (b) 会造出「字段有值、模型看不见」的第二层假象。
- **§2.2 `/auth/refresh`**：补后端路由，还是摘掉客户端这条链？**取决于 G1 做不做** —— 注册登录不做，刷新没有服务对象，摘掉更诚实。
- **G1 注册 / 登录**：邮箱 + 手机号双通道要接短信服务商（成本、资质、合规），不是纯工程决定。
- **G2 AI 语速 0.8x / 1.0x**：PRD V1.6 的 D6 明确说「成本极低、是可懂度开关」，但**它是合成参数，改的是 TTS 请求** ⇒ 跨仓（客户端 → `/tts/synthesize` 或网关）。要先定它走哪条路。
- **B1 迷你会话**：需要定「3-5 回合」由谁判定（客户端倒数 / 服务端计轮 / prompt 约束）。`CreateRequest` 现在没有这个字段。
- **工作台 Tab 1 的形态**：PRD §六 的结构与现状的「功能入口列表」是两个物种。另注意既有清单 **P0-13** 提的「房间入口应是会话列表（DeepSeek 那样）」会**取代** PRD 里「工作台放今日入口卡」那个形状 —— 这两条要先对齐再动手。

**③ 只能靠真机 / 需要外部资源**

- 首响 P90、评价 ≤15s、冷启动 ≤2.5s：**无量化基线**，真机一次端到端都没跑过。
- 推送：需要 APNs 证书与推送服务端，属基础设施。
- 短信通道：需要服务商。

---

## 5. 本轴**不**覆盖的东西

写下来是为了让下一个人不必再判一次：

- **模块 I 语音评测（I1-I4）**：PRD 自己后置到 V1.1 / V2.0，不算 MVP 缺口。
- **订阅与商业化（§十五）**：PRD 自己后置到 V1.1，MVP 期全免费。
- **既有三轴已登记的条目**（`iOS-S0-x` / `BE-S0-x` / `P0-x` / `P1-x` / `F-x`）：本轴不重复登记，只在**同一处缺口有交叉时**给出行内引用（如 B8 的 `RescueEnabled`、§2.1 的 P1-25）。
- **代码卫生类问题**（命名、死代码、注释过期）：那是既有三轴的活。

---

## 6. 一个值得记下来的观察

本轴查出的 13 条 ❌ 里，**有 5 条是「后端完整、客户端是零或占位」**（E1/E2/E4、H1/H2/H3）。

这不是「客户端进度慢」，而是**判据的选择本身会掩盖它**：后端每一条都有测试、门禁是绿的、`dev-check.sh` 七步全过 —— 因为**测试测的是服务端这一半**。这与本仓反复撞到的形状同源（「它测的是哪一半？」），只是换了个位置：**这次不是一条测试守错了对象，是两端的验收标准各自完整、而中间那条缝没有人负责。**

判据可以很简单：**问「这个功能在真机上有没有一个入口」，而不是问「这个功能的代码写完了没有」。**
