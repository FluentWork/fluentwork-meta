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

**我建议**：把 **API client + reducer + middleware + 导航目的地** 划入业务逻辑；**View 才是 UI**。
**未拍板前**：本工单**只排纯服务端票**，不动这条线。

### D3 — G1 注册登录做不做？

**为什么必须定**：决定 T2 走哪条路。注册登录要短信服务商（资质 + 成本 + 合规），**不是工程决定**。

**我建议**：G1 本轮不做 ⇒ **T2 走 b（摘链）**。理由：补一条没有服务对象的 `/auth/refresh`，等于亲手造出第二处「看起来有保护、实际没有」—— 而本仓已经撞到十次以上。
**未拍板前按 b 写。**

### D4 — 迷你会话的回合上限由谁定、定几轮？

**为什么必须定**：PRD 只说「3-5 回合」，没说谁数。`CreateRequest` 现在没有这个字段（`internal/session/types.go:252-255`），**网关全仓没有轮次概念**（`turnLimit|maxTurns|turnCount` 零命中）【实测】。

**我建议**：
- **计轮必须在服务端。** prompt-only（在 system prompt 里写"这是 3-5 回合"）是这个仓最贵的那类假保护 —— 模型不遵守时**没有任何东西会红**。
- 参数走 `session_length` 枚举（`standard` / `mini`），上限**服务端可配**，照 `DRILL_ROUND_SIZE` 的先例（`internal/config/config.go:34,95,146`）—— PRD §9.1 自己要求「调度参数与生成阈值全部服务端可配」。
- `mini` 上限默认 **5**（PRD 区间上沿，宁多不少：早收尾比晚收尾更伤对话感）。

**未拍板前按此写。**

---

## 2. 执行票

### T1 — 素材上下文注入对话（B2）｜最高价值

**目标**：会话打开时，把素材内容送进模型的 system prompt。

**现状（证据）**
- 网关只拼**编号**：`internal/voicegateway/provider_volc_duplex.go:1128-1140` —— `parts = append(parts, "素材编号："+material+"。")`【读码】
- 素材正文在全仓的**唯一**消费者是提炼 prompt：`internal/materials/refiner.go:62`【读码】
- **落点已经存在，不需要新端点、不需要改协议**：`POST /internal/v1/sessions/activate`（`internal/session/internal_http.go:30`），网关在 `openSession` 里必调一次（`internal/voicegateway/handler_control.go:288`）【读码】
- app-server **自己知道** `material_id`：`internal/session/types.go:49`（`Session.MaterialID`）【读码】
- `Start` 有**两个**调用点，都从 `rt` 取服务端解析出来的值：`handler_control.go:277`（首开）与 `handler.go:621`（透明重开），两边都传 `rt.continuation`【读码】

**改动形状**（照 `ContinuationTurn` 的先例 —— `internal/voicegateway/provider.go:39-50` 已经写清了为什么服务端解析的值要 "rides beside `SessionStart`"）

1. `internal/session`：`ActivateResponse` 加 `MaterialContext string`；`Service` 加 `MaterialSource` seam（照 `SetReviewGenerator` / `SetCorpusProvisioner` / `SetEvalProcessor`，`service.go:80-90`）；`Activate` 读 `session.MaterialID` → 取素材 → 拼上下文。
2. `cmd/app-server/main.go`：把素材读接口注入 seam。
3. `internal/voicegateway`：`SessionLifecycle.Activate` 由 `error` 改成返回 response（`session_client.go:20`，**注意 `cmd/integration-voice-gateway/main.go:269` 的假实现要一起改**）；`HTTPSessionClient.Activate` 同步（`session_client.go:160`）；`openSession` 把结果存进 `rt`（**与 `rt.continuation` 同址、同生命周期**）。
4. `VoiceProviderSession.Start` 的第二个参数由 `continuation []ContinuationTurn` **收敛成 `SessionContext`**（含 `Continuation` + `Material`）—— 让类型自己说明「这是服务端解析的、客户端帧带不了的东西」。
5. `instructionsForSessionStart` 用素材上下文替换「素材编号」。

**完成判据（以及它凭什么会红）**
- **主判据**：一条测试，**直接构造一个带 `material_id` 的 session**（绕过客户端），断言 provider 收到的 instructions 含素材文本。
  ⚠️ **不能只测 `instructionsForSessionStart` 这个纯函数 —— 它今天就是绿的**（P1-25 修完形状已经对了）。要测的是**值从 app-server 流到 provider** 这一整条。
- **反向**：没有 `material_id` 时 instructions 不含素材文本，且**不再出现「素材编号」**（旧行为要被替换掉，不是并存）。
- **重开路径**：断言透明重开后 instructions 仍含素材文本 —— 这是 `rt.continuation` 那一族的坑（`handler.go:621`）。

