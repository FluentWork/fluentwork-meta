# PRD 核心业务逻辑落地工单

**建立日期**：2026-09-26
**上游**：[`问题总清单-PRD模块轴.md`](./问题总清单-PRD模块轴.md)（那里是**缺什么**，这里是**怎么补**）
**范围口径**：**UI 先不考虑，核心业务逻辑优先。** 本工单里的票全部是**服务端 + 契约 + 数据链路**，不包含任何界面形态工作。
**命名**：活文档，**不带日期后缀** —— 理由见 `问题总清单.md` §0（带日期的活文档正是它被孤立的机制）。

---

## 0. 这份工单怎么读

拆成两段，**不混在一起**：

| 段 | 性质 | 回答 | 产出 |
|---|---|---|---|
| **决策票 D** | 待决问题 | 我们到底要建什么 | 结论 |
| **执行票 T** | 垂直切片 | 怎么建 | 可验证的代码 |

**决策票不关完就开执行票 = 在流沙上砌墙。** 但为了不空转：**每张决策票都带一个"未拍板前按这个写"的默认**，照默认开工，结论变了再改。

**每条执行票必须回答四个问题**：现状证据（`文件:行`）→ 落点 → 改动形状 → **完成判据（以及它凭什么会红）**。

**确定性**：【实测】= 命令可复现；【读码】= 源码字面；【推断】= 由前两者推出。

**开工纪律**（沿用本仓既有）：一次提交一张票 · `git add <显式路径>` · 跨仓串行 **infra → backend → iOS → meta** · 推送前先问 · **先写判据并确认它红，再改代码**。

---

## 1. 决策票

### D1 — B2 注入「素材正文」还是「提炼产物」？

**为什么必须定**：PRD A1 要求 AI 提炼**主题/术语/讨论点**，而现有 refine 只产**话术块**（`internal/materials/refiner_prompt.go` 的 JSON schema 只有 `intent_zh` / `expression_en` / `scene_tag` / `function_tag`）【读码】。B2 要注入的是哪一份，决定 T1 是小时级还是天级。

| 选项 | 代价 | 效果 |
|---|---|---|
| **a. 注入素材正文**（截断 N 字） | 零迁移、零 prompt 改动 | 「AI 话轮引用素材内容」**第一次成立** |
| **b. 先补提炼产物再注入** | 加列/新表 + 迁移 + 改 refine prompt + 重跑 | 更贴题，但**不是"能不能引用"的前提** |

**我建议 a。** 理由：B2 的验收标准是「≥60% AI 话轮引用素材内容」—— 注入正文就能满足它；提炼产物是"引用得更准"的优化。而且 a 可以立刻做，b 要先付一笔迁移与重新提炼的成本。
**未拍板前按 a 写。** b 留成独立票（见 T5-b）。

### D2 — 「客户端数据层」算不算 UI？

**为什么必须定**：`问题总清单-PRD模块轴.md` 里 13 条 ❌ 中，**5 条（E1/E2/E4、H1/H2/H3）缺的正是客户端**，而服务端是完整的。如果"数据层"也算 UI，这 5 条在本轮排期里**永远进不来** —— 服务端等客户端、客户端等 UI，死锁。

**原建议**：把 **API client + reducer + middleware + 导航目的地** 划入业务逻辑；**View 才是 UI**。

**✅ 拍板（2026-09-26）：按上面这条采纳。** API client / 数据层 / 中间件 / 导航目的地 = 业务逻辑，本轮可排；只有 View 才算 UI。
⇒ T1-c（客户端把 `material_id` 传上来）**从此不再是「等 D2」**。

**但拍这条并不能让 T1/T4 的价值落地 —— 我实测了它后面的真正断点：**

- iOS 侧**根本没有 materials 客户端**：`Shared/` 里搜 `materials` 只在
  `SessionHistoryModels.swift` 命中一处字段名，没有任何创建素材的调用。
- 两处 `createSession` 硬编码 `materialID: nil`（`DefaultSpeechSessionClient.swift:91-95` / `:107-111`）。
- 所以 T1-c 能补上的是**「数据层可以把 `material_id` 发出去」**，而**没有任何用户能触发它** ——
  创建素材的入口是**粘贴框**，也就是 **T5-c，当前「UI 未设计」**。

