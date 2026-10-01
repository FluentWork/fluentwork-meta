# FluentWork account（账号 / 登录）Issue 草案与排期

**版本**：V1.0
**日期**：2026-10-01
**定位**：把「游客 → 注册用户」这条链路排成可执行的 issue，并写明**开始排期的前置**是什么
**上游依据**：`fluentwork-ios/docs/design/2026-09-26-prd-v16-ux/index.html`（屏 12 账号组、屏 04 G4）·
`47_FluentWork-ios第一波Issue草案.md` · `46_FluentWork-backend第一波Issue草案.md`（B1 已含 merge 端点）
**变更说明**：2026-10-01 逐屏还原 屏 12 时查出来的现状 —— **机器造好了，门没开**，据此立草案

---

## 一、为什么要现在排期（截至 2026-10-01 的实测）

这一节只写**查到的事实**，每条都能回到代码。

### iOS 侧

| 事实 | 位置 |
|---|---|
| `AuthMode` 有三个值（`anonymous` / `guest` / `registered`） | `AuthFeature.swift:3-7` |
| 两条身份转移 action（`signedInAsGuest` / `mergedIntoRegistered`）**生产代码零派发者** —— 只出现在测试与守卫表里 | `ScreenEntryGuardTests.swift:75` 把它们归到「由中间件/传输泵派发」，而**中间件只响应、不派发**（`AppBootstrapMiddleware.swift:629/821`） |
| `.mode` 实际只被 bootstrap 路径写过 | `AppReducer.swift:145-155` |
| **没有任何登录 UI**（全仓搜 `登录` / `LoginView` / `verifyCode` / `手机号` 只命中注释与文案） | — |
| `mergeGuestAccount` 客户端方法在，**无调用方** | `SessionAPIClient.swift:7,42` |

### backend 侧

| 事实 | 位置 |
|---|---|
| `internal/account` 只注册 5 条：`POST /auth/guest`、`POST /auth/refresh`、`POST /account/merge`、`DELETE /account/data`、`POST /account/export` | `internal/account/http.go:28-35` |
| **`/account/merge` 由 `RequireRegistered` 把守** —— 要求一个**今天拿不到**的 registered 令牌 | 同上 `:33`、`:46` |
| **没有任何创建「注册用户」的端点**；`Service` 只有 `IssueGuest` / `Merge` / `Authenticate` / `IssueSession` / `Refresh` | `internal/account/service.go:78/110/174/200/205` |
| **合并机器已就绪**：`ReassignFromGuest`（corpus / session 各自一份）＋ `is_guest` / merge 的库结构 | `internal/corpus/service.go:49`、`internal/session/service.go:113`、`migrations/0001_create_users_and_auth_tokens.sql` |

### 稿子侧（形态依据）

- **屏 12 账号组**：`手机号 / 138****8888 · 已绑定 / [换绑]`（`index.html#scr12`）；
- **屏 04 的 G4 注册时机**：游客点「全部入库」时触发，文案「登录后，这些表达才真正属于你」——
  这是**注册真正会被点的那一下**；
- **§07 场景 01（首次进入·未登录）**：演示素材直接作为主路径，「注册后置到第一次『保存语料』时触发」。

---

## 二、开始排期的前置（不解决它，A1 无法开工）

| 前置 | 为什么它是**前置**而不是 A1 的一部分 | 谁来定 |
|---|---|---|
| **短信 / 验证码服务选型** | 它是**外部依赖**（第三方、有单价与发信合规要求），选型决定了接口形态（验证码位数、时效、频控口径）。选错了整条链路要重做 | 需产品 ＋ 运维一起定 |
| **「手机号即账号」的合规口径** | 手机号属个人信息，收集它要动隐私政策；而 A4 的「删除我的全部数据」链路也得覆盖它。两者必须同一口径 | 需产品 ＋ 法务口径 |
| **未登录用户的数据边界** | 今天游客数据与设备绑定。注册后合并的语义（覆盖 / 保留 / 冲突）要与 A4 删除语义一致，否则会出现「删了账号但留了衍生话术块」这种自相矛盾 | 需产品确认 |

> 建议：**这三条先开一个 decision ticket**（不写代码），把结论落到
> `47_D1_D5_开放技术决策备忘录` 的同一体例里。**它一天的产出决定后面三条 issue 的形态。**

---

## 三、Issue 草案

### A1（backend）：签出一个「注册身份」

**建议标题**：`A1: issue a registered identity (phone + verification code)`

**Goal**：让 `is_guest=0` 的用户**第一次可以被创建出来** —— 今天唯一创建用户的函数是 `createGuest`。

**Scope**
- 新增 `POST /auth/login`（或 `/auth/register`，命名取决于前置结论）：入参手机号 ＋ 验证码；
- 校验验证码（对接选定的服务），频控与失败次数上限；
- 复用已有的 `IssueSession(ctx, user)` 签发令牌（它已经在了，只是没人用）；
- 已存在同一手机号 → 走登录而不是创建。

