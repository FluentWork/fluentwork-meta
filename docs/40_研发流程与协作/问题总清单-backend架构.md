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

### 2. S1 — 结构上该补的「防线」

| # | 条目 | 动作 | 风险 | 价值 | 证据 | 2026-09-26 复核 |
|---|---|---|---|---|---|---|
| BE-S1-1 | **`materials` 的 refine 没有租约**，`processing` 是没有出口的状态 | 照 `session_jobs` 抄：加 `attempts`/`locked_at` + 清扫，或挪进 `cmd/worker` | 中 | **高** | `03` §2.2 | ✅ **已完成** `25a015e` |
| BE-S1-2 | **`httpjson.Error` 零测试**，而它是每个 API 错误的唯一出口，有 4 个分支 | 补 4 个分支的测试 | 低 | **高** | `04` §3.2 | ✅ **已完成** `01a5e55` |
| BE-S1-3 | **手写二进制解码器没有 fuzz**（全仓 0 个 fuzz） | 给 `DecodeAITTSAudio` 写 fuzz | 低 | **高** | `04` §3.1 | ✅ **已完成** `cb99ba3` |
| BE-S1-4 | **控制帧分派是线性扫描，同一帧被解码最多 8 次** | 改成按类型查表，把已解出的 `frameType` 传下去 | **低**（只搬分派） | 中 | `02` §3.1 | ❌ **仍开着**（且是 9 次不是 8） |
| BE-S1-5 | **`ai.audio.chunk` 常量无生产者、无注释** | 加一句「无生产者，保留为协议位」；**v1 已退役但这张死面仍在 v2**（见 §8） | 无 | 低 | `02` §3.2 | ❌ **仍开着** |
| BE-S1-6 | **音频热路径的逐帧 `Debug` 日志**，成本无法量化（因为 0 个 benchmark） | 先加 benchmark，再用数据决定 | 低 | 中 | `02` §3.4、`04` §3.1 | ✅ **已完成** `46bd534` |
| BE-S1-7 | **`providerErrorStrategies` 的 key 与 handler 的对应关系只靠读** | 补一条测试：表里每个 key 都必须是真会转发给 provider 的帧类型 | 低 | 中 | `02` §2.1 | ✅ **已完成** `8c6792c`（发现比旧文更严重） |
| BE-S1-8 | **`config.Config` 夹具复制到 23 个文件（约 56 处）** | 抽 `config.TestConfig(t)` | 低 | 中 | `04` §3.4 | ❌ **仍开着** |
| BE-S1-9 | **`golangci-lint` 未安装** → `depguard` 那条纪律在本机从未执行过 | 装进 CI 镜像 + 写进 `SETUP.md` 前置条件 | 低 | 中 | `04` §3.5 | ✅ **前提不成立** |

**逐条证据（2026-09-26）**

- **BE-S1-1 ✅** —— 落地物齐全：`materials/model.go` 的 `DefaultRefineLease` / `MaxRefineAttempts = 2` / `ReclaimInterval` / `ErrorLeaseExpired`；`Store.ReclaimExpired`（`memory_store.go:108` + `mysql_store.go:109` 两个实现）；`Service.ReclaimExpired` + `Service.SweepIfDue`（`refiner.go:111` / `:135`，带 `lastSweep` 节流）。判据是 `internal/materials/refine_lease_test.go` **7 条**（活租约不动 / 过期重排 / 次数用尽转 failed / 跳过已删素材 / service 级重跑 / `SweepIfDue` 一个间隔只扫一次 / 重排记 transition）。
- **BE-S1-2 ✅** —— `internal/httpjson/httpjson_test.go` 存在（与 `httpjson.go` 同级）。
- **BE-S1-3 ✅** —— `internal/voiceproto/frames_fuzz_test.go:11` 的 `FuzzDecodeAITTSAudio`。**全仓已不止一个 fuzz**（旧文「全仓 0 个」作废）。
- **BE-S1-4 ❌ 仍开着，且数字变了** —— `handler_control.go:124` 仍是 `handlers := []func(context.Context, *websocket.Conn, ConsumedTicket, []byte, *sessionRuntime) (controlOutcome, error)` 的顺序扫描；`voiceproto.DecodeType(data)` 在该文件出现 **9 次**、`controlNotMine` **12 次**（旧文说「最多 8 次」）。每个 handler 各自从头解一遍类型。
- **BE-S1-5 ❌ 仍开着** —— `TypeAIAudioChunk` 全仓只有 `voiceproto/frames.go:22` 的声明与 `frames_test.go:764` 的引用，**没有生产者**。§8 已论证 v1 退役没有清掉它。
- **BE-S1-6 ✅** —— `internal/voicegateway/handler_audio_bench_test.go` **3 条** benchmark（`BenchmarkHandleAudioFrame` / `…DebugEmitted` / `BenchmarkAudioFrameDebugCall`）⇒「0 个 benchmark」作废，热路径 Debug 的成本**第一次可以量化**。
- **BE-S1-7 ✅ 已完成（`8c6792c`），而且真实情况比旧文写的严重** —— 旧文写「对应关系只靠读」（言下之意是读出来的关系是对的）。实际读下去发现**表里 `user.speech.end` 那一行从来没有被读过**：`startCollectTurn`（`handler.go:686`）自己直接调 `HandleClientControl`，失败时**手写死** `provider_control_failed` ⇒ 改那一行行为不变，它是**装饰**。修法因此不是「补一条测试」，而是**把两处收成一个实现点** `providerErrorCode(frameType)`，两个调用点都走它，并补 `handler_control_policy_test.go`（两条测试 + 一个记录「provider 收到了哪些帧」的 spy）。
  ⚠️ **我自己第一遍复核把它误判成「仍开着」。** 原因：我搜的是 `ProviderErrorPolicy` / `providerErrorStrategies` 这两个标识符，而新测试里**不出现**它们 —— 它靠 spy 记录收到的帧类型来断言。**「搜不到标识符」≠「没有这个行为」**，是 `git log` 里 `8c6792c` 的提交信息救回来的。
  ⇒ **教训写在这里：这份文件按「标识符」复核是不可靠的，要按「行为」复核**（或者直接读 `git log`，本仓的提交信息写得很细）。