⇒ **T5-c 才是 T1 / T4 端到端价值的真正前置**，不是 D2。这条比「D2 拍不拍」更值得记：
**D2 解的是「这类工作算不算 UI」，T5-c 解的是「这条链路有没有入口」—— 两件事。**

### D3 — G1 注册登录做不做？

**为什么必须定**：决定 T2 走哪条路。注册登录要短信服务商（资质 + 成本 + 合规），**不是工程决定**。

**我建议**：G1 本轮不做 ⇒ ~~**T2 走 b（摘链）**~~。理由：补一条没有服务对象的 `/auth/refresh`，等于亲手造出第二处「看起来有保护、实际没有」—— 而本仓已经撞到十次以上。
~~**未拍板前按 b 写。**~~

**✅ 实际拍板（2026-09-26）：走了 a（补路由），与上面这条建议相反。**
记录这个分歧，因为它不是笔误：**「没有服务对象」这个前提不成立** —— 客户端本来就在调
`/auth/refresh`（`AuthenticatedNetworkClient.swift:72`，生产上由 corpus / dailyRead /
sessionHistory 三个客户端共用），所以**补路由不是造一个空端点，是把一条已经存在的死路接上**。
D3 真正要拦的是 G1（注册 / 登录 / 短信 / 邮箱验证码），T2-a 不碰这些。
**⇒ 教训：建议里的理由要连同它的前提一起写。** 这里前提（"没有服务对象"）是可实测的，
一旦实测为假，建议就该被推翻 —— 但原文只留了结论，读者（包括我）会照结论走。

⇒ **T2 已按 a 完成**（服务端 `4035f12` / 客户端 `be85ca6`）。

### D4 — 迷你会话的回合上限由谁定、定几轮？

**为什么必须定**：PRD 只说「3-5 回合」，没说谁数。`CreateRequest` 现在没有这个字段（`internal/session/types.go:252-255`），**网关全仓没有轮次概念**（`turnLimit|maxTurns|turnCount` 零命中）【实测】。

**我建议**：
- **计轮必须在服务端。** prompt-only（在 system prompt 里写"这是 3-5 回合"）是这个仓最贵的那类假保护 —— 模型不遵守时**没有任何东西会红**。
- 参数走 `session_length` 枚举（`standard` / `mini`），上限**服务端可配**，照 `DRILL_ROUND_SIZE` 的先例（`internal/config/config.go:34,95,146`）—— PRD §9.1 自己要求「调度参数与生成阈值全部服务端可配」。
- `mini` 上限默认 **5**（PRD 区间上沿，宁多不少：早收尾比晚收尾更伤对话感）。

**未拍板前按此写。**

---

## 2. 执行票

### T1 — 素材上下文注入对话（B2）｜最高价值 ✅

**目标**：会话打开时，把素材内容送进模型的 system prompt。

**现状（证据，改动前）**
- 网关只拼**编号**：`internal/voicegateway/provider_volc_duplex.go:1134` —— `parts = append(parts, "素材编号："+material+"。")`【读码】
- 素材正文在全仓的**唯一**消费者是提炼 prompt：`internal/materials/refiner.go:62`【读码】
- **落点已经存在，不需要新端点、不需要改协议**：`POST /internal/v1/sessions/activate`（`internal/session/internal_http.go:30`），网关在 `openSession` 里必调一次（`internal/voicegateway/handler_control.go:288`）【读码】
- app-server **自己知道** `material_id`：`internal/session/types.go:57`（`Session.MaterialID`）【读码】

**已落地（backend `89cd2df`）**
1. `session`：`ActivateResponse` 加 `material_context`；新增 `MaterialSource` seam + `SetMaterialSource`；`Activate` 从**会话记录**取 `material_id` → 取正文 → `capRunes` 到 2000 字；**查不到只记 WARN、不失败**（照 `ContinuationContext` 的先例）。
2. `materials.SessionMaterialSource` 适配器（照 `materials.PrivacyWiper` 的先例），`cmd/app-server` 在 materialSvc 建好后接线。
3. `voicegateway`：`SessionLifecycle.Activate` 由 `error` 改成返回 `ActivateResult{MaterialContext}`（**网关侧自有类型** —— 这个包不依赖 app-server 的包）；`openSession` 把结果存进 `rt.session`（**与 `rt.continuation` 同址、同生命周期**）。
4. `VoiceProviderSession.Start` 的第二个参数由 `[]ContinuationTurn` **收敛成 `SessionContext{Continuation, Material}`**（15 个实现同步）。
5. `instructionsForSessionStart` 用素材正文替换「素材编号」—— **替换，不是并存**。