**明确不做**
- **不改 `session.start` 帧**：那是客户端可写的字段，`provider.go:44-46` 已经写过这条理由。
- 不引入新端点（`ContinuationContext` 那种独立端点在这里没必要，因为 app-server 不需要客户端提供的第二个 id）。

**blocked-by**：D1

---

### T2 — 消除 `/auth/refresh` 这条 404 死路（G1 的前置）

**目标**：让「客户端在调、后端没有」这件事消失。

**现状（证据）**
- 客户端整条链是接上的：`FluentWorkAPI.swift:62-63` → `/auth/refresh`；`TokenRefreshCoordinator.swift:122`；`AuthenticatedNetworkClient.swift:72`（生产调用点）；`AppDependencies.swift:526-533` 的 `networkClient` 把它交给 **corpus / dailyRead / sessionHistory 三个客户端**（`:544-570`）【读码】
- 后端：`grep -rn "auth/refresh" fluentwork-backend/` **零命中**（含 `api/openapi-v1.yaml`）【实测】
- 访客 access token TTL = **2 小时**（`internal/config/config.go:20`）【读码】
- 已有件：`account.Store.ReplaceRefreshToken` / `DeleteRefreshTokensForUser`（`store.go:31-32`）；`token.go:49` 已在签发 refresh token【读码】

**两个选项（见 D3）**

- **T2-a 补路由**：`account.RegisterRoutes`（`internal/account/http.go:30-36`）加 `POST /auth/refresh`；请求体带 refresh token；**轮换**（旧的立即作废）。
  判据：① 有效 refresh token 换到新 access token；② **旧 refresh token 换不动了**（轮换必须单独一条，否则"没有轮换"的实现也能过）；③ 访客令牌也能刷。
- **T2-b 摘链**（默认）：客户端 401 时改成**重发访客令牌** —— 这条路已经存在（`DefaultSpeechSessionClient.swift:294-310` 的 `ensureAccessToken` 就是这么干的）；删掉 `TokenRefreshCoordinator` 的调用点与 `/auth/refresh` 那个 case。
  判据：① 一条测试断言「401 之后重发访客令牌并重试一次」；② 全仓 `/auth/refresh` 零命中。

**明确不做**：不实现注册 / 登录 / 短信 / 邮箱验证码（那是 G1，见范围外）。

**blocked-by**：D3

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

### T4 — 迷你会话的服务端契约（B1）

**目标**：3-5 回合的会话在服务端**真的会结束**。

**现状（证据）**
- `CreateRequest` 只有 `material_id` / `scene_type`（`internal/session/types.go:252-255`）【读码】
- **网关全仓没有轮次概念**：`turnLimit|maxTurns|turnCount` 零命中【实测】
- 客户端零（无 UI、无参数）

**改动形状**
1. `CreateRequest` 加 `session_length`（`standard` / `mini`）→ 落库（`practice_sessions` 加列 ⇒ **infra 迁移先行**）。
2. 网关从 `Activate` 的响应里拿（**与 T1 同一个落点，建议同一批做**）。
3. 每轮 `user.speech.end` 计数；达上限时给模型一句收尾指令 —— 复用**已有的**会话中注入通道 `UpdateInstructions`（`internal/voiceduplex/volc_duplex.go:240`，B14 V2）。
4. 向客户端发终止信号 —— **这是跨仓契约**：照 Stage 3 的先例走 v2 **可选**字段（`ai.turn.end` 加 `session_complete`），**不能碰 v1 冻结的二进制帧**。

**完成判据**
- 一条测试断言「第 N 轮之后产生收尾指令 + 终止信号」，且 **N 来自 session 配置而不是常量**（写死常量的实现必须过不了）。
- 反向：`standard` 不产生终止信号。

**blocked-by**：D4；契约那一半 blocked-by infra（`schemas/` 先行）

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
D1 ──→ T1 ──┐
             ├──→ （T4 与 T1 同落点，建议同一批）
D4 ──→ T4 ──┘        └──→ infra（契约）先行
D3 ──→ T2
        T3（无依赖，可立刻开工）
        T5-a ✅（已完成）
        T5-c ←── UI
        T5-b ←── D1
        T6 / T7（并行，不阻塞）
```

**建议顺序**：**T3 → T5-a → T2 → T1 → T4 → T5-b**

理由：T3 与 T5-a 都是**独立、可立刻开工、且做完就减少一处说谎的地方**（T3 把哑判据变真；T5-a 让一条测试从红变绿）；T2 次便宜；T1 是解锁客户端的先决条件，但它改的是热路径（`Start` 接口），**要留出足够的时间做红验证**；T4 与 T1 共用落点，紧挨着做。

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