- **BE-S1-8 ❌ 仍开着** —— 只有一个**包内私有**的 `testConfig()`（`internal/account/service_test.go:28`），没有跨包可用的 `config.TestConfig(t)`。旧文的「23 个文件 / 约 56 处」未复核，但「没有共享夹具」这条成立。
- **BE-S1-9 ✅ 前提不成立** —— `scripts/dev-check.sh` 的**第 3 步就是 `golangci-lint run ./...`**，本机跑出 **0 issues**（二进制在 `$(go env GOPATH)/bin`，由门禁自己 export）。⇒「未安装 ⇒ depguard 从未执行」已经不为真。**这一条要留意的不是安装，而是 CI 镜像里装没装** —— 那部分本次没复核。


**这两条「最该先做」的都已经做完了**（BE-S1-1 与 BE-S1-3，见上表 ✅）。旧文在这里写「BE-S1-1 是这张表里最该先做的一条」「BE-S1-3 是性价比最高的一条」，两条今天都只剩历史价值。值得记一笔的是：**它们被做掉的方式与旧文建议的完全一致**（照 `session_jobs` 抄租约；给手写解码器补 fuzz）—— 所以不是当时的判断错了，是**清单没跟上代码**。

⇒ 复核后剩下的 S1 只有 **BE-S1-4**（分派线性扫描，是一次重构）、**BE-S1-5**（要动 v2 契约 ⇒ 得先拍）、**BE-S1-8**（测试卫生）三条 —— **恰好是这张表里最贵、最需要决定的三个**。所以重排后 S2 的两条排到了前面，见 §5。

---

### 3. S2 — 卫生

| # | 条目 | 证据 | 2026-09-26 复核 |
|---|---|---|---|
| BE-S2-1 | `AGENTS.md:22` 把 `internal/corpus/` 写成 `internal/corpuss/`（照图找目录找不到） | `01` §4.3 | ✅ **已修**（全仓无 `corpuss`） |
| BE-S2-2 | `handler.go:776-778` 有一段**描述不存在的函数** `resolvedUserText` 的孤儿注释 | `02` §3.5 | ❌ **仍开着，且比旧文更糟** |
| BE-S2-3 | `uplink_constants.go` 的注释说「两份 uplink 路径都用这个常量」，实际 `voiceduplex` 有自己的 `uplinkChunkBytes` | `01` §2.5 | ❌ **仍开着** |
| BE-S2-4 | `httpserver` 的 `discovery` 手写了一份**部分**端点清单（列了 tts/hits/history/privacy/materials/topic_cards，没列 drill/corpus/content） | `01` §2.3 的边表 | ✅ **已完成**（backend `a003b16`） |
| BE-S2-5 | `/metrics` 是 7 个包手写文本的拼接，无 registry、无重名检测 | `01` §1 | ✅ **已完成**（backend `a003b16`） |
| BE-S2-6 | **指标发射器里只有一部分对 label 集合排序**，其余用 map range 直接渲染 → 每次 scrape 的**行序不同** | 见下方 | ✅ **已完成**（backend `8325795`；**判据本身在 `82bd06e` 修正**，见下） |
| BE-S2-9 | `tts` 的 `TestRouter_Stream_RouteHit` 断言全局计数器的**绝对值**（`expected 1 hit for voice-a, got 2`） | 本次复核新发现 | ❌ **仍开着** —— `-count>1` 必红；门禁固定 `-count=1`，所以从未暴露 |
| BE-S2-7 | `test/` 是空目录（只有 `.gitkeep`） | `04` §3.3 | ❌ **仍开着** |
| BE-S2-8 | 51 个环境变量没有一份清单（唯一来源是两个 `Config` 结构体） | `03` §4.1 | ❌ **仍开着，且比旧文更可量化** |