**完成判据与证据**
- 三个新文件 + 一处扩写，共 10 条判据：`internal/session/activate_material_test.go`（5）、`internal/voicegateway/handler_material_context_test.go`（2，走真 WSS 握手）、`handler_upstream_recovery_test.go` 加 1 条重开路径。
- **变异 7 次，各自被期望的那条咬住**：不填 context / 用空 userID 查 / 去掉长度上限 / `openSession` 丢掉结果 / 重开不带上下文 / 退回用帧里的 `material_id` / 素材块不发。
- 反向：没有素材 ⇒ 上下文为空、provider 收到空串、instructions 不出现「素材编号」。

**⚠️ 未关闭 —— 端到端仍然是空的**
服务端现在能从**会话记录**取素材正文，但**今天没有任何东西会建出一个带 `material_id` 的会话**：iOS 两处 `createSession` 都硬编码 `materialID: nil`（`DefaultSpeechSessionClient.swift:91-95` / `:107-111`），而创建素材的入口在客户端根本不存在。
⇒ **T1 是必要不充分**：服务端这半边做完了，端到端要等客户端把 `material_id` 传上来（**属 D2 那条线**，另立票 T1-c）。

**明确不做**
- **不改 `session.start` 帧**：那是客户端可写的字段（`provider.go:44-46`）。
- 不引入新端点（app-server 不需要客户端提供的第二个 id）。
- `SessionStart.MaterialID` 这个线上字段现在网关侧**无人读** —— 保留（不改帧），但它已不是素材进 prompt 的路径。

**blocked-by**：D1（按默认 a 落地）

---

### T2 — 消除 `/auth/refresh` 这条 404 死路（G1 的前置）｜✅ 已完成（选了 T2-a）

**目标**：让「客户端在调、后端没有」这件事消失。

**✅ 结论：选了 T2-a（补路由），服务端与客户端两半都已落地。**

| 半张 | 提交 | 内容 |
|---|---|---|
| 服务端 | backend `4035f12` | `account.RegisterRoutes` 加 `POST /auth/refresh`（`internal/account/http.go:32`）；`Service.Refresh`（`internal/account/service.go:205`）校验 refresh token → 换 `TokenResponse`；轮换靠 `issueSession` 里既有的「先删该用户全部令牌再插新的」。测试 `internal/account/refresh_http_test.go` 覆盖有效凭证 / **重放旧凭证换不动** / 空体 / 访客令牌 |
| 客户端 | iOS `be85ca6` | `AuthTokenStoreProtocol.refreshToken()` 读存储里的刷新凭证；`SessionAPIClient.refreshToken` 改成 **拿 refresh token 换 `TokenResponse`**（原来把**过期的 access token**当凭证发出去）；`TokenRefreshCoordinator` 刷新后把**整对令牌**落库 |

**字段形状已逐字对齐**（这是「能跑通」而不是「两边都改了」的证据）：服务端 `account.TokenResponse`
的 JSON 键（`user_id` / `is_guest` / `status` / `access_token` / `refresh_token` / `token_type` / `expires_in`）
与 iOS `TokenResponse` 的 `CodingKeys` 完全一致，且 `refresh_token` 两边都必填。

⚠️ **仍未验证的是真机那一跳**（401 → 刷新 → 重试）—— 本票关的是「404 死路消失」，
**不是「刷新链路在真机上端到端可用」**。

