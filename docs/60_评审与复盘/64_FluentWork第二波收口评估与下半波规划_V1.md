# FluentWork 第二波收口评估与下半波规划 V1

**版本**：V1.0
**日期**：2026-09-03
**定位**：用户对第一波 + 第二波真实落地做了一次集中评估后的收口记录，把"已落地但未走完 app 验收 / 已写代码但需补质量门禁 / 范围外继续挂账"三件事用统一口径写清，并给出第三波（质量门禁 + 真实语音 + 验收）的入口。
**上游依据**：
- `53_FluentWork第一波与第二波App可测试问题验收清单.md`
- `62_FluentWork第一波关闭记录.md`
- `63_FluentWork第二波阶段进度汇报_2026-08-31.md`
- `61_FluentWork第二波关键难点进度与缺口分析.md`
- 本次代码扫描（iOS / backend / docs）

---

## 一、结论句（最简版）

> **第二波主功能面 12 项里 11 项已并入 `main`，1 项（I11）代码已写但未在真链路端到端浮现过**。用户实测"麦克风 → 后端 → 一部分反馈"链路已通（B14 ClientASR relay + VolcDuplexProvider），这是从 stub 切真路径的关键一步；下一步不是继续开新功能，而是**收口**：(1) 验证 I11 / B12 真徽章浮现；(2) 用 Volcengine 真实凭证替换 dev 默认 mock；(3) 把 B15 真标注样本从合成扩到首批 20 条；(4) 按 `53` 号验收清单在 app 里逐项勾选。

---

## 二、本次评估的代码事实

### 2.1 主链路已接通的证据

| 现象 | 代码证据 |
|---|---|
| iOS 默认走真实音频采集 | `fluentwork-ios/Shared/FluentWorkCore/Dependencies/AppDependencies.swift` `audioEngine` factory 在生产路径返回 `LiveAudioEngine(decoder: RawPCM16FrameDecoder())`，`PlaceholderAudioEngine` 仅 XCTest 用 |
| 麦克风采集后 PCM 经 WSS 发出 | `LiveAudioEngine.swift` + `SpeechSessionMiddleware.swift` 已完成麦克风 → PCM 缓冲 → `sendAudioPCM` → WSS |
| 后端 WSS 接收并 relay 到火山 | `internal/voicegateway/handler.go` `handleAudio` → `rt.provider.HandleClientAudio` → `VolcDuplexProvider` (`provider_volc_duplex.go`) |
| 火山 ASR 回灌 iOS | `provider_volc_duplex.go` 在有 Doubao ASR 结果时 emit `ProviderOutbound{Control: ClientASRTranscription{...}}` → `handler.go` 写 WSS → iOS `SocketTransportEventMapper` 收到 `.control(.clientASRTranscription)` → 派发 `.serverASRReceived(text:turnID:)` |
| iOS 把 server ASR 写进 liveTranscript | `AppReducer` 走 `SpeakingRoomReducer.serverASRReceived` → `state.liveTranscript = text` |

### 2.2 I11 / B12 真链路当前的真实状态

| 面 | 代码状态 | 是否真链路浮现过 |
|---|---|---|
| `feedback.badge` 帧协议 | `wss-control-frames-v1.json` 已定义；iOS `WSControlFrame` 编解码有测试 | 协议层 ✅ |
| 后端 `BadgeEmitter` 实现 | `internal/voicegateway/badge_emitter.go`；5 个原 bug 已修复（dedupe 反向 / ctx 取消 / 排序 / 测试生命周期）；`handler.go` 在 `user.speech.end` 触发 `BadgeEmitter.Emit` | 单测 ✅，集成测 ✅ |
| iOS `BadgeFeedbackReducer` | `architecture/Features/BadgeFeedbackFeature.swift`，接 pullback 到 AppReducer，含去重 + 限时显示 + 容量上限 | 单测 ✅ |
| iOS `BadgeFeedbackOverlay` | `Shared/FluentWorkUI/BadgeFeedback/BadgeFeedbackOverlay.swift`，已在 SpeakingRoom 顶部 `TimelineView` 中渲染 | 代码 ✅，**未在真链路浮现过** |
| 真实徽章端到端浮现 | 需要 B12 真链路 + 语料库有命中目标 | **未验证**（语料库空；provider 默认 mock） |

> **判断**：I11 / B12 已具备代码层面的全部前提条件，**唯一缺的是真实的命中目标（语料库数据）+ 真实火山 ASR 转写**。dev 环境默认走 mock provider + 空语料库，是 8/31 阶段汇报指出的同样问题。

---

## 三、本轮已落地的两项新工作

### 3.1 A1 — iOS SpeakingRoom DEBUG 注入测试徽章

**变更文件**：
- `fluentwork-ios/Shared/FluentWorkUI/SpeakingRoom/SpeakingRoomView.swift`：新增可选回调 `onDebugBadgeInjected` + DEBUG-only 底部 footer，点击循环触发四种 tier（unknown / soft / highlight / celebrate），可即时验证 `BadgeFeedbackOverlay` 是否渲染正确。
- `fluentwork-ios/App/FluentWorkHost/HostRootView.swift`：把回调接到 `store.dispatch(.speakingRoom(.badgeHit(...)))`，与后端 `feedback.badge` 帧落入的同一 action 路径。

