# 问题总清单 · backend 架构轴

> 本文件是 [`问题总清单.md`](./问题总清单.md) 的**第 4 节**（按章节拆分，2026-09-25）。
> **唯一入口是索引**；本文件只承载这一条轴的内容。
> **编号**：本节所有 `S0-x`/`S1-x`/`S2-x` **写作 `BE-S…`**（与 iOS 的 S 号撞车，故加前缀）。
> **内容逐字来自** `fluentwork-backend/docs/104_架构分析/05_问题清单与建议.md`
> —— 该文件已于 2026-09-25 随整个 `docs/` 删除（提交 `e2c61f7`）。
> ⚠️ **本文件因此成了这些条目唯一的载体**，不再是「收敛件」。
> 原文件是 `104_架构分析/01`–`04` 四章的收敛，**那四章同样已删除**；
> 下文形如「`01`–`04` §x.y」的行号引用**不再可解析**，只能当历史标注读。


> 基线 `c23cc38`。本文件是 `01`–`04` 的收敛：**不引入新结论**，只把四章里已经带证据的条目排成一张可执行的表。
> 每条都指向原章的行号，可当场复核。

> ### ⚠️ 复核（2026-09-26）—— 本文件已按复核结果改写，读完再挑票
>
> 做 PRD 轴 T4 时顺手逐条复核了一遍（当时要挑下一张票，而本文件 §5 正是挑票入口）。
> **结论：这份清单落后于代码很多，§5 原来的「只做三件事」里 2/3 已经做完。**
> 照旧文挑票会去做已经存在的东西 —— 这正是本系列最贵的那个形状，只不过这次
> 骗的是做的人自己。
>
> 已复核并改写的：§1 的计数据、§2 全部 9 条、§3 全部 8 条、§5、§7 的租约行。
> 每条都标了 ✅ / ❌ 与复核用的命令。**未复核到的条目一律原样保留**（§4「不要改」
> 与 §6 没有复核，它们不是待做项）。方法：每条跑 `ls` / `Grep` / `git log`，
> 照旧标【实测】/【读码】；**不采信本节旧文里的任何计数**。
>
> **已做完的条目对应哪些提交**（`git log` 实测，方便回查）：
>
> | 条目 | 提交 |
> |---|---|
> | BE-S1-1 `materials` 租约 | `25a015e` fix(materials): 让 `processing` 有一个出口 |
> | BE-S1-2 `httpjson.Error` 测试 | `01a5e55` test(httpjson): 给每个 API 错误的唯一出口补上行为测试 |
> | BE-S1-3 解码器 fuzz | `cb99ba3` test(voiceproto): 给手写二进制解码器补上全仓第一个 fuzz |
> | BE-S1-6 音频热路径 benchmark | `46bd534` test(voicegateway): 给音频热路径补上全仓第一个 benchmark，并据此否掉那条怀疑 |
> | BE-S1-7 `providerErrorStrategies` | `8c6792c` fix(voicegateway): 让 provider 失败码只有一个来源，并钉住策略表的每一行都真被转发 |
> | BE-S2-6 `/metrics` 行序 | `8325795` fix(metrics): 5 处 /metrics 标签不再按 map 迭代序输出 |
> | BE-S2-6 **判据修正** | `82bd06e` test(metrics): 判据只该要求行序稳定，不该要求数值稳定 |
> | BE-S2-4 + BE-S2-5 端点/指标 | `a003b16` feat(httpserver): 端点与指标各有一处能一次答清 |
>
> ⇒ **这批恰好是按本文件 §5 旧版的顺序做的**（旧文「只做三件事」的第 1、2 条被
> 逐条落实）。**所以根因不是清单写错了，是清单没有收口环节** —— 做完没人回来标 ✅，
> 下一个人于是从一份已经兑现的推荐里再挑一遍。§6「推进方式」那套收口纪律写在
> [`PRD核心业务逻辑落地工单.md`](./PRD核心业务逻辑落地工单.md) §6，本文件没有对应的一节。
>
> ⚠️ **本轮（同一天第二轮）又演进了一次**：按上面那次重排，第 1 顺位是 `BE-S2-6`，
> 做完后又重排了一次（现在是「二次重排」）。**这份清单一天里重排了两次，每次都要人肉回来改 ——
> 收口环节仍然缺着，只是这次没让它烂掉。**
>
> ⚠️ **第三轮（同一天第三批）：两条复核方法上的教训，比条目本身值钱**
>
> 1. **「零消费方」的结论不能用 `grep -v _test.go` 得出。** 我判定 `discovery`
>    无人消费时恰好把它唯一的消费方（一条测试）筛掉了，于是差点把一次载荷形状变更
>    当成免费的。**与 BE-S1-7 同型、且是同一个坑的第二次**：搜不到标识符 ≠ 没有这个行为。
> 2. **变异跑出「存活」时，先怀疑判据，再怀疑实现。** M4 把内部前缀改成永不匹配以关掉
>    过滤，判据却绿 —— 因为 `account.RegisterInternalRoutes` 在 privacy 服务缺失时**早退**，
>    测试里从来没有挂过内部路由，那条「不泄露内部面」的断言**恒真**。
>    **判据可以被写成恒真的，而它看起来和真判据一模一样。**


### 0. 这份清单怎么读

**分级只按一件事**：不做会怎样。

| 级 | 含义 | 判据 |
|---|---|---|
| **S0** | 不做就继续带着已知风险跑 | 需要一个只有人能给的决定（或跨仓协调） |
| **S1** | 不做会在下一次同类缺陷上再花一遍时间 | 判据存在，但没有东西会因为它失效而变红 |
| **S2** | 不做只是让下一个人多读一会儿 | 重复、命名、注释、死代码 |

**「需决策」列是重点。** S1/S2 可以直接做；S0 里的每一条都卡在一个本仓无法单独回答的问题上。

**确定性沿用系列约定**：【实测】有命令输出；【读码】从源码读出；【推断】可能另有解释。

---

### 1. S0 — 需要先有决定

| # | 条目 | 卡在谁那里 | 证据 | 确定性 |
|---|---|---|---|---|
| BE-S0-1 | **二进制帧带 `turn_id`**（`[4B seq][turn_id][payload]` 或类似） | **跨仓：backend + iOS 同时改** | **2026-09-26 复核：四仓代码已落地**（infra `f1a06e8` → backend `662c732` → iOS `ab5f44e`）—— 走的是「控制帧带**可选** `turn_ref`，下行动态选 h4 / h8 帧头」而不是改帧布局。**剩：真机跑 + 冻结产物摘要重核**（见 §8） | 【实测】 |
| BE-S0-2 | **迁移机制的归属**：上 Go 迁移工具（用上已有的 embed，顺带补版本表）**或**明确声明「迁移靠 shell，只前滚」 | 你 / 运维 | **2026-09-26 复核**：28 up / 4 down / `schema_migrations` 全仓 `grep -rn` **零命中** / **3** 份 shell 副本（旧文写 5：现为 `local-db-init.sh`、`dev-up.sh`、`smoke-review-ready.sh`） | 【实测】 |
| BE-S0-3 | **28 个迁移里 24 个没有 down** —— 补 down，或正式声明「只前滚」 | 你 / 运维 | **2026-09-26 复核**：`ls migrations/*.sql` = **28**、`ls migrations/down/*.sql` = **4** ⇒ 24 个没有（旧文写 26 up / 22 无 down） | 【实测】 |
| BE-S0-4 | **`topic_cards` 加唯一键** —— 需先确认历史数据无重复 | 你（数据确认） | **2026-09-26 复核仍成立**：含 `topic_cards` 的迁移只有 `0015` / `0019` / `0025`，三份里**没有任何 `UNIQUE`**；`0015` 只有普通索引 `KEY idx_topic_cards_user_date`；`topic/scheduler.go:24` 的 `lastRun` 在内存里 | 【实测】 |
| BE-S0-5 | **`voicegateway` 的拆分目标形状**（按「连接/provider/救援」还是「协议/状态/IO」） | 你 / 架构 | **2026-09-26 复核**：`internal/voicegateway` **80** 个 `.go`（25 个非测试），仍是扁平包（旧文写 71） | 【实测】 |
| BE-S0-6 | **网关不能恢复会话** —— 要「携 `session_id` 重连」需要三样，一样都不存在 | 你 / 产品（要不要这个能力） | iOS 侧已标记 + 钉住：`P1-15` ✅ / `P1-23` ✅ | 【读码】 |
| BE-S0-7 | **`DuplexSession.Close()` 与静音泵 goroutine 抢 `s.conn`** —— 数据竞争，且最坏后果不是竞态而是 panic | 你 / 架构（修法三选一，见下） | ✅ **已完成**（backend `c5cd0f5`，取修法 2 + 一处收窄：锁只护**读**，写走「先脱开、再拆除」）—— 见下方收口 | 【实测】 |
| BE-S0-8 | **一轮结束时静音泵的 ctx 被取消，把整条 duplex 连接打死** —— 下一轮直接 `ErrDuplexClosed` | 你 / 架构（修法三选一，见下） | ✅ **已完成**（backend `4f5930d`，取修法 1 的**收敛版**：不是给泵打补丁，而是把**整条写路径**收到会话级 ctx 上；顺带挖出 `WithoutCancel` 是必需的，理由见下方收口） | 【实测】 |
| BE-S0-9 | **一轮超时 = 整条 duplex 死亡，而日志把死连接报成一次普通的 `outcome=timeout`** —— 读侧的同一个形状 | 你 / 架构（**读模型**三选一，见下方专段；不能照抄 `BE-S0-8` 的修法） | ✅ **已完成**（backend `753dbd6`）—— 取的**不是**原三选一里的第 2 条（承认超时即会话死、只把死连接报准），而是把「窗口」与「socket」彻底分开：唯一 reader goroutine 走会话级 ctx，三处窗口退化成纯 `select`。见下方收口 | 【实测】 |

**`BE-S0-1` 与 iOS 的 `iOS-S0-1` 是同一条。** 它在本仓的形态是：`AITTSAudio.Encode` 把 `[4B seq]` 和 payload 直接拼起来，**没有位置放 turn 号**；而 `ai.tts.start` / `ai.tts.end` 的 `turn_id` JSON tag **故意不带 `omitempty`**（声明为必填）。

**两侧合起来才是完整的事实**：turn 号在控制帧上是强制的、在音频帧上是缺失的 —— 所以「这一帧属于哪一轮」只能从它前后的控制帧推。这是 iOS D7 的根因，也是**本系列唯一一条需要两个仓同时改的条目**。

**BE-S0-2 的紧迫性不高，但「读错」的成本是实时的**：`migrations` 包 + `embed` + 一个测试，构成一个看起来完整的机制。任何新加入的人读到这里，都会以为迁移是 Go 程序应用的。

**`BE-S0-6` 是补记的，它此前只活在 iOS 轴上。** 在 iOS 的 [`问题总清单-产品与缺陷.md`](./问题总清单-产品与缺陷.md) 里，`P1-15` ✅ 与 `P1-23` ✅ 都指到同一件事：**网关无法恢复会话**。`P1-15` 原文写得很直白 —— 「**这是一张独立的后端票，不是 P1-15 的修复**」。但本仓这张 S0 表原本没有它，`grep` `resume|恢复会话|session_id|注册表|ticket` 在本文件里零命中。**一条被明确指认为「后端票」的缺口，在后端轴上不存在** —— 这正是本系列反复出现的形状：同一个事实有两个归属地，而只有一个真的记着它。

**「携 `session_id` 重连」需要三样，一样都不存在**（逐条来自 `P1-15` 的后端核实，本次复核未变）：

| 需要 | 现状 |
|---|---|
| 一个能携带 `session_id` 的帧 | `auth` 帧只有一次性 `ticket`，且 schema 声明 `additionalProperties: false` —— **没有位置**放 `session_id` |
| 会话查找 | 全 `voicegateway` 包**无注册表**；状态活在 `sessionRuntime` 里，**连接一断即丢** |
| 活上下文持久化 | MySQL 里只有**记录**（`practice_sessions` / `utterances` / `session_tickets`），没有活会话上下文、音频水位线、provider 句柄（`P1-23` 的核实结论） |

**再加上票据的语义**：`ticket` 是一次性的，重放 → `ErrTicketUsed`；换一张新票 = 一个新的 uuid。所以「重连」在协议层当前**不是「接不上」，是「无从表达」**。

**卡在谁那里：产品，不是工程。** 前面 `BE-S0-1`…`BE-S0-5` 卡的是「怎么改」；`BE-S0-6` 卡的是「**要不要这个能力**」。iOS 侧的处置是**标记 + 钉住、未删除** —— 守卫 `networkLossDegradesAndNeverAttemptsAReconnect` 钉住「不重连」（`startSessionCallCount == 1`），让将来真做重连成为**必须 deliberate 的事**。所以这条现在**不阻塞任何人**；它进表是因为一旦要动它，得先知道三样都不在。