**原状（保留作对照）**
- 客户端整条链是接上的：`FluentWorkAPI.swift:62-63` → `/auth/refresh`；`TokenRefreshCoordinator.swift:122`；`AuthenticatedNetworkClient.swift:72`（生产调用点）；`AppDependencies.swift:526-533` 的 `networkClient` 把它交给 **corpus / dailyRead / sessionHistory 三个客户端**（`:544-570`）【读码】
- ~~后端：`grep -rn "auth/refresh" fluentwork-backend/` **零命中**~~ ⇒ **已于 `4035f12` 过期**
- 访客 access token TTL = **2 小时**（`internal/config/config.go:20`）【读码】
- 已有件：`account.Store.ReplaceRefreshToken` / `DeleteRefreshTokensForUser`（`store.go:31-32`）；`token.go:49` 已在签发 refresh token【读码】

**两个选项（见 D3）**

- **✅ T2-a 补路由**：`account.RegisterRoutes`（`internal/account/http.go:30-36`）加 `POST /auth/refresh`；请求体带 refresh token；**轮换**（旧的立即作废）。
  判据：① 有效 refresh token 换到新 access token；② **旧 refresh token 换不动了**（轮换必须单独一条，否则"没有轮换"的实现也能过）；③ 访客令牌也能刷。
- **T2-b 摘链**（原默认，**未选**）：客户端 401 时改成**重发访客令牌** —— 这条路已经存在（`DefaultSpeechSessionClient.swift:294-310` 的 `ensureAccessToken` 就是这么干的）；删掉 `TokenRefreshCoordinator` 的调用点与 `/auth/refresh` 那个 case。
  判据：① 一条测试断言「401 之后重发访客令牌并重试一次」；② 全仓 `/auth/refresh` 零命中。

**明确不做**：不实现注册 / 登录 / 短信 / 邮箱验证码（那是 G1，见范围外）。

**blocked-by**：~~D3~~ ⇒ **D3 已按 T2-a 拍板**（服务端侧不做注册/登录，只把那一条路由补上，属「消除死路」而非「上 G1」）。

---

### T3 — 放弃练习的服务端语义（B6）｜把一个哑判据变真

**目标**：让 `abandoned` 这个状态**第一次真的被写入**。

**现状（★ 本轮新查出的哑判据）**
- `StatusAbandoned = "abandoned"` 已声明：`internal/session/types.go:17`【读码】
- 读侧守卫已有：`internal/session/service.go:467` —— abandoned 的 `GetReview` 返回 404【读码】
- **但全仓没有任何地方写入它**：`grep -rn StatusAbandoned` 只有上面两处【实测】⇒ **那行守卫永远不会取真值**
- 载体已有：`EndRequest.Reason`（`types.go:148`），网关已在传（`handler.go:868`）【读码】
- 邻居逻辑已有：无用户语音的会话不进回顾管线（`service.go:411-424`，`ReviewSkipped`）【读码】

**改动形状**
1. 约定 `reason == "abandoned"` ⇒ 置 `StatusAbandoned`（store 加一个方法，memory + mysql 两个实现）。
2. `End` 的 abandon 分支**不进回顾管线**：不 enqueue `session.eval`、不 enqueue `session.finished`。
3. 守住 `End` 的既有幂等（`service.go:335-347` 的 `StatusReviewed` 早退、`alreadyEnded`）。

**完成判据（必须先红）**
- **先写这条并确认它红**：abandon 之后 `GetReview` 返回 404。
  现在它写不出来（没有 abandon 的入口）—— 所以分两步：**先**用「手工置为 abandoned 的 store」确认那行守卫是真的；**再**写端到端的。
- **反向**：正常 `ended` 仍然进管线（既有测试保持绿）。
- **误伤检查**：跑 `internal/session` 的**全部**既有测试。

**明确不做**：不碰**单轮** abort（`ClientTurnAbortUserAbandoned`，`internal/voiceproto/frames.go:104`）—— 两件事，别混。

**blocked-by**：无 —— **可以立刻开工**

---

### T4 — 迷你会话的服务端契约（B1）｜✅

**目标**：3-5 回合的会话在服务端**真的会结束**。

**现状（证据，改动前）**
- `CreateRequest` 只有 `material_id` / `scene_type`（`internal/session/types.go:252-255`）【读码】
- **网关全仓没有轮次概念**：`turnLimit|maxTurns|turnCount` 零命中【实测】
- 客户端零（无 UI、无参数）
⇒ 「3-5 回合」此前只是提示词里的一句话：**模型不遵守时没有任何东西会红**。