**Acceptance Criteria**
- 第一次调用创建 `is_guest=0` 的用户并返回 `access_token`；
- 同一手机号第二次调用返回同一个 `user_id`（不重复建号）；
- 验证码错误返回 `422`，**不**泄露「这个号存在不存在」；
- 频控命中返回 `429`；
- 单测覆盖：新号 / 老号 / 错码 / 超频。

**Out of Scope**：第三方登录、邮箱、密码。

**Depends on**：前置三条。

---

### A2（backend）：把 merge 链路补完测试与幂等

**建议标题**：`A2: cover guest→registered merge end to end`

**Goal**：`Merge` ＋ `ReassignFromGuest` 已经在了，但**从没被真实调用过**（因为没有注册用户）。
它现在是一条只跑过单测的路。

**Scope**
- 用 A1 签出的注册身份，跑一次真实 merge；
- 幂等：第二次 merge 返回 `already_merged`（契约里已有这个字段）；
- 合并后游客侧的会话 / 语料块归属正确（`practice_sessions` / `phrase_blocks`）。

**Acceptance Criteria**：契约里 `MergeResponse` 的每个字段都有断言；跨模块的
`ReassignFromGuest` 各有一条集成测试。

---

### A3（iOS）：登录 / 换绑 UI ＋ 把身份状态机接上

**建议标题**：`A3: phone sign-in and the missing auth dispatchers`

**Goal**：两件事，缺一不可：**给学员一个登录的地方**（屏 12 账号组），
**给两条身份 action 一个派发者**（今天它们零派发者，状态机的那一半是死的）。

**Scope**
- 屏 12 账号组按稿子形态：`手机号 / <号码> · 已绑定 / [换绑]`；未绑定时是「未绑定 · 现在用的是游客身份」＋「绑定手机号」；
- 验证码输入与倒计时（复用已有的加载态纪律：骨架块 / 明确的进度文案）；
- 登录成功后 `dispatch(.auth(.mergedIntoRegistered(...)))`，并**走已有的合并反应链**
  （`corpus.mergeRebuildStarted` ＋ 缓存 scope 重建 —— 那条链已经写好了，只是从没被触发过）；
- 失败与频控的说人话文案。

**Acceptance Criteria**
- 未绑定 → 绑定 → 显示脱敏号码，且 `AppState.auth.mode == .registered`；
- 合并反应链被触发（判据：`corpus.mergeRebuildStarted` 被派发）；
- 换绑路径可回退，不停在半路。

**Depends on**：A1。

---

### A4（iOS）：注册时机（屏 04 的 G4）

**建议标题**：`A4: prompt sign-in at first 全部入库`

**Goal**：稿子把注册的触发点定在**游客第一次点「全部入库」**，文案「登录后，这些表达才真正属于你」。

**Scope**：回顾页的入库动作上加重轻量面板（不是全屏），可跳过；跳过后不再反复弹。

**Depends on**：A3（没有登录可走时，这个面板就是「点了没反应的入口」——
这也是它当初被记成独立票的原因）。

---

### A5（跨仓）：端到端验收

**建议标题**：`A5: guest→registered end-to-end acceptance`

**Scope**：真机走一遍：新装 → 游客练一场 → 入库时注册 → 数据合并 → 重启后仍是注册身份 →
删除我的全部数据（A4 链路）→ 注册身份与数据一并消失。

**Acceptance Criteria**：这条链路每一步都有可核对的证据（后端日志 / 接口回包 / 屏幕截图）。

---

## 四、顺序与依赖（排期骨架）

```
[前置] 短信服务选型 ＋ 合规口径 ＋ 未登录数据边界   ← 一天的决策，决定后面全部形态
   │
   ├─ A1 backend（签注册身份）────┬─ A3 iOS（登录 UI ＋ 派发者）
   │                              │        │
   └─ A2 backend（merge 补测）────┘        └─ A4 iOS（G4 注册时机，屏 04）
                                            │
                                            └─ A5 跨仓端到端验收
```

**关键路径**：前置 → A1 → A3 → A4 → A5。A2 与 A1 并行（它不依赖 A1 的接口形态，
但需要 A1 产出的身份才能跑真实 merge）。

**不建议并行**：A3 与 A1 并行会让 UI 对着一个还在变的接口形状写 —— 验证码的时效与频控
口径一变，UI 的倒计时与文案都要跟着改。

---

## 五、明确不在本期

- 第三方登录（微信 / 苹果）、邮箱登录、密码登录；
- 发音评测（模块 I，V1.1）；
- 订阅与商业化（屏 13，V1.1 由服务端开关放出）。

---

## 六、与 UI 逐屏还原的关系（一处依赖要说清）

屏 12 的**账号组今天显示「未绑定 · 现在用的是游客身份」**，那是**实话**，不是占位。
A3 落地后它变成真实号码与「换绑」—— **那一步不需要改版式**，只改投影里的一个字段来源。
也就是说：屏 12 的形态已经就位，account 这条链路补进来时不用回头重排。