**逐条证据（2026-09-26）**

- **BE-S2-1 ✅** —— 在 `AGENTS.md` 里搜 `corpuss` 零命中；第 96 行现在写的是 `corpus-seed`。
- **BE-S2-2 ❌ 仍开着，且比旧文更糟** —— 不是「一段孤立注释」，而是**并进了别的函数的文档注释**：`handler.go:862-868` 是 `extractServerASRText` 的 doc，其中 3 行（「resolvedUserText returns the user's utterance…」）描述的是**另一个函数**，中间还空了一行；而全仓 **没有 `resolvedUserText` 的定义**（只有这一处注释命中，`grep "func.*resolvedUserText"` 零命中）。
- **BE-S2-3 ❌ 仍开着** —— `voicegateway/uplink_constants.go:12` 是 `const UplinkChunkBytes = 640`，`voiceduplex/volc_duplex.go:956-957` 另有 `uplinkChunkBytes`（注释「20ms of 16 kHz mono s16le (640 bytes)」），数值一致但**改一处不会让另一处红**。
- **BE-S2-4 ✅ 已完成**（backend `a003b16`）—— `discovery` 不再手写：`publicEndpoints` 读 `engine.Routes()`，按 `apiPrefix` 后的首段分组、组内排序。**并且顺带关掉一个披露面**：旧表把 `/internal/v1/tts/synthesize` 与 `/internal/v1/voicegateway/hits` 写在了一个**不需要任何令牌**的端点上，现在内部面按前缀整体扣掉。
  ⚠️ **这一步我最初判定「零消费方」是错的**：我用 `grep -v _test.go` 检索，正好把消费方筛掉了 —— 它是一条测试（`internal/session/http_test.go` 的 `TestOpenAPIDiscoveryEndpoints`，断言 `discovery["openapi"] == "/openapi.yaml"`），已同步到新形状。**与 BE-S1-7 同型：「搜不到标识符」≠「没有这个行为」。**
- **BE-S2-5 ✅ 已完成**（backend `a003b16`）—— `/metrics` 走 `metricsRegistry`：一处登记；`serveMetrics` 只调 `renderMetrics()`。三条守卫：源码树扫描比对注册表（带反空洞下限）、同一族指标不得被两个包渲染、登记了必须真被送出。
- **BE-S2-6 ✅ 已完成**（backend `8325795`）—— 5 处补排序，见下表。
  ⚠️ **但上一轮的判据是错的，已在 `82bd06e` 修正**：它把「行序稳定」和「**数值**稳定」混在一起（渲染 17 次要求逐字节相同），而同包其它测试真的在跑 refine 管线 ⇒ 整包跑时红、单跑时绿，**上一轮它绿是运气**。现在只断言标签集合**按升序出现**，数值丢弃；5 个包共用 `internal/metricstest.LabelSetsIn`。