**改动形状**
1. `CreateRequest` 加 `session_length`（`standard` / `mini`）→ 落库（`practice_sessions` 加列 ⇒ **infra 迁移先行**）。
2. 网关从 `Activate` 的响应里拿（**与 T1 同一个落点，建议同一批做**）。
3. 每轮 `user.speech.end` 计数；达上限时给模型一句收尾指令 —— 复用**已有的**会话中注入通道 `UpdateInstructions`（`internal/voiceduplex/volc_duplex.go:240`，B14 V2）。
4. 向客户端发终止信号 —— **这是跨仓契约**：照 Stage 3 的先例走 v2 **可选**字段（`ai.turn.end` 加 `session_complete`），**不能碰 v1 冻结的二进制帧**。

**已落地（infra `17a4d91` → backend `5320818`）**
1. infra：`$defs.aiTurnEnd` 加可选 `session_complete: boolean`。**可选而非必需** —— 在 `additionalProperties: false` 下，「必需」会让还不认识这个字段的客户端整帧解析失败；缺席是有意义的值（「这场没有长度契约」）。
2. `session`：`CreateRequest.session_length` + migration `0028` 加列 `practice_sessions.session_length`（`DEFAULT 'standard'`，所以既有行与不提这个字段的客户端行为不变）。
3. **N 只有一个来源**：`session.Service.turnLimit()` 读**记录**判是不是 mini、读 **`cfg.MiniSessionTurnLimit`**（env `MINI_SESSION_TURN_LIMIT`）给数，`<= 0` 归一为 `session.DefaultMiniSessionTurnLimit = 5`。config 只读 env、只拒负、**不设默认** —— 照 corpus 对 DRILL 的先例「0 = unset，消费者归一」，于是裸 `Config{}` 也拿到 PRD 行为，而那个数只在 session 包里写一次。
4. 链路：`ActivateResponse.turn_limit` → `ActivateResult.TurnLimit` → `SessionContext.TurnLimit`（**与 T1 同一个落点**）。
5. `voicegateway`：`noteUserTurn` 在 collect goroutine 里计轮（`collecting` 的 CAS 保证一轮一次）；达上限置 `sessionComplete` 并投一句收尾指令。`markSessionComplete` 在 `sendOutbound` 给 `ai.turn.end` 盖章。
6. `provider`：新可选接口 `SessionInstructionInjector`，方法是 **`QueueSessionInstruction`（排队，不是立即下发）** —— 只有 provider 知道 commit 边界在哪。`deliverPendingInstruction` 在 `CommitAudio` **之前** `UpdateInstructions`。

**完成判据与证据**
- 判据 **11 条 / 4 个位置**：`internal/session/session_length_http_test.go`（5，走真 HTTP + 内部端点）、`internal/voicegateway/handler_turn_limit_test.go`（4，走真 WSS 握手）、`internal/voicegateway/provider_session_instruction_test.go`（2，provider 级）、`internal/voiceproto/frames_test.go` 的反射条（1）。
- **先红后绿**：session 5 条先红（`turn_limit = 0, want 3` / `status = 200, want 400`）、gateway 4 条先红、voiceproto 反射条先红（`$defs.aiTurnEnd does not declare "session_complete"`）。
- **N 来自配置而不是常量**：`TestTheTurnLimitIsWhateverActivationReported` 用子表 **1 与 3** 两轮跑同一条路径 ⇒ **写死 5 的实现必红**（这正是 §4 第 3 问要的）。另有 `TestActivate_MiniTurnLimitFollowsASecondConfiguredValue`（配 7）与 `TestActivate_UnsetMiniTurnLimitFallsBackToTheDefault`（未设 ⇒ 5）。
- 反向：`standard` 不产生收尾指令、不产生终止信号（3 轮驱动）。
- **变异 11 次，10 次被期望的测试咬住**。M8/M11 第一次都是**假信号**（丢 `skipped` / 去 `preload` 声明 ⇒ 变量未使用 ⇒ 编译失败 ⇒ 测试二进制没跑）；换成保持编译的变体后才咬住。**本仓第三次撞同一陷阱**（T5-a、T1、本票）。