---

**`BE-S0-7` 是本系列唯一一条由「验证」而不是由「读码」发现的条目**，单列一段 —— 它已于 `c5cd0f5` 关闭，下面保留的是**发现时**的形状与证据（行号是发现时的树，修完后该文件的行号已变）。

`internal/voiceduplex/volc_duplex.go` 的 `DuplexSession`：

| 位置 | 行为 |
|---|---|
| `:819` `Close()` | 发 `session.close`，关 WebSocket，然后 **`:828` `s.conn = nil`** |
| `:852` `send()` | **`:857` `s.conn.Write(...)`** —— 直接读该字段，**无 nil 保护** |
| `:738` `collectTurn()` | 每个 turn 起一个 goroutine 跑 `sendSilence`，`:800` 起每 **20ms** 调一次 `send` |

`sendSilence` 上挂的 `defer stopSilence()`（`:737`）只能保证 goroutine **下次回到 `select`** 时退出 —— 拦不住已经走进 `s.send` 的那一次。于是：

- **必然发生的是数据竞争**：`Close` 写 `s.conn`、静音泵读 `s.conn`，`go test -race` 间歇性命中。
- **可能发生的是 panic**：若交错落在 `s.conn = nil` 之后，`s.conn.Write(...)` 是对 **nil 接收者**的调用。`coder/websocket` 的 `Conn.Write` 第一条语句就是 `c.write(...)` → `c.writer(...)` → `c.msgWriter.reset(...)`，**没有任何 nil 检查**。静音泵是个没有 `recover` 的 goroutine ⇒ 崩的是整个 `voice-gateway` 进程。
  ⚠️ **收口时收紧了这一句**：无 nil 检查是读源码验证过的；「交错真会落在 nil 之后」**没有**被验证 —— 它需要一次在 `Close` 返回之后才开始发帧的调用，而本仓找不到。证据分级见下方收口第 1 条。

**确定性与范围**：已在 HEAD `182ec75` 的**干净 worktree** 上复现（不只是本次改动的树），且触发者是**另一个**测试（`provider_volc_duplex_mute_test.go:92` 而非 `provider_session_instruction_test.go:158`）—— 所以这不是某个新测试写坏了，是 `Close()` 与「turn 级 goroutine」之间的生命周期缺口，任何在 turn 进行中关会话的路径都能走到（`session.end`、连接断开、provider close）。

**为什么它至今没人看见**：落地门禁 `./scripts/dev-check.sh` 的第 4 步是 `go test`，**不带 `-race`**。这条缺陷对整套门禁完全不可见 —— 它需要有人主动去跑一次 `-race`。**⇒ 门禁本身缺一条**。**2026-09-27 已补上**：第 4 步改为 `go test -race ./...`，并加了第 8 步 `scripts/check-gate.sh` 钉住门禁自身的形状（backend `ca58de9`）。同一个盲区后来还养出了 `BE-S0-8` 与 `BE-S0-9`。

**三种修法（决定在这里，不在实现里）**：
1. `Close()` 先取消本会话的静音泵并**等待它退出**（要 `collectTurn` 把 `stopSilence` 交出来，或让泵自己注册到 session 上）；
2. 给 `conn` 加互斥/`atomic.Pointer`，读写都走同一把锁；
3. `send()` 对 nil 安全（返回错误 → `sendSilence` 已经在丢弃错误）—— 消掉 panic，但**数据竞争本身仍在**，所以这条只能作为另外两条的补充，不能单独用。

**本轮未修的理由**：`BE-S1-4` 是分派重构，改动落在 `internal/voicegateway`；这条在 `internal/voiceduplex`，且修法需要上面这个选择。**不把两条不同包、不同性质的改动塞进一个提交**，所以单独立票、单独立一个 turn 做。

#### ✅ 收口（backend `c5cd0f5`）

**取的是修法 2，但收窄了一处**，这一处值得单独记，因为原方案有个反向风险：若按字面「读写都持锁」，`send` 会在**持锁期间做网络写**，而 `Close` 要拿同一把锁 —— 于是一条堵在 socket 上的发送会把「本该来解锁它的拆除」堵住，把竞态换成死锁。所以最终形状是：

- 锁只护**读**：`currentConn()` 是字段唯一的读点（持锁、字段没了返回 `errSessionClosed`）。
- `Close` **先脱开再拆除**：在锁内取走连接并置 nil，然后**在锁外**发 `session.close` 与关连接。于是它对同一条连接恰好做一次拆除，第二次 `Close` 或另一个 goroutine 的帧看到的是「会话已关」。
- `writeJSON` 提成自由函数：`Close` 必须写给**它刚脱开**的那条连接，走 `send` 会重新读自己清掉的字段。

`errSessionClosed` **包住 `ErrDuplexClosed`** 而不是另立第三种错 —— 调用方（`provider_volc_duplex.go:652`）正是用这个哨兵决定「换掉这条 duplex」，而「会话已关」与「socket 死了」该走同一个响应。原先那 4 个会检查的方法返回的是裸字符串 `"duplex session is nil"`，**没有调用方能 match**（全仓 grep 过，无消费者）。

**修法 3 没有采用，理由也变了**：它现在不只是「不完整」，而是**多余** —— nil 守卫已经在 `currentConn` 里，一处覆盖全部 8 个方法 + 全部 turn 入口。

**判据四条**（`internal/voiceduplex/session_close_test.go`）：① 关掉后 8 个发帧方法逐个必须给出包住 `ErrDuplexClosed` 的错误（**改前红**：4 个 panic、4 个无哨兵文案）；② turn 入口经 `recv` 继承同一条规则（**改前红**）；③ 该字段的读只允许出现在两个持锁函数里、写只允许构造与 `Close`（**改前红**，8 个函数各一处；读/写**分开**判 —— 构造期的写安全，任何位置的读都不安全）；④ `session.close` 必须真的上线（**特征化守卫，改前就是绿的**，本仓此前无任何地方断言这一帧；它存在的理由是本票自己造的：`Close` 现在绕开 `send` 写这帧，谁把它「简化」回去，这帧就静默消失）。变异 7 条全部咬住（M1 不走锁 / M2 去 nil 守卫 / M3 访问器外读 / M4 允许外写 / M5 弄坏提取器 / M6 帧改回走 `send` / M7 只关不脱开）。

**本轮最重要的产出不是修复，是两条「口径收回」**：

1. **「最坏是 panic」的措辞比证据大。** 已验证的是 `Conn.Write → write → writer → msgWriter.reset` 全链路无 nil 检查（去 module cache 读的源码）；**未验证**的是「读到 nil」在生产里发生 —— 它需要一次**在 `Close` 返回之后才开始**的发帧，而本仓找不到这样的生产调用方。⇒ 是雷，不是已发生的事故。上一轮把它写成「触发路径是任何『turn 进行中关会话』，不是边角」，那是**推断**，不是实测。
2. **一条并发判据写了又删。** 试过「让发送方与 `Close` 撞上，断言不 panic」，想法是「字段一旦为 nil 就不会变回来，所以只以错误为出口的发送方躲不开那次读」。**它在未修的代码上不红** —— 在 HEAD `67c37e0` 的干净 worktree 上不带 `-race`、带 `-race` 各试 3 轮，8 次全绿。实测原因是：写方在 `Close` 把字段置 nil **之前**就撞上「连接已关闭」的传输错误退出了。**它被删掉、没有进仓** —— 一条通不过自己牙的判据，与一条真判据外观完全一样。

**⇒ 重叠本身（数据竞争）现在没有任何判据钉着。** 它按调度发生，不按命令发生；能钉住的是它绝不该产出的状态（panic）和让两者都不可能的访问规则（判据 ③）。这也是为什么下面 §5 的第 1 位（门禁接 `-race`）**仍然要接**：这一条 -race 撞出来过两次，而它没有判据。

---

**`BE-S0-8` 是第二条第「由验证发现」的条目，而且它是被「接 `-race` 进门的那个动作本身」撞出来的。** 2026-09-27 准备把 `-race` 接进 `dev-check.sh` 第 4 步时，先跑一遍确认「今天接不会红」，**第三轮就红了**：

```
--- FAIL: TestCollectTurn_DoesNotFinishOnOutputAudioDoneBeforeResponseDone (0.09s)
    collect_turn_stream_test.go:326: mock server write: ... write: broken pipe
    collect_turn_stream_test.go:351: collectTurn turn-2: duplex connection closed:
                                     failed to get reader: use of closed network connection
```

**红的不是数据竞争**（输出里没有 `WARNING: DATA RACE`），是 `internal/voiceduplex` 的一个**时序相关测试**。复现率：

| 跑法 | 红 |
|---|---|
| `-race`，单实例隔离（`-count=1` + `-run` 只挑这一个，30 个独立进程） | **10 / 30** |
| 不带 `-race`，同进程 `-count=20` | 1 / 20 |
| `-race -count=15`，在 `67c37e0`（**BE-S0-7 之前**）的干净 worktree | **7 / 15** |

第三行是关键：**它不是 `BE-S0-7` 引入的** —— 那个提交里的 `session_close_test.go` 在 `67c37e0` 上根本不存在，flake 照样红。所以它一直在这里，只是门禁不带 `-race`，没人看见（和 `BE-S0-7` 同一个盲区，见 §5.5）。

#### 定位：两次把「看起来像证据的东西」推翻

1. **`DuplexSession.Close` 不是凶手。** 第一直觉是「有人关了会话」。上探针（`Close` 打栈）后发现：失败的那一轮里，进程内**唯一一次** `Close` 的栈是 `t.Fatalf`(line 351) → `runtime.Goexit` → 那个 defer —— 也就是说它是**失败的结果**，不是原因；而且 mock server 的 handler 从头到尾没返回过。
2. **`net.ErrClosed` 不证明「本地调用了 Close」。** 一度据它断定「客户端自己关了」。去 module cache 读 `github.com/coder/websocket@v1.8.14` 才看清：`read.go` 的 `prepareRead` 开头 `select { case <-c.closed: return nil, net.ErrClosed }`，`done` 里还有 `case <-c.closed: if *err != nil { *err = net.ErrClosed }` —— **它是个兜底覆盖错误**：任何原因把 `c.closed` 关掉之后，下一次 `Read` 都报它，真实原因（EOF / close 帧 / 本地关闭）被盖掉。⇒ 按它反推调用者必然推错。

#### 机制（读依赖源码 + 对照实验坐实）

**`coder/websocket` 里「ctx 被取消」= 整条连接被关掉，不是只中断这一次读写：**

```go
// conn.go:164
func (c *Conn) setupWriteTimeout(ctx context.Context) {
    stop := context.AfterFunc(ctx, func() {
        c.clearWriteTimeout()
        c.close()          // ← 关的是整个 Conn
    })
    ...
}
// conn.go:141  setupReadTimeout 同形
// conn.go:153  func (c *Conn) close() { ... close(c.closed); c.rwc.Close() ... }
```

**而静音泵正好把「轮的生命周期」当成了「写的生命周期」：**

```go
// volc_duplex.go:767
silenceCtx, stopSilence := context.WithCancel(ctx)
defer stopSilence()                  // ← 轮一结束就取消
go s.sendSilence(silenceCtx)

// volc_duplex.go:840（sendSilence 内部）
_ = s.send(ctx, map[string]any{"type": "input_audio_buffer.append", ...})
//           ↑ 用的就是那个 ctx
```

于是有两个触发口：① `select` 里 `<-ctx.Done()` 与 `<-ticker.C` 同时就绪时**吃掉了 ticker 那支**，用一个**已经取消的 ctx** 调 `conn.Write` → `AfterFunc` 立即触发；② ctx 在写**在飞**时被取消。两条路都通向 `c.close()` ⇒ **一轮结束把整条 duplex 打死**，下一轮的第一帧就拿到 `ErrDuplexClosed`（`collectTurn` 于是走 `:802` 那支，把它包成 `ErrDuplexClosed`）。

**对照实验（机制的证据，不是猜）**：只把 `sendSilence` 的写换成 `context.WithoutCancel(ctx)`，同尺度再跑 30 个独立进程 ⇒ **0 红**（对照 10/30）。**实验用的改动已还原，没有留在树里。**

#### 生产含义，以及证据分级

- **【实测】** 一轮的结束会（以约 0.1～0.3 的概率）把整条连接打死，且是在**没有任何调用方要求关会话**的情况下。
- **【推断】** 后果为什么重要：本仓的真实调用方（`internal/voicegateway/provider_volc_duplex.go`）在轮与轮之间**要继续用同一条 duplex** —— vendor 侧的会话上下文就挂在这条连接上。连接被打死后调用方看到的是 `ErrDuplexClosed`，走「换掉这条 duplex」，**丢掉的是 vendor 侧会话上下文**。
- **【推断，未实测】** 生产频率。对端会正常读，写的完成更快、窗口更窄，所以不能拿测试里的 10/30 当生产数字。**别把它写成「每轮必挂」。**