- **BE-S2-9 ❌ 仍开着**（新发现）—— `internal/content/tts/router_test.go:53` 断言 `routeHits` 的绝对值，而它是包级全局：`-count=3` 时输出 `expected 1 hit for voice-a, got 3`。本次未修（与本次改动正交）。
- **BE-S2-7 ❌ 仍开着** —— `ls test/` 只有 `.gitkeep`（1 字节）。
- **BE-S2-8 ❌ 仍开着，且比旧文更可量化** —— 把两侧对了一遍：`configs/*.env.example` 声明 **27** 个变量，代码里 **读** 了 **42** 个 ⇒ **21 个只被读、没被声明**（`APP_BASE_URL` / `APP_RUN_REVIEW_WORKER` / `ARK_PRICING_FILE` / `ARK_THINKING` / `DRILL_DAILY_NEW_BLOCK_LIMIT` / `DRILL_PROMOTE_STREAK` / `DRILL_ROUND_SIZE` / `MINI_SESSION_TURN_LIMIT` / `MYSQL_DSN` / `TOPIC_MIN_BLOCKS` / `VOICE_CLIENT_ASR_REQUIRED` / `VOICE_DEV_ECHO_FIXTURE` / `VOICE_DEV_ECHO_TEXT` / `VOLC_DUPLEX_MODEL` / `VOLC_DUPLEX_VOICE` / `VOLC_POC_*`(4) / `VOLC_SPEECH_RESOURCE_TTS` / `VOLC_T9_TRIALS` / `WORKER_ID`）。
  ⚠️ **这不是纯文档问题**：`dev-up.sh:123-124` 在没有真实 env 文件时**会把 `configs/app-server.env.example` 当环境文件加载** ⇒ 示例漏一个键 = 那个旋钮在开发环境里不存在。**本次已顺手补上 `MINI_SESSION_TURN_LIMIT`**（backend `4cd446a`，T4 的收尾），其余 20 个未动。
  （现成形状：`TestVolcEnvExampleCarriesTheGatewayWiring` 断言的就是**模板**而非文件 —— 注释写着 "asserts the *template the file is rebuilt from*, which is the half that was wrong"。所以判据形状已经有了，只是没铺开。）


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

### 5. 如果只做三件事（2026-09-26 三次重排）

⚠️ **前两版的三条也都做完了**：旧版 BE-S1-1 ✅ `25a015e`、BE-S1-2 + BE-S1-3 ✅ `01a5e55` / `cb99ba3`；
二次重排的第 1 顺位 BE-S2-4 + BE-S2-5 ✅ `a003b16`。以下是同一判据下的当前顺位，且只从**复核后确认 ❌** 的条目里挑。

1. **BE-S2-8（21 个环境变量只被读、没被声明）** —— 剩下唯一一条**后果落在开发环境里**、且**判据今天就会红**的：`dev-up.sh:123-124` 在没有真实 env 文件时会把 `configs/*.env.example` 当环境文件加载 ⇒ 示例漏一个键 = 那个旋钮在开发环境里不存在。判据形状已经有了（`TestVolcEnvExampleCarriesTheGatewayWiring` 断言模板），只是没铺开；21 个的名单也已经在 §3 逐条列出。
2. **BE-S1-4（控制帧分派是线性扫描 + 同一帧被重复解码）** —— 剩下三条 S1 里**唯一既不需要决定、也不需要动契约**的一条（另两条：`BE-S1-5` 要改 v2 契约，`BE-S1-8` 只是测试夹具收敛）。每加一类帧，就要在 9 个 handler 里各插一次 `DecodeType` + `controlNotMine`。
3. **BE-S2-2 + BE-S2-3（两处注释在描述别的东西）** —— 两条同源，且 `BE-S2-2` **比旧文写的更糟**：不是一段孤立注释，而是并进了 `extractServerASRText` 的文档注释里（3 行描述一个全仓不存在的函数）。它们不是功能缺陷，但它们**主动误导读者** —— 而本清单一天里被同一种病咬了三次。

**为什么这次把 `BE-S2-9` 排除在三位之外**：它的后果（`-count>1` 会红）**今天没有任何东西会走到**，
因为门禁固定 `-count=1`。它是一条真缺陷，但排在「修复后无人受益」的位置上 —— 先修会误导读者的那两条。

**为什么 S2 仍然排在 S1 前面**：旧文的分级（S1 = 不做会在下一次同类缺陷上再花一遍时间；S2 = 只是让下一个人多读一会儿）本身没错，但**复核后剩下的 S1 恰好是最贵、最需要前提的**。分级答的是「不做会怎样」，不是「先做哪一个」。

**BE-S0-1 已从这张表里出局**（2026-09-26 复核）：四仓代码都落地了，剩下真机跑与冻结产物摘要重核，见 §8。它当初被排除在「三件事」之外的理由是**它不能单独做**（需要协议版本、两侧同时改、先确认没有老客户端）—— 而这个理由后来被**一次专门排期**解决了。**它不是被塞进「三件事」里做掉的，是单独做掉的**，这个区分值得留着：S0 的条目不该为了「凑进三件事」而开工。

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
| 路由表要能断言 | ✅ 字典 + 生产工厂驱动 | ⚠️ 失败策略表可断言，**分派是线性扫描**（BE-S1-4） |
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