**两个不那么显然的决定（理由在这里，代码里只有一行注释）**
- **终止信号是粘性的**：`sessionComplete` 一旦置位，之后**每个** `ai.turn.end` 都盖。它是**会话的属性**，不是一次性事件 —— 客户端漏读一帧不该就此静默，否则它再也等不到终止信号。
- **注入必须交回它排空的事件**：`deliverPendingInstruction` 把这次等待**吞掉的厂商事件当 `preload` 交回** `WaitTurn`。不交回就是**静默丢转写**（厂商在音频上传期间就推 transcription，8s 等待会吃掉它），而旧的 `WaitTurnResult` 没有 preload 入口。**这是本票最容易漏掉的半张。**

**⚠️ 未关闭 —— 端到端仍然是空的（与 T1 同一种残留）**
契约与网关都就绪，但客户端**零命中**（`fluentwork-ios` 对 `session_length|session_complete|turn_limit` 全仓零命中）⇒ **今天没有任何调用方会要一场迷你会话**。

**⚠️ 门禁第 7 步未验证**
第 1-6 步绿（gofumpt / goimports / golangci-lint 0 issues / `go test ./...` / `go build` / 环境加载器 17 条）。**第 7 步 `check-dev-service.sh` 在本机跑不了**：它第 4 节用 `ps -o command= -p`，而该执行层 deny `ps`（`/bin/ps` 绝对路径也是 operation not permitted），脚本开了 `pipefail` ⇒ 127 被当成整条管道失败 ⇒ 中止。xtrace 实测第 **1-3 节 10 条断言零 ✘**，死在 4-10 节 ⇒ 那 7 节未验证。**待本地跑一次补上。**

**残留（本票不做）**
- 收尾指令是中文硬编码（`voicegateway.closingInstruction`）；PRD 未规定多语言口径。
- 契约存在**第三份副本**（iOS `Shared/FluentWorkCore/Resources/Schemas/wss-control-frames-v2.json`），本票未同步 —— **两个仓都没有门禁校验副本一致**，留作 iOS 侧 follow-up（见 §7）。

**blocked-by**：D4（按默认落地）；契约那一半 blocked-by infra（`schemas/` 先行）

---

### T5 — A1 的口径与提炼产物

**现状（证据）**
- **上限是两层漂移，不是一处不一致**：
  - PRD A1（`20_…PRD.md:179` / `:161`）：粘贴上限 **2000 字**；UI 文档（`21_…:255`）：「超限**截断**并说明『将基于前 **2000 字**生成』」——**截断是客户端行为**。
  - B21 后端草案（`70_…:90-91`）：文本 ≤ **5000 字符**。
  - 实现（`internal/materials/service.go:68`）：`len(content)`，即 **5000 字节** ⇒ 中文 5000 字节 ≈ **1666 字**，**比 PRD 更严**；英文则更松。
  - **值（2000 vs 5000）与单位（字 vs 字节）是两件事**，原文把两者捆成一句，导致判据自相矛盾（只改单位则「2001 必须拒绝」不成立）。
- **提炼产物不存在**：refine 只产话术块（`refiner_prompt.go`），**没有主题 / 术语 / 讨论点**。

**T5-a（✅ 已完成）**：只修**单位** —— 判据改 `utf8.RuneCountInString`（`service.go:69`），`MaxContentLen` **保留 5000**，doc 注释与错误文案从「字节」改为「字符」。
判据：`internal/materials/content_limit_test.go` 三条 —— 2000 汉字通过 / 上限处通过 / 上限+1 拒绝。改前前两条红（`content exceeds 5000 bytes`），改后全绿；三次变异各自被期望的测试咬住（退回 `len()` / 边界 `>`→`>=` / 上限放大到失效）。
**为什么值不动**：UI 文档要的是**截断 + 说明**，不是拒绝 ⇒ 2000 字是**客户端 UX 上限**，服务端留 5000 字符作安全网（`MEDIUMTEXT`，无 DB 级约束）。若产品要服务端硬拒 2000，那是把客户端文案改成「拒绝」的**语义变更**，另立票。

**T5-c（客户端半张，依赖 UI）**：粘贴框实时字数 + 超 2000 字截断 + 「将基于前 2000 字生成」文案。**UI 未定前不做。**