**顺带一条要记的事**：上一轮写 `BE-S0-7` 时**已经把这段线索写下来了** —— 原话是「`sendSilence` 上挂的 `defer stopSilence()` 只能保证 goroutine **下次回到 `select`** 时退出 —— 拦不住已经走进 `s.send` 的那一次」。当时把它读成「**竞态窗口**」，而它真正的后果是「**取消掉的 ctx 通过库把整条连接关掉**」。同一句话，两种读法，差一条 P0。

#### 三条修法（决定在这里，不在实现里）

1. **写帧用一个不会在轮结束被取消的 ctx**（`context.WithoutCancel(ctx)` 或会话级 ctx），停泵仍由轮级信号控制。**已用对照实验验证能消掉 flake**；代价是这一次写没有超时了（`AfterFunc` 本来就是写超时），堵死的 socket 会让泵 goroutine 挂到会话关闭为止。
2. **用独立 `done chan struct{}` 停泵**，写帧走会话生命周期 ctx。比 1 更明确（「停」与「取消」彻底分开），但要把「这一次写要不要超时」重新想一遍。
3. **`collectTurn` 不取消泵的 ctx，而只在轮末等它退出**（与 `BE-S0-7` 的修法 1 同形）。

**它挡住了 §5 的第 1 位**：`-race` 一接进门禁，这条 flake 就会让门禁常红（约 1/3 的运行）。⇒ 顺序变为**先 `BE-S0-8`、再 `-race`**（这条顺序不是风格问题，是「门禁必须稳定绿」这个前提）。

#### 收口（2026-09-27，backend `4f5930d`）

**落地的不是「给泵打个补丁」，是把整条写路径收到会话级 ctx 上。** 复核时按源码重数发现：`s.send(ctx, …)` 有 **9 个调用点**，每一个都把调用者的 ctx 直接送进 `conn.Write` —— 泵只是**最先撞上**的那一个，其余 8 个（`SendPCM` / `CommitAudio` / `AppendPCMChunk` / `RequestTextTTS` …）是同一个隐患。只修泵的话，下一次谁在写窗口里撤销一次 ctx，就再来一遍。

- `DuplexSession` 加 `writeCtx` / `cancelWrite`，在 `OpenDuplex` 里 `context.WithCancel(context.WithoutCancel(ctx))` 建立，**只在 `Close` 里释放**。
- `send` 不再把调用者的 ctx 交给 `conn.Write`：它只用来判「这一帧还要不要发」。一个撤销的调用者**失去这一帧，不再失去整个会话**。
- `Close` 保留调用者 ctx 写告别帧 —— 唯一例外，理由具体：它写的是自己**已经脱开**的那条连接，而且它本身就是拆除动作。
- 泵的签名与行为**没变**：它仍然拿轮级 ctx 当停止信号，但那个 ctx 再也碰不到 socket。

**⚠️ `WithoutCancel` 是必需的，不是装饰** —— 这一条不在原修法里，是复核生产调用方时挖出来的：`provider_volc_duplex.go:915`（重连路径 `defaultResetDuplex`）用 `resetCtx, cancel := context.WithTimeout(ctx, …)` 开新会话，**然后 `defer cancel()`** —— 函数一返回那个 ctx 就没了，而会话要继续服务后面的每一轮。会话级写 ctx 若直接从它派生，**重连之后每一次写都打在一条作废的 ctx 上**。⇒ 判据里专门留了一条（`TestTheSessionOutlivesTheContextThatOpenedIt`），变异 M4 咬住的就是它。

**判据（`internal/voiceduplex/session_write_ctx_test.go`，三条，全部留仓）**

| 判据 | 形态 | 修前 | 修后 |
|---|---|---|---|
| `TestEveryFrameOnTheWireUsesTheSessionWriteContext` | **结构（确定性）** —— `conn.Write` 只许在 `writeJSON` 一处；`writeJSON` 的 ctx 实参只许是 `s.writeCtx`（`Close` 除外）；`s.cancelWrite` 只许在 `Close` 里被调 | 红（确定性） | 绿 |
| `TestACallerCancellationDoesNotKillTheSocket` | 行为 —— 把**已经撤销**的调用者 ctx 六次灌进写路径，连接必须照常可用 | **9/10 红**（不带 `-race`） | 0/15 红 |
| `TestTheSessionOutlivesTheContextThatOpenedIt` | 行为 —— 取消「打开会话用的 ctx」后会话照常写 | 绿（特征化） | 绿 |

变异 **6 条全部咬住**（并点名 `文件:行:函数`）：`send` 换回调用者 ctx / `Close` 不再释放 / 改在轮末释放 / 会话级 ctx 直接派生自打开它的 ctx / 泵绕过 `send` 自写帧 / 另开一条直写路。

**清理验证（这才是「修好了」的证据）**：原 flake 用例在 `-race` 下 **修前 10/30 红 → 修后 0/20 红**；全仓 `go test -race -count=1 ./...` 连跑 **3 轮全绿**；门禁八步 `All checks passed.`。

**这一轮也把「判据怎么写」补了一课**：竞争类缺陷的**行为判据**在没有 `-race` 时几乎测不出来（`BE-S0-8` 的多轮回归判据修前只有 2/10，`-race` 下才 10/10）—— 所以这类条目的判据**必须有一条钉在结构上**，行为判据只能作后果回归。另外：时基判据的**停顿时长要按机制校准相位**（泵 20ms 一拍，停 25ms 永远错开、一次不红；停 80ms＝4 个整节拍才撞得上）。

**上一轮那句「最坏后果是 panic」的同类风险，这里没有重犯**：`AfterFunc` 的触发、写窗口的重叠都是【实测】或读依赖源码得到，生产频率标的是【推断，未实测】。

---

**`BE-S0-9` 是第三条第「由验证发现」的条目，也是本轮查写侧时顺手查出来的读侧对称面。** 同一个根因（**轮级取消 = 连接级后果**），但**修法不能照抄 `BE-S0-8`** —— 写侧可以「不需要超时」（写堵住了还有 `Close` 兜底），读侧不行：读**必须**有办法放弃等待，而在这个库上「放弃等待」就等于「关连接」。

复现是**确定性**的（一次性探针，未进仓）：mock 服务端握手后一直静默但保持读；`collectTurn(ctx, now, nil, 150ms)` →

```
第 1 轮（静默到超时）：outcome="timeout" err=duplex turn timeout: failed to get reader: context deadline exceeded
服务端读循环退出：failed to read frame header: EOF          ← 客户端 socket 已经关了
第 2 轮：outcome="error" err=duplex connection closed: failed to get reader: use of closed network connection
```

机制和写侧是同一处：`read.go:232` 的 `setupReadTimeout` 也是 `context.AfterFunc(ctx, c.close)`，而那个 ctx 是 `collectTurn` 里带轮窗口 deadline 的 `readCtx`（`:795`）。同样形状的读点还有两处：`UpdateInstructions`（`:312`）与 `sendUserPCMInject` 的 `drainCtx`（`:545`，固定 3s）。

**为什么这条比 `BE-S0-8` 更值得担心**：`TurnOutcomeTimeout` 是**设计好的正常状态**（`docs/20 §1.2.a`，会照常报给 iOS）。也就是说「一轮超时」在日志里长得完全无害，而它已经把会话打死了 —— 下一轮的失败会被算到别的原因头上。

**三条修法（这次卡的是读模型，不是实现）**：

1. **轮窗口不再用「取消」实现**：读走会话级 ctx，窗口改成「读跑在 goroutine 里 + `select` 到期放手」。问题是放手之后那一次读**还会读到事件**，得由读方自己持有并转交给下一轮 —— 这是要设计的东西，不是改一行。
2. **承认「超时即会话死」，并让它可见**：`outcome=timeout` 显式标记为「需要重连」，让调用方的「换一条 duplex」变成有意的动作，而不是一次被读成 timeout 的静默死亡。**最小改动，但它只是把死连接报准，不是修好。**
3. **改成唯一 reader goroutine 派发事件**（有界事件 channel）：轮窗口只影响**消费者**的等待，永远不碰 socket。最彻底，也最大。

**✅ 收口（2026-09-27，backend `753dbd6`）—— 取的是第 3 条，但比它小得多**

票里说三条「成本差一个数量级」，**复核后这个判断是错的**：第 1 条与第 3 条其实是**同一件事的两半**（唯一 reader + 窗口退化成「消费者自己的等待」），合起来才是完整答案 —— 而它的收敛面只有 `recv` 一个函数，三处调用点一行都不用改。

| 项 | 结果 |
|---|---|
| 改法 | 新增会话级 `readCtx`（`WithCancel(WithoutCancel(ctx))`）+ `cancelRead`，**只在 `Close` 里释放**；新增 `readLoop`（唯一读 socket 的 goroutine）；`recv` 变成纯 `select`（`events` / `ctx.Done()` / `readDone`） |
| 三处窗口调用点 | **一行都没改** —— 照旧传自己的窗口 ctx，语义逐条保持：`DeadlineExceeded` → `timeout`/`partial`、`Canceled` → `ErrTurnCancelled`、死连接 → `ErrDuplexClosed` |
| 收敛面 | 按源码重数是**三处**对称的点（轮窗口 `:795`、8s update 窗口 `:312`、3s drain `:545`）—— 与 `BE-S0-8` 一样，不是一处。票里只写了轮窗口那一处 |
| `WithoutCancel` | **必需，不是装饰**，与写侧同理由（`defaultResetDuplex` 用带 `defer cancel()` 的 ctx 开会话）—— 判据 4 钉它 |

**判据（`internal/voiceduplex/session_read_ctx_test.go`，四条，全部留仓）**

| 判据 | 形态 | 修前 | 修后 |
|---|---|---|---|
| `TestEveryFrameOffTheWireUsesTheSessionReadContext` | **结构（确定性）** —— `conn.Read` 只许在 `readLoop` 一处；`cancelRead` 只许在 `Close` 里被调 | 红 | 绿 |
| `TestATurnTimeoutDoesNotKillTheSession` | 行为 —— 第 1 轮必然超时，第 2 轮必须成功 | 红（确定性） | 绿 |
| `TestAWindowExpiryLeavesTheSocketReadable` | 行为 —— `recv` 上最小的那一形式 | 红（确定性） | 绿 |
| `TestTheReaderOutlivesTheContextThatOpenedIt` | 行为 —— 对称于写侧那条 | 绿（特征化） | 绿 |

变异 **6 条全部咬住**（并点名测试）：`recv` 回到直读 socket / `Close` 不再释放读 ctx / 读 ctx 派生自调用者 / 死连接的错不再包 `ErrDuplexClosed` / 轮窗口过期时取消会话级读 ctx / 删掉放手分支。

**这一轮最值得记的是：既有判据当场拦下了一次回归。** 第一版 `recv` 在 `readDone` 分支返回的是库的原始 close-frame 错误，而 `BE-S0-7` 立的契约（关掉的会话，每个方法都要给出**包住 `ErrDuplexClosed`** 的答案）就在 `TestAClosedSessionAnswersEveryFrameWithAnError` 里 —— **它红了**，修好之后由变异 M4 钉住。⇒ 前几轮写的判据不是文档，是**真的会在下一轮咬人**的。

**判据自己也被咬了一次**：结构判据的第一版只匹配裸 `recv(`，而调用点是 `s.recv(...)`，于是「一个调用点都没扫到」—— 反空洞分支**按设计报了「提取分支失效，本判据给不出结论」**，而不是报「仓库干净」。这是反空洞第一次在**判据自身**的缺陷上生效。


### 2. S1 — 结构上该补的「防线」