**作用**：
- 不依赖 B12 真命中即可在 dev build 中验证 I11 overlay 浮现是否正常
- 通过 tier 切换可肉眼对比四种视觉权重
- 通过统计注入次数（×N）观察 dedupe / 容量上限行为

**测试**：239 项 swift test 全绿（含 `SpeechSessionMiddleware B14 Integration` 套件）

### 3.2 A2 — backend dev corpus seed 工具

**新增文件**：
- `fluentwork-backend/cmd/corpus-seed/main.go`：CLI 工具，issue guest → batch-accept 10 条预置职场英语 phrase block → list 验证
- `fluentwork-backend/scripts/corpus-seed.sh`：wrapper，默认 target `http://127.0.0.1:8080`

**预置 10 条 seed 内容**（节选）：
- "I'm blocked on the API review." / `standup.report`
- "Let's wrap up." / `review.summarize`
- "Let's ship it." / `review.commit`
- "Bottom line: we'll ship next Tuesday." / `review.summarize`
- ...

**作用**：dev 环境一键预置语料库，使 B12 真链路在本地有命中目标；idempotent，可反复跑。

---

## 四、剩余待办（按"建议用户下一步先做"排序）

### 第一优先级 — 真链路端到端验收（与 `53` 号验收清单对齐）

1. **I11 + B12 真链路浮现**
   - 路径：把 dev backend 跑起来 → `./scripts/corpus-seed.sh` 预置语料 → 配置火山凭证（若可用）/ 否则保留 mock ASR → iOS dev build 走真说话 → 观察 `feedback.badge` 帧是否到达 iOS → overlay 是否浮现
   - fallback 验证：若无火山凭证，至少在 dev build 内用 A1 的 DEBUG 注入按钮验证 overlay 视觉

2. **Volcengine 凭证 + provider 切换**
   - 当前 `provider_factory.go` 默认 mock；需在 `.env` 设 `VOICE_PROVIDER=volc_duplex` + 火山 AK/SK/AppID 才能走真链路
   - 商务前置仍未闭环（meta #12 PREREQ），凭证开通是技术开工的前提

3. **I12 三项量化验证**
   - 首响 P90 ≤ 1.5s
   - 打断 ≤ 200ms
   - 三级降级切换成功率
   - 当前 `LiveAudioEngine` + `MockVoiceProvider` / `VolcDuplexProvider` 切换逻辑已具备，缺活体测量

### 第二优先级 — 质量门禁

4. **B15 真标注样本首批 20 条**
   - 当前 `eval/offline/samples/wave2-synth-v1.json` 合成样本过门
   - 真标注样本需覆盖：评价引用原句 / 炼化三元组完整 / 锚点落转录
   - 这是 B8 提测前必须跑通的门禁

5. **I10 真机 lock screen 验证**
   - `docs/09_I10_smoke_test_runbook.md` 已具备 9 项 P0
   - 缺：iPhone 15/16 真机实测（锁屏持续 / 切后台 / 来电中断 / 网络 fallback / K 不出分）

6. **I9 弱网本地优先实测**
   - 飞行模式下浏览语料库、出网后 replay Outbox、tombstone 不残留
   - 缺：活体 runbook

### 第三优先级 — 流程纪律

7. **`53` 号验收清单逐项勾选**
   - §4 第一波 14 项 + 第二波 13 项在 app 中真点
   - 每项标 `[ ] 通过 / [ ] 不通过`，不通过项落到 issue

8. **meta #12 商务前置收口**
   - 火山供应商三项（额度 / 协议 / 并发）任一项未闭环，B13 全量生产化就会被卡

### 不在本轮范围（按 `53` §5 口径挂 backlog）

- W3+ 范围：`I1–I4` 发音评测（V1.1）、`E1–E5` 闪测、`H1–H4` 话题建议、`C4` 创建练习弹层、`A1–A2` 素材模块、`billing`、APNs 推送

---

## 五、给下一轮（第三波）的入口建议

> **不要继续开新功能，先把第二波已落地的 11 项在 app 里真走一遍**，然后按"真链路端到端 / 质量门禁 / 流程纪律"三层顺序推进。

具体来说：

1. 立即：把这次新增的 A1（iOS DEBUG 注入）和 A2（backend corpus seed）作为 dev 工作流固定下来，写进 `53` 号验收清单的"准备阶段"
2. 本周：解决 Volcengine 凭证 + I11 真链路浮现，跑 `53` §2.1 / §2.4 / §2.5 的 P0 case
3. 下周：补 B15 真样本 + I10 真机锁屏 + I9 弱网，写活体 runbook
4. 再下周：按 `53` §4 checklist 收口，没过的不糊弄，不阻塞的挂 backlog

---

## 六、对接关系

| 本文档 | 上游 | 下游 |
|---|---|---|
| 评估层（V1） | `53` 验收清单、`63` 8/31 进度汇报、`61` 难点分析 | 第三波开发 issue / PR、`meta #12` 商务前置 |

---

## 七、最终口径

> **第二波主功能面代码已基本落地，缺的是真链路浮现 + 质量门禁 + app 内真点验收**。这次新增的 A1 + A2 是为了让用户在 dev 环境下能立即验证（不依赖火山凭证），B12 / I11 / B15 / I10 / I9 这五项是收口的关键路径。**不要把"第二波没完"误读成"还要写很多代码"，真正的下一步是用已有的代码走完端到端验收。**