**T5-b（大，依赖 D1）**：refine prompt 加产物 + 加列/新表 + 迁移。**如果 T1 选了 a（注入正文），这张票可以不做。**

**blocked-by**：T5-b blocked-by D1；T5-c blocked-by UI

---

### T6 — h8 端到端真机跑一次

Stage 3（轮次归属）四仓已收口并推送，但 **h8 链路一次端到端真机跑都没有**（既有清单已记）。
判据：真机跑一场，`timing_audio_engine_configuration_changed` 之后出现 `timing_audio_capture_first_buffer`，且不再 `phase_transition → failed`；服务端侧 h8 帧被两端正确解析。
**不阻塞开发，可与 T1–T5 并行。**

### T7 — 性能红线的量化基线

PRD §9.3 的三条红线（首响 P90 ≤1.5s / 评价 ≤15s / 冷启动 ≤2.5s）**现在一个数都没有**。
埋点都在（`server_ts_ms`、`timing_*`），缺的是**跑一次 + 汇总**。
**不阻塞开发。** ⚠️ 汇总时先读 `问题总清单-产品与缺陷.md` 的 P1-13（`receiveLatency` 三处测错对象）——**不要直接用旧的 `timing_socket_receive` 读数**。

---

## 3. 依赖图与开工顺序

```
D1 ──→ T1 ✅ ─┐
              ├──→ （T4 与 T1 同落点，已一批做掉）
D4 ──→ T4 ✅ ─┘        └──→ infra（契约）先行 ✅ 17a4d91
D3 ──→ T2 ✅（a）
D2 ──→ T1-c（客户端把 material_id 传上来 —— 端到端的前提）
        T3 ✅（已完成）
        T5-a ✅（已完成）
        T5-c ←── UI
        T5-b ←── D1
        T6 / T7（并行，不阻塞）
```

**建议顺序**：**T3 ✅ → T5-a ✅ → T2 ✅ → T1 ✅ → T4 ✅ → T5-b**

**服务端执行票已全部清空。** T4 是最后一张未被阻塞的服务端票；T2 的两半随后也各自落地
（服务端 `4035f12` 早已在树里，客户端半张 `be85ca6`）。剩下的分别是：

| 剩票 | 卡在哪 | 谁来解 |
|---|---|---|
| **T5-c**（粘贴框字数与截断） | UI 未设计 | 设计 —— **⚠️ 它现在是关键路径**：T1 与 T4 的服务端都完成了，而**创建素材的入口在客户端根本不存在**，所以这两张票今天是「必要不充分」 |
| **T1-c**（客户端把 `material_id` 传上来） | **D2 已拍（2026-09-26）** ⇒ 不再是「等决定」；真正缺的是 T5-c 那个入口 | 可开工：先补 materials 数据层 + 让 `createSession` 带上 `material_id`（两处硬编码 `nil`：`DefaultSpeechSessionClient.swift:91-95` / `:107-111`） |
| **T5-b**（提炼产物） | D1 | Tango 拍 D1；**若 T1 选了 a，这张可以不做** |
| **T6 / T7** | 需要真机 / 需要一次真跑 | 硬件与联调窗口，不阻塞开发 |

⇒ 服务端侧已清空。**但「服务端清空」不等于「功能可用」** —— PRD B1 这条线现在缺的是**客户端的入口**（T5-c），不是服务端的实现。

**T2 已关闭（见上）**：服务端 `4035f12` + 客户端 `be85ca6`，选的是 **a（补路由）**。
⚠️ 仍未验证的是**真机那一跳**（401 → 刷新 → 重试）—— 关掉的是「404 死路消失」，
**不是「刷新链路在真机上端到端可用」**。

---

## 4. 每条判据的自查（开工前先问一遍）

本仓最贵的形状是「**看起来有保护、实际没有**」。每张票写测试之前，先问这四个问题：

1. **这条判据真的会红吗？** —— T1 的 `instructionsForSessionStart` 纯函数测试**今天是绿的**，所以它不算判据。
2. **它测的是哪一半？** —— T3 的守卫在**读侧**，而缺的是**写侧**；只测读侧等于测了半个不变量。
3. **它能区分「接线正确」与「硬编码同一个值」吗？** —— T4 的 N 必须来自配置，否则写死 5 的实现也能过。
4. **这条绿测试守的是行为，还是它恰好钉住了缺陷？** —— T2-a 的轮换要**单独一条**测试，否则"没轮换"也能过。