| # | 条目 | 动作 | 风险 | 价值 | 证据 | 2026-09-26 复核 |
|---|---|---|---|---|---|---|
| BE-S1-1 | **`materials` 的 refine 没有租约**，`processing` 是没有出口的状态 | 照 `session_jobs` 抄：加 `attempts`/`locked_at` + 清扫，或挪进 `cmd/worker` | 中 | **高** | `03` §2.2 | ✅ **已完成** `25a015e` |
| BE-S1-2 | **`httpjson.Error` 零测试**，而它是每个 API 错误的唯一出口，有 4 个分支 | 补 4 个分支的测试 | 低 | **高** | `04` §3.2 | ✅ **已完成** `01a5e55` |
| BE-S1-3 | **手写二进制解码器没有 fuzz**（全仓 0 个 fuzz） | 给 `DecodeAITTSAudio` 写 fuzz | 低 | **高** | `04` §3.1 | ✅ **已完成** `cb99ba3` |
| BE-S1-4 | **控制帧分派是线性扫描，同一帧被解码最多 8 次** | 改成按类型查表，把已解出的 `frameType` 传下去 | **低**（只搬分派） | 中 | `02` §3.1 | ✅ **已完成**（backend `67c37e0`；实测是 **9** 次不是 8） |
| BE-S1-5 | **`ai.audio.chunk` 常量无生产者、无注释** | 加一句「无生产者，保留为协议位」；**v1 已退役但这张死面仍在 v2**（见 §8） | 无 | 低 | `02` §3.2 | ❌ **仍开着** |
| BE-S1-6 | **音频热路径的逐帧 `Debug` 日志**，成本无法量化（因为 0 个 benchmark） | 先加 benchmark，再用数据决定 | 低 | 中 | `02` §3.4、`04` §3.1 | ✅ **已完成** `46bd534` |
| BE-S1-7 | **`providerErrorStrategies` 的 key 与 handler 的对应关系只靠读** | 补一条测试：表里每个 key 都必须是真会转发给 provider 的帧类型 | 低 | 中 | `02` §2.1 | ✅ **已完成** `8c6792c`（发现比旧文更严重） |
| BE-S1-8 | **`config.Config` 完整夹具复制到 27 个文件（61 处，其中 35 处是完整夹具、跨 14 个文件）** | 收敛成 `configtest.Config()` + 一条守卫 | 低 | 中 | `04` §3.4 | ✅ **已完成** `e6458f3`（实测 **23→27** 文件、**约 56→61** 处；原文那句「已经漂移：未知」**实测是「已漂」**） |
| BE-S1-9 | **`golangci-lint` 未安装** → `depguard` 那条纪律在本机从未执行过 | 装进 CI 镜像 + 写进 `SETUP.md` 前置条件 | 低 | 中 | `04` §3.5 | ✅ **前提不成立** |

**逐条证据（2026-09-26）**

- **BE-S1-1 ✅** —— 落地物齐全：`materials/model.go` 的 `DefaultRefineLease` / `MaxRefineAttempts = 2` / `ReclaimInterval` / `ErrorLeaseExpired`；`Store.ReclaimExpired`（`memory_store.go:108` + `mysql_store.go:109` 两个实现）；`Service.ReclaimExpired` + `Service.SweepIfDue`（`refiner.go:111` / `:135`，带 `lastSweep` 节流）。判据是 `internal/materials/refine_lease_test.go` **7 条**（活租约不动 / 过期重排 / 次数用尽转 failed / 跳过已删素材 / service 级重跑 / `SweepIfDue` 一个间隔只扫一次 / 重排记 transition）。
- **BE-S1-2 ✅** —— `internal/httpjson/httpjson_test.go` 存在（与 `httpjson.go` 同级）。
- **BE-S1-3 ✅** —— `internal/voiceproto/frames_fuzz_test.go:11` 的 `FuzzDecodeAITTSAudio`。**全仓已不止一个 fuzz**（旧文「全仓 0 个」作废）。
- **BE-S1-4 ✅ 已完成**（backend `67c37e0`）—— 旧文说「同一帧最多解码 **8** 次」，实测是 **9**：`handler_control.go` 里 9 处 `DecodeType`（1 处分派 + 8 个 handler 各一遍），另 1 处在 `handler.go:255` 的 `handshake`（auth 帧，发生在分派存在之前，不受影响）。旧文那句「`handler_control.go:124` 的 9 个 handler」也不对 —— 是 **8** 个 handler、9 个**类型**（`controlUserSpeech` 认领一句发言的两端）。
  **改法**：`controlDispatch map[string]controlHandler`（一帧一行，查表即分派）+ `controlFrame{typeName, data}`（已解出的类型随帧交下去）；`controlOutcome` / `controlHandled` / `controlNotMine` **整个删掉**（12 处引用 → 0，查表已决定归属，handler 没有「不是我的」可答）。顺带拆掉一层**不可达且行为不一致**的死分支：handler 内部那句 `if err != nil { return controlNotMine, nil }` 外层已经解过一次，永远走不到 —— 而它若走到会掉进 `unknownFrame`（静默吞掉），与外层解码失败的 `invalid_frame` 是两个答案。
  **判据三条**（`internal/voicegateway/handler_control_dispatch_test.go`）：① 包内非测试文件的 `DecodeType` 调用点只允许 `handshake` 与 `HandleControl` 两处（**改之前跑是红的**，点名 8 个处理器与文件行号）；② 9 个 C→S 类型逐个写死断言，**不从表反推**（反推会让判据同意表的任何说法，包括删掉一行之后）；③ AST 扫 `func (h *Handler) control*` 与表双向比对 —— 编译器只管「行指向不存在的方法」，管不了「写了 handler 忘了加行」，而这次改动恰好把失败模式从「我得改 8 个文件」（响）挪到了后者（哑）。
  **变异 6 条全部咬住**：表漏一行（判据 ② 报「客户端发得出、没人认领」+ 判据 ③ 报「处理器没人路由」，两侧各一口）/ 表多一行 / 把解码加回 handler / **包级变量初始化里解码**（上一轮 `BE-S2-8` 真漏过的盲点，这次提前堵上）/ 别名导入 `vp "…/voiceproto"` / **故意弄坏提取器**（报的是「扫描坏了，不是仓库干净了」，不是零命中）。
  ⚠️ 本条的复核**顺带撞出了 `BE-S0-7`** —— 见 §1。
- **BE-S1-5 ❌ 仍开着** —— `TypeAIAudioChunk` 全仓只有 `voiceproto/frames.go:22` 的声明与 `frames_test.go:764` 的引用，**没有生产者**。§8 已论证 v1 退役没有清掉它。
- **BE-S1-6 ✅** —— `internal/voicegateway/handler_audio_bench_test.go` **3 条** benchmark（`BenchmarkHandleAudioFrame` / `…DebugEmitted` / `BenchmarkAudioFrameDebugCall`）⇒「0 个 benchmark」作废，热路径 Debug 的成本**第一次可以量化**。
- **BE-S1-7 ✅ 已完成（`8c6792c`），而且真实情况比旧文写的严重** —— 旧文写「对应关系只靠读」（言下之意是读出来的关系是对的）。实际读下去发现**表里 `user.speech.end` 那一行从来没有被读过**：`startCollectTurn`（`handler.go:686`）自己直接调 `HandleClientControl`，失败时**手写死** `provider_control_failed` ⇒ 改那一行行为不变，它是**装饰**。修法因此不是「补一条测试」，而是**把两处收成一个实现点** `providerErrorCode(frameType)`，两个调用点都走它，并补 `handler_control_policy_test.go`（两条测试 + 一个记录「provider 收到了哪些帧」的 spy）。
  ⚠️ **我自己第一遍复核把它误判成「仍开着」。** 原因：我搜的是 `ProviderErrorPolicy` / `providerErrorStrategies` 这两个标识符，而新测试里**不出现**它们 —— 它靠 spy 记录收到的帧类型来断言。**「搜不到标识符」≠「没有这个行为」**，是 `git log` 里 `8c6792c` 的提交信息救回来的。
  ⇒ **教训写在这里：这份文件按「标识符」复核是不可靠的，要按「行为」复核**（或者直接读 `git log`，本仓的提交信息写得很细）。
- **BE-S1-8 ✅ 已完成（backend `e6458f3`）** —— 按源码逐份比对过，原文三处数字要改、一处判断要收回。
  **数字**：构造字面量的测试文件 **23 → 27**；出现总次数 **约 56 → 61**；`session/service_test.go` 的分档原文写「17 完整 + 5 空 + 1 单字段」，实测「**18 完整 + 4 空 + 1 单字段**」（总数同为 23）。原文「没有共享构造器」成立（唯一 `testConfig()` 在 `account/service_test.go:28`，包内私有）。
  **收回的一处判断**：原文写「加字段…`Validate()` 会在运行时让 23 个文件的测试一起红」—— **不成立**。加字段给的是零值，`Validate()` 只会因为「必填字段为零」红，不会因为「多了个字段」红；而这 35 处里**没有一处调用过 `Validate()`**（其中 5 处缺 `InternalAPIToken`、根本通不过，正说明没人调）。真实的失败模式是「**两份互相矛盾的定义并存，且没有任何东西会红**」。
  **原文说「已经漂移：未知（未逐份比对）」—— 逐份比对过了：已漂，四层。** ① 四套字段集（5/6/8/9 字段各一种）；② 一个时长三种写法：`SessionTicketTTL` 写成 `time.Minute`（17）/`60*time.Second`（11）/`60_000_000_000`（1，等值），`AccessTokenTTL` 写成 `2*time.Hour`（14）/`time.Hour`（3）；③ 同一个 voice gateway URL 两个主机名：`ws://example.test`（18）与 `ws://127.0.0.1:8081`（8）；④ 5 处缺 `InternalAPIToken`，通不过 `Validate()`。
  **作用面比原文小一半**：61 处里 26 处是「按需最小构造」（0-3 字段，表达「这个测试只关心这几项」），换成全字段基线会**扩大**测试语义 ⇒ 不动。该收敛的只有 35 处（跨 14 个文件）。
  **落点改了**：原文建议「抽 `config.TestConfig(t)` 到测试支撑包」——方向对，但不往 `config` 里塞 `testing` 依赖（那会让每个引用 `config` 的生产包都链上 `testing`），而是新建 `internal/configtest` —— 本仓 2026-09-26 已为 `BE-S2-6` 建过同形的测试支撑包 `internal/metricstest`。
  **判据**：`internal/config/config_fixture_test.go`（**第四个仓级守卫**，与另三个同包同理）—— `_test.go` 里的 `config.Config` 字面量不许拼出 ≥4 个字段。**阈值 4 落在真实空档上**：本仓字段数实测 `0,1,2,3,5,6,8,9`，**4 从来没有被用过**。判据在未修的树上是**确定性红、且红的正是那 35 处**（`git worktree` 到 `92103f4` 上跑，逐条点名文件:行:字段数:字段名）⇒ 是缺陷判据，不是特征化守卫。**边界**：只认 `config.Config`；`voicegateway.Config` 的 6 处 ≥4 字段**明确不管**（那是「测校验逻辑的多用例表」，字段不同**正是测试的目的**）。**变异 6 条**全咬（含两条反向：1 字段放行、给基线加字段放行；两层反空洞各自独立咬住）。
- **BE-S1-9 ✅ 前提不成立** —— `scripts/dev-check.sh` 的**第 3 步就是 `golangci-lint run ./...`**，本机跑出 **0 issues**（二进制在 `$(go env GOPATH)/bin`，由门禁自己 export）。⇒「未安装 ⇒ depguard 从未执行」已经不为真。**这一条要留意的不是安装，而是 CI 镜像里装没装** —— 那部分本次没复核。


**这两条「最该先做」的都已经做完了**（BE-S1-1 与 BE-S1-3，见上表 ✅）。旧文在这里写「BE-S1-1 是这张表里最该先做的一条」「BE-S1-3 是性价比最高的一条」，两条今天都只剩历史价值。值得记一笔的是：**它们被做掉的方式与旧文建议的完全一致**（照 `session_jobs` 抄租约；给手写解码器补 fuzz）—— 所以不是当时的判断错了，是**清单没跟上代码**。

⇒ 复核后剩下的 S1 只有 **BE-S1-5**（要动 v2 契约 ⇒ 得先拍）一条 —— `BE-S1-4` 已在第五轮做完（`67c37e0`）、`BE-S1-8` 已做完（`e6458f3`，见上）。**整张 backend 表只剩 2 条 ❌**：`BE-S1-5` 与本文件 S2 节的 `BE-S2-10`。

---

### 3. S2 — 卫生

| # | 条目 | 证据 | 2026-09-26 复核 |
|---|---|---|---|
| BE-S2-1 | `AGENTS.md:22` 把 `internal/corpus/` 写成 `internal/corpuss/`（照图找目录找不到） | `01` §4.3 | ✅ **已修**（全仓无 `corpuss`） |
| BE-S2-2 | 注释在描述别的东西：`handler.go:862-868` 的 doc 里并进 3 行描述**不存在的函数** `resolvedUserText` 的文字；`voiceduplex/volc_duplex.go:1153-1157` 是叠在 `FirstNonEmpty` 上的**第二段** doc（只有下面那段生效） | `02` §3.5 | ✅ **已完成**（backend `2aa629c`）—— 四处同形一次修完（清单只收了 2 处），并加了一条守卫，见下 |
| BE-S2-3 | `uplink_constants.go` 的注释说「两份 uplink 路径都用这个常量」，实际 `voiceduplex` 有自己的 `uplinkChunkBytes` | `01` §2.5 | ✅ **已完成**（backend `2aa629c`）—— ⚠️ 这半句**没有通用判据**（要读英语），见下 |
| BE-S2-4 | `httpserver` 的 `discovery` 手写了一份**部分**端点清单（列了 tts/hits/history/privacy/materials/topic_cards，没列 drill/corpus/content） | `01` §2.3 的边表 | ✅ **已完成**（backend `a003b16`） |
| BE-S2-5 | `/metrics` 是 7 个包手写文本的拼接，无 registry、无重名检测 | `01` §1 | ✅ **已完成**（backend `a003b16`） |
| BE-S2-6 | **指标发射器里只有一部分对 label 集合排序**，其余用 map range 直接渲染 → 每次 scrape 的**行序不同** | 见下方 | ✅ **已完成**（backend `8325795`；**判据本身在 `82bd06e` 修正**，见下） |
| BE-S2-9 | `tts` 的 `TestRouter_Stream_RouteHit` 断言全局计数器的**绝对值**（`expected 1 hit for voice-a, got 2`）—— 而票只记了 `:53` **一处**，实际是两处断言 | 本次复核新发现 | ✅ **已完成**（backend `92103f4`）—— 断言增量 **+ 门禁接 `-count>1`**（成本 21.0s→49.96s），见下 |
| BE-S2-7 | `test/` 是空目录（只有 `.gitkeep`）—— **而 README 的 Layout 表把它当成真实能力写着** | `04` §3.3 | ✅ **已完成**（backend `41cea71`）—— 删空壳 + 一条守卫盯住 Layout 表的路径列，见下 |
| BE-S2-10 | **README 的 Layout 表 Contents 列也在描述一个已经不在的仓库**（`internal/` 22→23 且漏了 `metricstest`；`api/` 25 paths→26；`migrations/` 0026→0028） | 本次复核新发现（做 `BE-S2-7` 时顺手重数） | ❌ **仍开着** —— 与 `BE-S2-7` 同一张表、同一个病；是散文不是路径，需先给每句定口径 |
| BE-S2-8 | 51 个环境变量没有一份清单（唯一来源是两个 `Config` 结构体） | `03` §4.1 | ✅ **已完成**（backend `182ec75`；**旧文的三个数字全错**，见下） |

**逐条证据（2026-09-26）**

- **BE-S2-1 ✅** —— 在 `AGENTS.md` 里搜 `corpuss` 零命中；第 96 行现在写的是 `corpus-seed`。
- **BE-S2-2 ❌ 仍开着，且比旧文更糟** —— 不是「一段孤立注释」，而是**并进了别的函数的文档注释**：`handler.go:862-868` 是 `extractServerASRText` 的 doc，其中 3 行（「resolvedUserText returns the user's utterance…」）描述的是**另一个函数**，中间还空了一行；而全仓 **没有 `resolvedUserText` 的定义**（只有这一处注释命中，`grep "func.*resolvedUserText"` 零命中）。
  **2026-09-27 补第三条（同形，只是不是「缺失」而是「重复」）**：`voiceduplex/volc_duplex.go:1153-1157` 与 `:1159-1163` 是**两段** `FirstNonEmpty` 文档注释，中间隔一个空行；两段都以 "FirstNonEmpty returns the first value with any non-space content." 开头，第二段 `Exported because the smoke probes in the PoC package…` 才是与代码对得上的那个，而第一段那句 `both halves of the package this was split out of` 说的是这次拆分**之前**的形状。⇒ 同一种病在同一天的第 6 次：**注释把读者带到一个不存在的世界，而代码看起来完全正常**。
  ✅ **已完成（backend `2aa629c`）—— 而且按源码重数后是四处，不是三处。** 新守卫（见下）在真仓库命中 **4 处**：清单写的两处，加上它顺手找出的两处同形 —— `internal/corpus/service.go:305`（`BatchAccept` 的一行 doc 粘在 `const dedupeScanLimit` 上，而 `BatchAccept` 在别处已有 doc）与 `cmd/gen-eval-samples/main.go:23`（`sceneFrames` 的 doc 粘在 `type sceneFrame` 上，而 12 行之下的 `func sceneFrames` 没有 doc）。**这四处是「判据一次扫出来的清单」，不再是抽样**。
  修法：`handler.go` 那 3 行**删掉**（它描述的行为真实存在，但已内联在 `user.speech.end` 分支里，且那里有自己的注释 ⇒ 化石）；`volc_duplex.go` 删第一段（保留与现状相符的第二段）；`corpus/service.go` 删那行、把「from one session」并入 `BatchAccept` 自己的 doc；`main.go` 整段**移回** `func sceneFrames` 上方。
- **BE-S2-3 ❌ 仍开着** —— `voicegateway/uplink_constants.go:12` 是 `const UplinkChunkBytes = 640`，`voiceduplex/volc_duplex.go:956-957` 另有 `uplinkChunkBytes`（注释「20ms of 16 kHz mono s16le (640 bytes)」），数值一致但**改一处不会让另一处红**。
  ✅ **已完成（backend `2aa629c`）** —— 但**这半条没有通用判据，而且我判断它不该有**：注释里的这句断言（「both uplink paths … use this size」）要判断真伪必须读英语，不可判定；能判定的那半边（doc 附着）已经做成了守卫（下一条）。所以这一半只做了两件事：① 把注释改成准确的说法（`provider_dev_echo.go` 用它；`voiceduplex` 有自己的一份、值相同、无关联）；② 把**最该被看见**的那句直接写进注释 —— 「改这个常量不会让那边编译失败，也不会让它的测试红」。**一句不能自动验证的断言，只能靠写得更难误读来防，这是它和 BE-S2-2 的本质差别。**
- **BE-S2-4 ✅ 已完成**（backend `a003b16`）—— `discovery` 不再手写：`publicEndpoints` 读 `engine.Routes()`，按 `apiPrefix` 后的首段分组、组内排序。**并且顺带关掉一个披露面**：旧表把 `/internal/v1/tts/synthesize` 与 `/internal/v1/voicegateway/hits` 写在了一个**不需要任何令牌**的端点上，现在内部面按前缀整体扣掉。
  ⚠️ **这一步我最初判定「零消费方」是错的**：我用 `grep -v _test.go` 检索，正好把消费方筛掉了 —— 它是一条测试（`internal/session/http_test.go` 的 `TestOpenAPIDiscoveryEndpoints`，断言 `discovery["openapi"] == "/openapi.yaml"`），已同步到新形状。**与 BE-S1-7 同型：「搜不到标识符」≠「没有这个行为」。**
- **BE-S2-5 ✅ 已完成**（backend `a003b16`）—— `/metrics` 走 `metricsRegistry`：一处登记；`serveMetrics` 只调 `renderMetrics()`。三条守卫：源码树扫描比对注册表（带反空洞下限）、同一族指标不得被两个包渲染、登记了必须真被送出。
- **BE-S2-6 ✅ 已完成**（backend `8325795`）—— 5 处补排序，见下表。
  ⚠️ **但上一轮的判据是错的，已在 `82bd06e` 修正**：它把「行序稳定」和「**数值**稳定」混在一起（渲染 17 次要求逐字节相同），而同包其它测试真的在跑 refine 管线 ⇒ 整包跑时红、单跑时绿，**上一轮它绿是运气**。现在只断言标签集合**按升序出现**，数值丢弃；5 个包共用 `internal/metricstest.LabelSetsIn`。
- **BE-S2-9 ✅ 已完成（backend `92103f4`）—— 数字要收回一处，而票里点名要判的那件事判了：接。**
  **① 数字：** 票写「`router_test.go:53` 一处」，实际是**两处断言**（`:52` 的条件 + `:55` 的条件，报错落在 `:53`/`:56`）。
  **② 全集这次由命令给，不由人肉抽样给：** `go test -count=2 ./...` 全仓**只红这一个测试** —— 这是这一系列第一次用可执行的跑法、而不是读源码，来定「一共几处」。
  **③ 撞出第二个同形单例：** `internal/orchestrator/metrics.go:22` 的 `var globalMetrics`。它在 `-count=2` 下**不红**，而原因必须分清 —— 它有 `Reset()`（注释写明「仅用于测试」），三条测试都 `t.Cleanup(globalMetrics.Reset)`。⇒ 它是本仓对这个病的**第一个既有解**，不是漏网。同一包内还有第二种解：两条 `RouteMiss` 测试用的是 **delta 写法**（`beforeMisses` + 断言增量）。所以本包内三种写法并存，**delta 2 : 绝对值 1**。
  **口径：选 delta，不选 Reset。** delta 不改生产代码 —— `tts.Metrics` 用 `atomic` + `map`，给它加 `Reset()` 要重建 map，而 `PrometheusMetrics()` 正在读同一个 map，那是把一条测试问题变成生产代码的并发面；且 delta 是**自足**的（不依赖「每条测试都记得 Cleanup」，也不依赖执行顺序）。改完三种写法收敛成一种。
  **门禁接 `-count>1`（票里点名要判的）：判决 = 接。** 它与 `-race` 是同一个盲区的两半 —— **门禁的选项也是覆盖范围**：数据竞争 `-count=1` 照不到，跨测试残留 `-race` 也照不到，只有跑第二遍才照得到。形状：`dev-check.sh` 第 4 步改 `go test -race -count=2 ./...`；`check-gate.sh` 第 2 条断言从 1 条扩成 2 条（`-race` + `-count=N, N>=2`），并把原来的 `elif` 链拆成两段独立 `if`，让两条断言各自报出、各自计数。**成本（实测）：21.0s → 49.955s**（大头是 `voiceduplex` 31.6s 与 `voicegateway` 11.7s）—— 要减这个成本，正确的做法是找一个更便宜的等价手段，不是删掉这一个词。
  **⚠️ 第一版踩了本仓 house rule：** 给测试加了 4 行 inline rationale（含行号），而 `AGENTS.md` 规则 10 / `CLAUDE.md` 规则 7 明确禁止「为新改的代码加 doc comment 或 inline rationale」—— **理由属于 commit body**，删掉。这件事反过来说明上面那条为什么必须做：「为什么必须是 delta」这条知识在代码里**无处可放**，只剩机械保护。
  **哪些是真红、哪些是特征化（不假装全部红过）：** `router_test.go` 的修复**改之前确定性红**（`-count=2`）；`check-gate.sh` 的第 4 条断言是**特征化守卫** —— 改前门禁里没有 `-count=2` 这个词，它没有「改前红」的状态，牙**只能靠变异证明**。**6 条变异全部咬住**，含一条**反向**（门禁写 `-count=3` 必须放行 ⇒ 证明判的是「跑不止一遍」不是「字面等于 2」）和两条守卫自身的（测试步骤换成 `go vet` ⇒ 报「解析失效」；去掉 `-race` ⇒ 第 1 条独立咬住，证明拆 `if` 没弄坏旧断言）。
- **BE-S2-7 ✅ 已完成（backend `41cea71`）—— 缺口比票里写的多一处，而票里有两处判断要收回。**
  票里只记了「`ls test/` 只有 `.gitkeep`（1 字节）」。**同一件事在 README 里还有第二个地方**：
  `README.md:124` 写着 `| eval/, test/ | eval datasets and test support |` —— 目录表把 `test/`
  当成真实存在的能力。清单一个字没提这一处。而 `test/` 在别处**没有任何引用**（`scripts/`、
  `.github/`、`.gitignore`、`go.mod` 全仓零命中）⇒ 一个读者信了这张表，会去找跨层测试，
  然后找到一个空目录。
  两处口径修正：
  1. 「backend 的替代品是一批 `scripts/smoke-*.sh`（**12 个**）」—— 实测 **8 个**。
  2. 「跨层验证是有的，只是**不在 Go 测试里**」—— 只对了一半。跨层测试在 `go test` 里很多：
     `internal/session/http_test.go` 一个文件就 import **7** 个 internal 包，起真的
     `account.Service` / `session.Service` / `httpserver.Server` 走 `httptest` 真发请求；
     `internal/voicegateway/handler_b14_test.go` 是另一处。**成立的那一半**是「需要真进程或
     真凭证的跨层验证不在 `go test` 里」：8 个 smoke 脚本 + `cmd/integration-voice-gateway`
     （在 `httptest` 上起生产 `Handler`、用真 websocket 客户端驱动，**但它是 `main` 包，
     不进 `go test`**）。README 表下现在写下了这件事。
  **改法：删空壳，不补内容。** 建一个 `test/` 等于给跨层测试开第三个家 ——
  `internal/<pkg>/*_test.go` 已经是本仓既有的、idiomatic 的家（测试贴着被测代码），
  需要真进程/凭证时的家在 `scripts/smoke-*.sh` 与 `cmd/integration-voice-gateway`。
  第三个家只会重复第一个家在做的事，而本仓刚花了一整批把「两处各自列举同一件东西」
  收敛成一处（`BE-S2-4` / `BE-S2-5` / `BE-S2-6`）。
  ⚠️ **同一个判断在 iOS 侧已经做过一次**：`fluentwork-ios` 的 `Modules/.gitkeep`、
  `Resources/.gitkeep`、`Services/.gitkeep` **三个空壳在 `80c8209`（2026-09-25）被一起删掉**。
  ⇒ 本轮与那次是同一个选择，不是新发明的口径。
  **守卫**：`internal/config/readme_layout_test.go`（第三个仓级守卫，寄居理由与前两个相同）——
  README 的 Layout 表 `Path` 列点名的每个路径必须真实存在；以 `/` 结尾的必须**有内容**
  （只有一个 `.gitkeep` 不算）。判据今天确定性红、且只红 `test/` 这一条；修完绿。
  **变异 6 条**：写回 `test/`（不存在）→ 咬住；复原成原始空壳 → 咬住（原样复现）；
  反向给 `test/` 一个真文件 → **放行**（证明它报的是「空壳」不是「多了行」）；
  把一行写成目录而实际是文件 → 咬住；`## Layout` 改成 `## Layouts` → **反空洞报
  「parsed 0 paths … proved nothing」而不是绿**；把一行的首个 `|` 去掉 → 静默通过
  （这是边界不是洞：那行不再是一张声明，同一条边界在正控里钉着）。