---

## 5. 范围外（本轮不做，写下来免得被读成漏了）

- **UI / 界面形态**：本工单范围外（这是 Tango 的口径）。
- **E / H 的客户端整块**（闪测、话题建议）：取决于 **D2**。
- **G1 注册 / 登录 / 短信 / 邮箱验证码**：需要外部资源（服务商资质、成本、合规），不是工程决定。
- **推送通知**（PRD §9.2 / H4）：需要 APNs 证书与推送服务端。
- **模块 I 语音评测**：PRD 自己后置到 V1.1 / V2.0。
- **订阅与商业化**（PRD §十五）：PRD 自己后置到 V1.1。
- **既有三轴的条目**（`iOS-S0-x` / `BE-S0-x` / `P0-x` / `P1-x` / `F-x`）：归 `问题总清单.md` 三条子文件，本工单不重复登记。

---

## 6. 推进方式

1. **一次只推进一张票**，跨仓串行（infra → backend → iOS → meta）。
2. 每关一张票：在本文对应行标 ✅，补**提交哈希**与一句话结论。
3. **决策票关掉时，把结论写回 D 那一节**，并把受影响的执行票的"未拍板前按…"改掉。
4. 新增问题直接进表，不另开文档。

---

## 7. 跨仓债务：共享契约其实有**三份**副本，没有门禁守它们

写下这条是因为它会在**每一次**契约变更里复发，而且不红。

真源只有一处：`fluentwork-infra/schemas/transport/wss-control-frames-v2.json`。但它有两份**被 check-in 的下游镜像**，各由一个同名的 `scripts/sync-shared-schemas.sh` 抄过去：

| 副本 | 路径 | 消费方 |
|---|---|---|
| 真源 | `fluentwork-infra/schemas/transport/…` | CI 的 `check-schema-freeze.sh` / `check-repo-structure.sh` |
| ② | `fluentwork-backend/schemas/transport/…` | `schemas/embed.go` 的 `go:embed` + `voiceproto` 的反射式契约测试 |
| ③ | `fluentwork-ios/Shared/FluentWorkCore/Resources/Schemas/…` | `SharedSchemaMirror` + `PackageBaselineTests` |

**坏消息**：两个仓的 `setup-git-hooks.sh` 与各自的落地门禁**都不校验副本是否一致**（查过，没有）。⇒ 改了真源忘了同步某一份，**没有任何东西会红**。

**半好消息**：iOS 那份的测试只断言字段**存在**、从不断言**不许多字段**（`Tests/FluentWorkCoreTests/PackageBaselineTests.swift`）⇒ 同步过去只会让它更全，不会弄红。

**本票实例（已收尾）**：T4 加了 `session_complete` 后，② 随本票同步（`5320818`）；
③ 当时**未同步**（实测 `diff` 显示 ③ 缺该字段，而 `log_id` 在 —— 说明这份副本此前是被维护的，
属于新增漂移），延期的理由是「iOS 工作区里另有在途未提交的改动（T2 客户端半张），混进去会污染提交」。
**那个理由已随 T2 客户端半张提交（`be85ca6`）消失**，③ 已在 iOS `721554b` 用
`Scripts/sync-shared-schemas.sh` 同步，同步前后各跑一次 `swift test`（626 / 30 全绿）。
⇒ **三份副本今天一致。**

⚠️ **但这只说明今天对了，不说明以后会红** —— 而且**同步之后这条检查更弱了**：
现在三份一致，再补一条「三份 sha256 相等」的门禁，它**在今天必然是绿的**，
按本仓的纪律（「先写判据并确认它红」）就等于又造一处看起来有保护的东西。
要补就得**用变异证明它会咬**（把任意一份的某个字段删掉，确认检查红）。
**要不要补是一条独立决定，不在本工单范围内。**

**要不要补一条门禁**（在 infra 加一个「三份副本 sha256 相等」的检查）：是独立的一张票，不在本工单范围内 —— 但要补就应该**先写那条检查并确认它红**（今天 ③ 就是红的），否则又是一处「看起来有保护、实际没有」。