- **BE-S2-10 ❌ 仍开着（本次复核新发现，做 `BE-S2-7` 时顺手重数）** —— **同一张 Layout 表的
  Contents 列也在描述一个已经不在的仓库**：
  `internal/` 写 **22 packages**，实测 **23**（漏的正是 `metricstest`，昨天为 `BE-S2-6` 新建）；
  `api/` 写 **25 paths**，实测 **26**；`migrations/` 写 **0001–0026**，实测 **0001–0028**。
  反例（复核过、是对的）：`cmd/` 20 个 `main` 包 ✅；`voicegateway`（25 files）✅（不含 `_test.go`）。
  ⇒ **本提交不修**：这些是散文不是路径，验证它们要先给每一句定口径（「files」算不算 `_test.go`？
  「packages」是目录还是包？），一次改动里混进两条口径会让两边都不可复核。
  **耐久修法**（写进这条票，别丢）：把 `readme_layout_test.go` 从「路径列」扩到 Contents 列里
  **可判定**的部分 —— `internal/` 那张包名清单是可判定的（集合相等），openapi/migrations 的
  计数不是（没有哪处能一次答清）。
- **BE-S2-8 ✅ 已完成**（backend `182ec75`）—— 旧文那句话里**三个数字全错**：「读 42 / 声明 27 / 缺 21」。
  按源码重数是 **读 70（16 个文件）/ 提及 41 / 缺 35**。差在两处口径，都很具体：
  1. 旧文只数了两个 `Config` 结构体，漏掉 `cmd/app-server`、`cmd/worker`、`cmd/voice-gateway` 与 5 个 POC/smoke 工具；
  2. 旧文把「读过一次」当成了「读过」，而 `os.Getenv(key)` 这种**键来自形参**的读取辅助函数（`envOr`/`intOr`/`envFirst`/…）在文本层面是看不见的，反过来 —— `range endpointEnvVars` 那种具名键表也一样。

  **判据三条**（`internal/config/env_declaration_test.go`，同一次扫描）：
  - **准入**：某个 surface 读到的每个变量必须出现在**它自己的**模板里（或其 shared 模板里）。合并成「三个文件里有一个提到就算」不够 —— 变异 M2 证明过：把 `MYSQL_DSN` 的声明挪进 `voice-gateway.env.example`，弱判据会绿，这条会红。
  - **分类**：任何读环境变量的目录必须归入某个 surface。新读者落在没人管的目录里，第一条判据永远不会看它。
  - **登记**：把键转发给 `os.Getenv` 的辅助函数必须在 `envReaderFuncs` 里。**漏登记不会让任何东西红** —— 传进去的键会被当作「绑定的名字」跳过，判据照旧通过，只是查得更少。

  提取器认四种形状（都真实存在）：字面量 / 字符串常量 / `[]struct{key string}` 键表（`durationStrict(knob.key, …)`）/ 具名 `[]string` 键表（`for _, env := range endpointEnvVars`）。每个形状一个正向控制点，所以「某个形状失效」报的是形状，不是变量缺失。无法解析的键参数**报错而不是静默跳过** —— 看不见的键就是查不了的键。

  ⚠️ **口径两处，写下来免得下一个人以为判据松了**：`KEY=`（值为空）算声明，因为本仓每个读取器都把空值当未设置（`envOr`/`intOr`/`boolOr`/`durationOr`/`durationStrict`/`envFirst`/`envTruthy` 逐个核过）；`# KEY=value` 也算，因为 `volc.env.example` 的既有风格就是「注释里写编译期默认值」。散文式提及**不算** —— `# fill ARK_API_KEY(_DEV)` 那种句子不会让判据变松。

  **修法**：35 个键全部按既有风格补成「注释 + 编译期默认值」。它们每一个都有编译期默认值（逐个核过），所以这是**纯发现性**缺口，补注释是忠实的，不是掩盖。实证：`load_env_file configs/app-server.env.example` 改动前后加载出的环境**逐字节相同（10 个键）**。

  ⚠️ **变异 M3 第一次存活，抓出的是提取器本身的盲点**：它只遍历 `file.Decls` 里的 `*ast.FuncDecl`，**漏掉包级变量初始化** —— `var x = os.Getenv("…")` 是一次读，而扫描看不见它。修了提取器（包级声明走同一条 `inspectForReads`）之后 M3 才咬住。
  这是「用变异验证判据」而不是「用判据证明改动」的又一例：**如果只跑一遍门禁看它变绿，这个盲点会原封不动留在仓里**，而且外观与一条真判据完全一样。

  ⚠️ 顺带纠一处旧记录：`volc.env.example` 写 `# VOLC_DUPLEX_MODEL=1.2.6.0`，而 `internal/voiceduplex` 自己的默认是 `1.2.6.1`。**两个默认值本来就不同**（POC/smoke 工具走 1.2.6.0，duplex 客户端走 1.2.6.1），所以改的是注释（把分歧写清楚），不是值。



**BE-S2-4 值得单独说（已解）**：`discovery` 原来是一个**手写的路由表副本**，而路由表本身是 nil-gated 动态挂载的（`httpserver/server.go:93-129`）—— 所以「服务有哪些端点」这个问题，**在代码里没有一个地方能一次回答清楚**。现在它读路由表本身（`engine.Routes()`，gin 的官方入口），手抄的可能从根上被去掉。它和 iOS 的 `TransportEventRouter` 那个对照（那边是一张**静态可断言**的表）现在不再成立了：这边变成**运行时派生**的表。
两条判据：`TestDiscoveryListsEveryMountedRoute`（挂一条**没有任何模块注册**的探针路由，手写清单必然漏它）、`TestDiscoveryWithholdsTheInternalSurface`。

**BE-S2-6 的实测明细（2026-09-26 复核）**：

先纠正旧文的计数口径：**7 个发射器里只有 6 个带 label map**（`review/metrics.go` 只有两个无标签计数器），
带标签的渲染块共 8 处，**修复前只有 3 处排序**。旧文把 `topic` 记成 ✅ **是错的** —— ✅ 只对了它的
`dismissals`，同一个函数里紧挨着的 `skips` 是裸 map range。**旧文的「2 / 7」既数错了文件，也漏掉了主题那半边。**

| 带标签的渲染块 | 修复前 | 复核注 |
|---|---|---|
| `corpus` `deprecated_post_favorite_total{user_agent}` | ✅ | |
| `corpus` `block_feedback_total{reason}` | ✅ | |
| `topic` `dismissed_total{reason}` | ✅ | |
| `topic` `gen_skipped_total{reason}` | ❌ | **旧文把它所在的文件记成 ✅** —— 讽刺的是同一函数里两个 map，一个排了、一个没排 |
| `drill` `state_transition_total{from,to}` | ❌ | 已经收进 `keys` 切片，但**没有排序**，注释写着 `stable-ish: unsorted ok for tests Contains` ⇒ 是不确定序，后果一样 |
| `materials` `refine_status_transition_total{from,to}` | ❌ | 裸 `for key, n := range transitions` |
| `tts` `route_hits_total{voice_id}` | ❌ | |
| `account` `tombstone_inserted_total{entity_type}` | ❌ | |

**修复（backend `8325795`）**：5 处照 `corpus` 的既有形状 —— 先收 key、`sort.Strings`、再渲染。
判据是旧文自己写的那句（「同一状态渲染两次必须逐字节相同」）：5 个包各一条
`TestMetricsRenderingIsReproducible`，改之前五条全红。

⚠️ **那句判据本身写错了，`82bd06e` 已修正** —— 它把两件事混在一起：

1. **行序**要稳定（这才是要的，也是两条 scrape 能 diff 的前提）；
2. **数值**要稳定（**不是要的，而且不成立**）—— 同包其它测试真的在跑 refine 管线，
   `material_refine_timeout_total 2`、`processing->failed 13` 这类数字在两次渲染之间就被推高了。

后果很具体：**整包跑（`go test ./...`）红、`-run` 单跑绿 ⇒ 上一轮那条绿是运气。**
现在只断言标签集合**按升序出现**（数值丢弃），仍保留 8 次渲染（一次 map 迭代碰巧有序的概率
是 1/6，8 次是 (1/6)^8），5 个包共用 `internal/metricstest.LabelSetsIn`。

⚠️ **留下一条更值得记的教训**：这条缺陷能被留下来，是因为**既有判据只做 `strings.Contains`** ——
行序从来没有东西守。所以新判据每条前面加了一道**反空洞**断言（渲染出的 label 行数必须不少于塞进去的键数）。
**否则哪天标签块不再渲染，这条测试会比较两个占位结果而静默通过。**

**确定性：【实测】。**

---

### 4. 反过来：这些看起来是问题，但**不要改**

写下来是为了防止下一个人（包括我）把它们当缺陷修掉。

| 看起来像 | 实际是 | 证据 |
|---|---|---|
| `writableOutbound` 与 `writeProviderOutbound` 各自枚举一遍字段 | **刻意分离**。漏谓词 = 多抢一次锁（无害）；漏写侧 = 字段静默不上线（丢数据）。合并会把无害的漏换成有害的漏 —— 且有反射测试护着 | `handler_write_bound_test.go:237-244` |
| 未知控制帧不回错误，只忽略 + 计数 | **与 iOS 对称**，且两侧都记录了那次事故（`unsupported_frame` → iOS `.failed` → 杀会话） | `handler_control.go:465-486` |
| `silentBecause` 是必填字段，沉默也要写理由 | **刻意**。「silence is the choice that needs justifying, because the failure it hides is a frame the client thinks succeeded」 | `handler_control.go:41-45` |
| `defaultIdleTimeout` 不回收空闲用户 | **明确声明未实现**，并指向 `77_` P1-17。它只做半开连接探测 | `config.go:16-28` |
| `RescueEnabled` 生产默认关 | **数据诚实性**：宁可不记，也不记一条用户没经历过的救援 | `config.go:57-65` |
| 救援音频与 AI 音频不共享流控制 | `103_/04_` §七已论证（批处理 vs 流式；时机控制位置不同） | `103_/04_` |
| `Refine` 对非 `queued` 行直接 no-op | **幂等守卫**，配合 `MarkProcessing` 的 CAS。问题不在它，在缺租约（BE-S1-1） | `materials/refiner.go:42-51` |
| `providerErrorStrategies` 只有 4 行 | 另外 3 个 handler（`ping`/`session.end`/`auth`）**不经过 provider**，所以不需要失败策略 | `handler_control.go:112-120` |

---

### 5. 下一步做什么（2026-09-28 ——「三件事」这个格式在本轮退休）

⚠️ **前九版的前三条也都做完了**：旧版 BE-S1-1 ✅ `25a015e`、BE-S1-2 + BE-S1-3 ✅ `01a5e55` / `cb99ba3`；
二次重排的第 1 顺位 BE-S2-4 + BE-S2-5 ✅ `a003b16`；三次重排的第 1 顺位 BE-S2-8 ✅ `182ec75`；
四次重排的第 1 顺位 BE-S1-4 ✅ `67c37e0`；五次重排的第 1 顺位 BE-S0-7 ✅ `c5cd0f5`；
七次重排的第 1、2 位 ✅ `4f5930d` / `ca58de9`；八次重排的第 1 顺位 `BE-S0-9` ✅ `753dbd6`；
**九次重排的第 1 顺位 `BE-S2-2` + `BE-S2-3` ✅ `2aa629c`**；
**十次重排的第 1 顺位 `BE-S2-7` ✅ `41cea71`**；
**十一次重排的第 1 顺位 `BE-S2-9` ✅ `92103f4`**；
**十二次重排的第 1 顺位 `BE-S1-8` ✅ `e6458f3`**。

⚠️ **「三件事」这个格式在本轮退休 —— 不是因为它不好，是因为它没有东西可装了。**
上一版在这里留下的问题是「要不要一次把剩下三条做完，让它自然退休」；本轮给出答案：
做完 `BE-S1-8` 之后，全清单**只剩 2 条 ❌**，「挑三件」不再是一次筛选，三位本身都凑不满。
⇒ **此后不再有「第 N 次重排」**，本节改成「下一步做什么」。

⚠️ **十二次重排的第 1 位 `BE-S1-8` 本轮做完了，全清单只剩 2 条 ❌**：
`BE-S2-10` / `BE-S1-5`。**下面不再给顺位** —— 只剩两条时，「先做哪一条」已经不是优先级
问题，而是被两条各自的**前置**决定的。

1. **`BE-S2-10`（README Layout 表的 Contents 列）** —— 是 `BE-S2-7` 的同一个病在**同一张表**的另一列上：`internal/` 写 22 packages 实测 **23**（漏的正是为 `BE-S2-6` 新建的 `metricstest`）、`api/` 写 25 paths 实测 **26**、`migrations/` 写 0001–0026 实测 **0001–0028**。**它的前置是「定口径」，不是「拍板」**：「files」算不算 `_test.go`？「packages」是目录还是包？定完之后，耐久修法是把 `readme_layout_test.go` 从**路径列**扩到 Contents 列里**可判定**的部分 —— `internal/` 那张包名清单可判定，openapi/migrations 的计数不可判定。**这一条可以接着做，不需要等任何人。**
2. **`BE-S1-5`（`ai.audio.chunk` 那张死面）** —— 全表**价值最低**的一条，也是唯一「动作只有一个方向」的。**它的前置是「有人拍方向」**：要动 v2 契约（`ai.audio.chunk` 在 v1 已退役、在 v2 仍是一张无生产者的死面）。**拍之前不开工。**

**⚠️ 十二次重排的三位本轮做完了第 1 位**（原文保留在下方，**不要按它推下一轮**）：

> 1. **`BE-S1-8`（`config.Config` 夹具复制到 23 个文件、约 56 处）** —— 升到第 1 位：它是剩下三条里**唯一还带着真实维护成本**的一条（一处字段变更要改 23 个文件，且改漏了不会有任何东西变红），且低风险、不需拍板、一次做完。它在前十一版里**从没进过第 1 位**。
> 2. **`BE-S2-10`（README Layout 表的 Contents 列）** —— `BE-S2-7` 的同一个病在**同一张表**的另一列上。排第 2 只因为它**需要先定口径**（「files」算不算 `_test.go`？「packages」是目录还是包？），而定完口径之后，它的耐久修法是把 `readme_layout_test.go` 从路径列扩到 Contents 列里**可判定**的部分 —— **两件事一起做才不白做**。
> 3. **`BE-S1-5`（`ai.audio.chunk` 那张死面）** —— 它是全表**价值最低**的一条（价值：低），也是唯一「动作只有一个方向」的；它从三位之外进到第 3 位，**只因为其余都做完了**。要动 v2 契约 ⇒ 得先拍。

- 第 1 位 `BE-S1-8` ✅ **`e6458f3`** —— **数字三处要改**（23→**27** 文件、约 56→**61** 处、分档「17 完整 + 5 空」→**「18 完整 + 4 空」**）；**判断一处要收回**：原文「加字段…`Validate()` 会让 23 个文件的测试一起红」**不成立**（加字段给的是零值；而这 35 处里**没有一处调用过** `Validate()`）；原文写「**已经漂移：未知（未逐份比对）**」—— 逐份比对后是**已漂**（四套字段集 / 一个时长三种写法 / 同一个 URL 两个主机名 / 5 处通不过 `Validate()`）。**原文那句「它的失败模式是『23 个文件一起报配置错误』，吵闹而不危险」只对了一半** —— 吵闹的是「一起报错」这个想象，真实的失败模式是**两份互相矛盾的定义并存，且不会有任何东西红**。**作用面比原文小一半**：61 处里 26 处是「按需最小构造」，换成全字段基线会**扩大**测试语义 ⇒ 该收敛的只有 35 处。**落点也改了**：不往 `config` 里塞 `testing` 依赖，而是新建 `internal/configtest`（本仓已有 `internal/metricstest` 这个先例）。
- 第 2 位 `BE-S2-10`、第 3 位 `BE-S1-5` 未动（见上）。

**⚠️ 十一次重排的三位本轮做完了第 1 位**（原文保留在下方，**不要按它推下一轮**）：

> 1. **`BE-S2-9`（`tts` 断言全局计数器绝对值）** —— 升到第 1 位，两条理由：① **成本最低、且可以被一条命令立刻证伪**（`go test -count=2 ./internal/content/tts/`）；② 它和**已完成的「门禁接 `-race`」是同一个根因** —— 门禁只跑一种配置，于是「换一种跑法才会红」的缺陷有一整类，`-race` 只关掉一半，剩下的一半正是它。做它的时候**顺带要判一件事**：门禁该不该加一条 `-count>1`（一条命令就能照到这一类）。
> 2. **`BE-S1-8`（`config.Config` 夹具复制到 23 个文件、约 56 处）** —— 它是剩下四条里**唯一还带着真实维护成本**的一条：一处字段变更要改 23 个文件，且改漏了不会有任何东西变红。低风险、不需拍板、一次做完。（它在前十版里从没进过三位。）
> 3. **`BE-S2-10`（README Layout 表的 Contents 列）** —— 本轮刚撞出来的，是 `BE-S2-7` 的同一个病在**同一张表**的另一列上。排在最后只因为它**需要先定口径**（「files」算不算 `_test.go`？「packages」是目录还是包？），而定完口径之后，它的耐久修法是把刚写的 `readme_layout_test.go` 从路径列扩到 Contents 列里**可判定**的部分 —— **两件事一起做才不白做**。
>
> **为什么 `BE-S1-5` 不在这三位里**：它要动 v2 契约（`ai.audio.chunk` 那张死面），得先拍；而且它是四条里唯一「动作只有一个方向、收益也是全表最低」的（价值：低）。

- 第 1 位 `BE-S2-9` ✅ **`92103f4`** —— 断言增量 + **门禁接 `-count>1`**。数字要收回一处（票写 `:53` 一处，实际 `:52`/`:55` **两处**）；**全集这次由命令给而不是人肉抽样给**（`go test -count=2 ./...` 全仓只红这一个测试）；撞出**第二个同形单例** `orchestrator` 的 `globalMetrics` —— 它不红是因为**本仓已有第一种解**（`Reset()` + `t.Cleanup`），而同包的 `routeHits` 用的是第二种解（delta），**本包内 delta 2 : 绝对值 1**。选 delta 的理由：不改生产代码（给 `atomic`+`map` 加 `Reset()` 会与 `PrometheusMetrics()` 的 map 读构成新的并发面）。⚠️ **第一版踩了本仓 house rule**：`AGENTS.md` 规则 10 禁止为新改代码加 inline rationale ⇒ 注释删掉、理由进 commit body —— **这恰好证明门禁是唯一防线**。
- 第 2、3 位 `BE-S1-8` / `BE-S2-10` 未动，本轮升到第 1、2；`BE-S1-5` 从三位之外进第 3 位（**只因为其余都做完了**）。

**⚠️ 十次重排的三位本轮做完了第 1 位**（原文保留在下方，**不要按它推下一轮**）：

> 1. **`BE-S2-7`（`test/` 是空目录）** —— 唯一一条答案只能二选一的收尾题：要么把那层集成测试的落点补上，要么删掉空壳。它排第 1 是因为**它今天就在骗人**（目录存在意味着「集成测试在这」，实际 1 字节），而且它是这四条里唯一**不依赖任何判断**的：删或补，都是一次改动。
> 2. **`BE-S2-9`（`tts` 断言全局计数器绝对值）** —— 它和**已完成的「门禁接 `-race`」是同一个根因**：门禁只跑一种配置，于是「换一种跑法才会红」的缺陷有一整类。`-race` 接了，这个盲区只关掉一半；剩下的一半正是它（`-count>1` 必红），而门禁只需要一条 `-count>1` 就会照到。**成本最低、且可以被一条命令立刻证伪**：`go test -count=2 ./internal/content/tts/`。它不影响生产，所以排在「在骗人」的那条后面。
> 3. **`BE-S1-8`（`config.Config` 夹具复制到 23 个文件、约 56 处）** —— 把它排进来是因为它**在前九版里从没进过三位**（一直被「`-race` 挡着」「S2 更急」压着），而它现在是剩下四条里**唯一还带着真实维护成本**的一条：一处字段变更要改 23 个文件，且改漏了不会有任何东西变红。低风险、不需拍板、一次做完。
>
> **为什么 `BE-S1-5` 不在这三位里**：它要动 v2 契约（`ai.audio.chunk` 那张死面），得先拍；而且它是四条里唯一「动作只有一个方向、收益也是全表最低」的（价值：低）。
>
> **为什么「S2 排在最前面」不再需要解释**：上一版那段话的前提是「S1 剩下的两条更贵、更需要前提」。现在 `BE-S1-8` 进了第 3 位 —— **前提是它，不是级别**。

- 第 1 位 `BE-S2-7` ✅ **`41cea71`** —— **删空壳，不补内容**，因为跨层测试在本仓已经有**两个真实的家**（`internal/<pkg>/*_test.go` 里走 `httptest` 的跨层测试、`scripts/smoke-*.sh` + `cmd/integration-voice-gateway`）—— 第三个家只会重复第一个家在做的事，而本仓刚花一整批把「两处各自列举同一件东西」收敛成一处。**票里两处判断被收回**：smoke 脚本不是 12 个而是 **8** 个；「跨层验证不在 Go 测试里」**只对了一半**（`internal/session/http_test.go` 一个文件就起真的 service + server 走 httptest）。**并且挖出新的一条 `BE-S2-10`。**⚠️ 同一个选择 iOS 侧已经做过：`Modules/` `Resources/` `Services/` 三个空壳在 `80c8209`（2026-09-25）被一起删掉。
- 第 2、3 位 `BE-S2-9` / `BE-S1-8` 未动，本轮升到第 1、2；`BE-S2-10` 进第 3 位。


**⚠️ 九次重排的第 1 位本轮做完了**（原文保留在下方，**不要按它推下一轮**）：

> 1. **`BE-S2-2` + `BE-S2-3`（注释在描述别的东西）** —— 两条同源，且 `BE-S2-2` **比旧文写的更糟**：不是一段孤立注释，而是并进了 `extractServerASRText` 的文档注释里（3 行描述一个全仓不存在的函数）。**2026-09-27 又发现第三条同形的，就在本轮改的文件里**：`voiceduplex/volc_duplex.go:1153-1157` 是一段**孤立的 `FirstNonEmpty` 文档注释**，与它下面（`:1159-1163`）**真正的**那段重复了「returns the first value」与「Exported because…」，而孤立的那个理由（"the two halves of the package this was split out of"）与代码现状对不上。它们不是功能缺陷，但它们**主动误导读者** —— 而本清单一天里被同一种病咬了五次。

- 第 1 位 `BE-S2-2` + `BE-S2-3` ✅ **`2aa629c`** —— **修法比票里写的宽，而且「三处」这个数字是错的**：写了一条守卫后，真仓库一次扫出 **4 处**同形（清单只收了 2 处，判据又找出 `corpus/service.go:305` 与 `cmd/gen-eval-samples/main.go:23`）。**这是「抽样 vs 全量」第一次在清单自身上显形** —— 前几版所有条目都靠人肉抽样，这一条第一次由判据给出全集。
- 第 2、3 位 `BE-S2-7` / `BE-S2-9` 未动，本轮升到第 1、2。

**⚠️ 八次重排的第 1 位也已做完**（原文保留在下方，**不要按它推下一轮**）：

> 1. **`BE-S0-9`（一轮超时把整条 duplex 打死，而日志报成一次普通的 `timeout`）** —— 本轮新发现，读侧。它能排第 1 位有两条硬的：① **生产链路**上带着已知风险在跑（`TurnOutcomeTimeout` 是设计好的正常状态，所以「超时」这条路会真的走到），而它把会话打死后，下一轮的失败被归到别的原因上；② 它的复现是**确定性**的（不像 `BE-S0-8` 那样靠概率），所以判据好写、修好可验。**它卡的不是人手，是读模型的选择**（三条见 §1 专段）—— 而这三条的成本差一个数量级，所以**第一步不是挑方案，是挑第一步**：建议先做第 2 条（把「超时即会话死」显式标出来，让调用方的重连变成有意的动作），它是一个小改动，且无论后面选 1 还是 3 都不会白做。

- 第 1 位 `BE-S0-9` ✅ **`753dbd6`** —— 修法**变了，而且票里那句「三条成本差一个数量级」被推翻**：第 1 条与第 3 条其实是同一件事的两半（唯一 reader + 窗口退化成「消费者自己的等待」），合起来才是完整答案，收敛面只有 `recv` 一个函数、三处调用点一行都不用改。**没有取「先做第 2 条」那条建议** —— 第 2 条只是把死连接报准，而真正的修法成本并不比它高一个数量级。

**七次重排的第 1、2 位（更早那一版）也已做完**：

> 1. **`BE-S0-8`（轮边界打死整条 duplex）** …… 它是新的第 1 位……
> 2. **补上门禁的那条 `-race`** …… 但它未接线、门禁也没改……

- 第 1 位 `BE-S0-8` ✅ **`4f5930d`** —— 修法**变了**：不是「给泵打补丁」，而是把整条写路径收到会话级 ctx 上（9 个写点都走同一条路）。唯一原样采纳的是修法 1 的方向。
- 第 2 位（门禁接 `-race`）✅ **`ca58de9`** —— 第 4 步改 `go test -race ./...`，加第 8 步 `scripts/check-gate.sh`；CI 同步接上。八步门禁端到端 **27.3s**。

**为什么把 `BE-S1-5` / `BE-S1-8` 排除在三位之外**（⚠️ **这段是九次重排写的；十次重排已改判 —— `BE-S1-8` 进了第 3 位，理由见上**）：
`BE-S1-5` 要动 v2 契约（得先拍）；`BE-S1-8` 只是测试夹具收敛（把 `config.Config` 复制到 23 个文件的事收成一个 helper）—— 两条既不在骗人，也不带着风险在跑。

⚠️ **`BE-S2-9` 本轮从「排除」变成第 3 位。** 上一版挡它的理由是「`-count>1` 今天没有任何东西会走到，因为门禁不跑 `-count>1`」—— 那个事实没错，但它被用成了「所以不用做」。正确的读法是反过来的：**它是「门禁的选项」这个盲区的另一半，而且门禁只需要一条 `-count>1` 就会照到它**。第 2 位（接 `-race`）关掉的是这个盲区的一半。

**为什么那一版 S2 排在 S1 前面**（⚠️ **历史；十次重排时 `BE-S1-8` 已进第 3 位，见上**）：旧文的分级（S1 = 不做会在下一次同类缺陷上再花一遍时间；S2 = 只是让下一个人多读一会儿）本身没错，但**复核后剩下的 S1 恰好是最贵、最需要前提的**。分级答的是「不做会怎样」，不是「先做哪一个」。

**注意第 1 位把 S0 的排法打开了**：下面是 §5 原有的那段话，它是**上一轮**写的，
`BE-S0-7` 恰好是它的反例 —— 留在这里，不要按它去推下一轮。

> **`BE-S0-1` 已从这张表里出局**（2026-09-26 复核）：四仓代码都落地了，剩下真机跑与冻结产物摘要重核，见 §8。它当初被排除在「三件事」之外的理由是**它不能单独做**（需要协议版本、两侧同时改、先确认没有老客户端）—— 而这个理由后来被**一次专门排期**解决了。**它不是被塞进「三件事」里做掉的，是单独做掉的**，这个区分值得留着：S0 的条目不该为了「凑进三件事」而开工。

⚠️ **收回上面这段的一个推论。** 它隐含的是「S0 要等排期，所以不该进三件事」。但 `BE-S0-7` 说明：**S0 进三件事的条件不是「能不能一个人做完」，而是「它还带不带着已知风险在跑」** —— `BE-S0-7` 三种修法都不需要产品决定，只缺一个工程选择，而它每天都在生产链路上跑。分级答的是「不做会怎样」，`BE-S0-7` 的答案是「继续带着数据竞争跑」，这比 S1/S2 的任何一条都靠前。

**而且它确实一次就做完了**（`c5cd0f5`，一个提交、四条判据、七条变异、七步门禁）：从「进三件事」到「落地」之间没有出现任何需要排期的前置。对照 `BE-S0-1`（唯一需要两仓同时改的那条）—— 所以这条收得更准了：**卡的从来不是「S0」这个级别，是「需要第二方」这件事**。

---

### 6. 与 `103_tts_wss_architecture_audit/` 的关系

> ⚠️ `fluentwork-backend/docs/103_tts_wss_architecture_audit/` 已于 2026-09-25 随 `docs/` 删除。
> 本节保留的是**当时的对照结论**，其中的数字与章节引用已无法回查。

**两份文档不矛盾，但边界要说清楚。**

| | `103_` | `104_`（本系列） |
|---|---|---|
| 范围 | `internal/voicegateway` 的 TTS/WSS 一条链路 | 整个仓 |
| 结论 | 架构评级 **A-（90/100）可生产**；M1–M6 全部落地 | 见上 |
| 交付 | 里程碑验收（对照 `94_`–`99_` 设计） | 按架构轴分章 |

**两处需要读者自己判断的地方：**

1. **`103_` 的「所有测试通过（208+ 个）」是当时的计数。** 现在全仓是 **746 个 `func Test`**（`voicegateway` 单包 233 个）。这不是错误，是时点差异 —— 但它说明 `103_` 的数字不应被当作现状引用。

2. **`103_/04_` §八 说「编码统一度 95%」，唯一缺口是上行 640 常量。** 核实结果：`voicegateway/uplink_constants.go:12` 已有具名常量 `UplinkChunkBytes`，但 `voiceduplex/volc_duplex.go:958` 仍有**自己的** `uplinkChunkBytes = 640`。**所以缺口从「字面量重复」变成了「两份具名常量」** —— 数值一致，但改一处不会让另一处红。这是 BE-S2-3。

**一句话**：`103_` 回答的是「TTS/WSS 那条链路做到设计了没有」，答案是「做到了」。本系列回答的是「这个仓作为整体，哪里会出事」，答案是 §1–§3。

---

### 7. 与 iOS 侧的对应

同一形状在两侧的出现情况（这是本次工作的主要产出之一）：

| 形状 | iOS | backend |
|---|---|---|
| 未知帧要宽容 | ✅ 已做（`316-332`） | ✅ 已做（`465-486`） |
| 路由表要能断言 | ✅ 字典 + 生产工厂驱动 | ✅ **已修** —— 分派改成 `controlDispatch` 查表，判据双向比对（BE-S1-4，`67c37e0`） |
| 「判据必须真的能红」 | ❌ 8 份 `waitUntil` 已漂移 5 类 | ✅ `check-defect-discipline.sh` 主动修掉「永远通过」的写法 |
| 两处枚举会漂移 | ❌ 正在一张票一张票地补（D6/D11） | ✅ **反射双向分类测试**（`handler_write_bound_test.go`） |
| 同一生命周期两套实现 | ❌ `pcmBuffer` 不变量无测试 | ✅ **已修** —— `materials` 现在也有租约（BE-S1-1，2026-09-26 复核） |
| 死声明（声明了没人用） | ❌ `handshake` 死面 | ❌ `ai.audio.chunk` 无生产者（BE-S1-5） |
| 二进制帧缺 `turn_id` | ❌ D7 的根因 | ❌ 同一件事的另一侧（BE-S0-1） |
| 跨层/端到端测试位置空着 | ❌ `Integration/` 51 行 | ⚠️ `test/` 空，但 `scripts/smoke-*.sh` 有 12 个 |

**读法**：backend 在**工程纪律的机械化**上领先（门禁脚本、depguard、反射分类测试、租约队列）；iOS 在**语音链路的所有权收敛**上更完整（`TTSPlaybackCoordinator` 是单一所有者，backend 的两条音频路径是有意分开的）。

**两侧共同缺的**：一个能覆盖「真机/真进程」层级的验证位置，以及「把判断变成判据」的习惯在**所有**关键不变量上的落实。

---

### 8. v1 控制帧契约已退役（2026-09-25）

`fluentwork-infra/schemas/transport/wss-control-frames-v1.json` 已删除，两侧镜像与全部
消费点同步移除。**决定依据**：v2 是 v1 的严格超集（帧集合 17 ⊇ 14，字段只增不减），
且 v1 **没有任何生产消费者** —— backend 侧只有 `schemas/embed.go` 的 embed 声明和
`internal/voiceproto/frames_test.go` 在读它，iOS 侧只有 `PackageBaselineTests.swift`。

**随 v1 一起处置的 backend 测试**（它们全部只用于钉住 v1 的字节）：

| 测试 | 处置 |
|---|---|
| `TestSchemaV1BytesAreFrozen` | 删除（sha256 摘要钉随文件消失） |
| `TestSchemaV1OmitsClientTurnAbort` | 删除 |
| `TestSchemaV1AITextDeltaOmitsServerTsMs` | 删除 |
| `TestSchemaV1AITurnEndStaysFrozenWithoutOutcome` | 删除 |
| `TestSchemaV1DeadFacesAreStillPresentAndStillDead` | **拆开**：读 v1 的那半删除；断言 `Interrupt` 不序列化 `max_seq` 的那半保留为 `TestInterruptDoesNotSerialiseMaxSeq`（已用「去掉 `omitempty`」验证它会红） |
| `TestSchemaFilePresent` | 改指 v2，断言原样保留 |

iOS 侧 `wssControlFramesSchemaHasUserSpeechEndTurnAndText` 与
`wssControlFramesSchemaHasFeedbackBadgePhraseBlockAndTier` 原本读 v1，但它们断言的是
`user.speech.end` 与 `feedback.badge` —— 这两个 `$defs` 在 v2 里**逐字相同**，改指 v2
后断言一字未丢。

**P1-18 的教训（原记录写在被删的那个测试头上，移到这里）**：v1 冻结了两个从未运行过的
面 —— `ai.audio.chunk`（无生产者、无消费者、两侧无测试）与 `interrupt.max_seq`（声明
`omitempty` 且从未被赋值，客户端根本不解析）。它们不是「被遗忘」，而是**在有任何东西跑过
它们之前就被冻结**，于是把未知变成了永久负债：v1 是一句承诺，而关于没人执行过的代码的
承诺，是对「无」的承诺。**只冻结已经执行过的形态。**

⚠️ **退役 v1 并没有清掉这两个死面。** `aiAudioChunk` 与 `interrupt.max_seq` 在 **v2 里
逐字存在**，而 v2 没有 sha256 冻结、是可改的。**BE-S1-5 因此仍然成立。** 要真清掉，
得单独动 v2 —— 那是另一件事，不在本次范围内。

**仍未纠正的历史引用**（有日期的时点快照，记录的是当时为真的事实，**刻意不改**；
搜到 v1 时按本节理解）：`51_`、`61_`、`64_`、`76_`、`77_`。

**同一个坑对新冻结产物的提醒**：`schemas/transport/wss-binary-audio-frames-v1.json`
同样用 sha256 冻结，而其中 h8 布局**在写下这段时没有任何一端实现过**。它用 `status` 字段显式标注了
「target (Stage 3) — not implemented by either side yet」；实现落地后必须重新核对并刷新
摘要，不能把当时的猜测当成已验证的契约。

⚠️ **这段提醒在 2026-09-26 已经到期**（复核）：Stage 3 的代码**四个仓都已落地** ——
infra `f1a06e8`（v2 加可选 `turn_ref`，即布局开关）+ `1a54ee4`（刷新冻结产物的实现状态与摘要）、
backend `662c732`（下行 TTS 音频分配并置 `turn_ref`）、iOS `ab5f44e`（客户端容忍 h4 / h8
两种二进制帧头）。backend 侧已有 `voiceproto.AudioFrameLayoutH4` / `AudioFrameLayoutH8` 与
`AudioFrameLayoutFor`，并有 `TestAudioFrameLayoutForIsTheOnlyLayoutDecision` 钉住「它是布局
的**唯一**决策点」（H4 头 4 字节 / H8 头 8 字节 / 有 `turn_ref` 才选 H8）。
⇒ **只剩两件，且都不是本文件能单独关掉的**：① `status` 与冻结摘要要按**实现后的事实**重新核对
（不能留着 "not implemented by either side yet"）；② **h8 链路一次端到端真机跑都没有**
（= [`PRD核心业务逻辑落地工单.md`](./PRD核心业务逻辑落地工单.md) 的 **T6**）。
**BE-S0-1 因此应当从「不能单独做」升级为「四仓已落地，待真机验证」** —— 见 §1 与总清单 §1.1。

