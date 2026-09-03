# FluentWork 第一波 + 第二波 App 可测试问题验收清单

**版本**：V1.0
**日期**：2026-09-02
**定位**：把第一波 / 第二波中可在 iOS App 中实际点击验收的 issue 收口成一份可执行清单，供用户本地测试与验收；本文档**只列已 CLOSED 的可验项**，把仍 OPEN 的项作为已知限制单独标出
**上游依据**：
- `fluentwork-meta/docs/40_研发流程与协作/44_...第一波跨仓任务结构...md`
- `fluentwork-meta/docs/40_研发流程与协作/51_...第二波跨仓任务结构...md`
- `fluentwork-meta/docs/50_测试与验收/51_...第一波能力验证与第二波薄弱点检查门禁.md`
- `fluentwork-meta/docs/50_测试与验收/52_...第一波遗留问题清账与第二波启动前计划.md`
- `fluentwork-meta/docs/60_评审与复盘/62_FluentWork第一波关闭记录.md`
- 仓内各 runbook（`docs/06_`、`docs/09_`、`docs/11_` 等）

**状态**：执行文档。本文是测试者侧的入口，不替代 `51/52` 号门禁文档。

> 2026-09-03 真机主链路验收记录见 `55_FluentWork主链路验收记录_2026-09-03.md`；量化 / 弱网 / 锁屏项仍延后。

---

## 〇、读这份文档前请先知道的四件事

### 1. 当前哪些 issue 已经可以测

| 仓 | issue 范围 | 当前状态 |
|---|---|---|
| backend | B1–B7 | 全 CLOSED |
| backend | B9–B14 | 全 CLOSED |
| ios | I1–I13 | 全 CLOSED |
| meta | Epic 1–3 | 全 CLOSED |

也就是说：

> **第一波 B1–B7 / I1–I6 与第二波 B9–B14 / I7–I13 均已在 GitHub 上 CLOSED，可作为本轮可测试基线。**

### 2. 当前哪些 issue 仍 OPEN（不可在 app 中完整验收，作为已知限制列在最后一节）

| issue | 标题 | 现状对 app 的影响 |
|---|---|---|
| `meta #7` | EPIC: 回顾与炼化闭环 | 仅元仓层定义，不影响 app |
| `meta #8` | EPIC: 语料库基础版 | 同上 |
| `meta #9` | EPIC: 每日一读 | 同上 |
| `meta #10` | EPIC: 即时反馈命中检测 | 同上 |
| `meta #11` | EPIC: 语音链路火山做实 | 同上 |
| `meta #12` | PREREQ: 火山供应商三项商务前置闭环 | 仅商务前置 |
| `backend #21` | **B8: implement review evaluation and refine in ai-worker** | **核心 OPEN 项**：回顾页内的"转录 / 三层评价 / 双栏对照 / 炼化卡"内容由 B8 真实 LLM 产出，B8 未闭合前，回顾页只有骨架 / 占位数据 |
| `backend #28` | B15: build offline evaluation dataset and prompt regression baseline | 仅回归门禁脚本，不直接影响 app |

### 3. 验收时的运行环境

本文列出的所有 case 默认假设：

1. **iOS Host**：以 `iPhone 17 Pro` 模拟器或本机真机为准
2. **后端**：`fluentwork-backend` 本地 dev 栈就绪，对应 baseURL 默认 `http://127.0.0.1:8080/api/v1`，WSS 默认 `ws://127.0.0.1:8081/v1/voice`
3. **iOS App 启动环境变量**：默认走 `AppEnvironment.local`，`Info.plist` 中需含 `UIBackgroundModes = audio`（每日一读后台播放用）
4. **麦克风权限**：首次进入"说的房间"会触发系统弹窗；调试器可通过重置模拟器拒绝/允许权限
5. **离线兜底**：daily read / corpus 在弱网或后端 5xx 时会有 fallback 占位文案，不要把它误判为 bug

### 4. 验收走法的统一约定

1. 每条 case 给出 `[步骤]` / `[预期]` / `[是否通过 ❑ 是 / ❑ 否]` / `[问题备注]`
2. 验收时在 App 内由用户手点操作；脚手架命令（runbook / smoke 脚本）只在需要后端就位的前提下执行
3. 任一 case 期望落空，先记录**具体复现步骤、截图与时间戳**，再决定 bug 归属；不要立刻改实现

---

## 一、第一波可在 App 中验证的 issue

### 1.1 后端 B1–B7（7 项）

> 后端 issue 单点验证主要通过 backend smoke 脚本完成，但都体现在 app 的"用户旅程"上，所以下面按 app 视角列举。

| 编号 | issue 标题 | 在 app 中具体验证什么 | 必跑脚本 / 命令 | 通过证据 |
|---|---|---|---|---|
| B1 | add guest auth and merge endpoints | 首次打开 app 即自动签发游客身份；后续在设置或账户处触发 `merge` 时不产生重复账户 | `./scripts/dev-up.sh`（含 `/auth/guest` smoke） | `local-services-start.sh` 启动后 `guest token` 可正常签发并写入 storage |
| B2 | add session creation endpoint | 进入"说的房间"自动触发 `POST /sessions` 创建会话；返回票据有效期可观察 | `./scripts/smoke-review-ready.sh` 输出中的 `session_id` 段 | `session_id` 不为空；`ticket` 字段存在 |
| B3 | scaffold voice-gateway WSS protocol | app 内部 SocketTransport 能与 `voice-gateway` 完成握手并收发至少一条控制帧 | `swift test --filter SpeakingRoomTransportBridge` | 69+ swift 测试全绿（详见第三部分 §3.1） |
| B4 | persist session end data | 在说的房间点结束会话，能让 backend 完成 `session.end` 落库与 `session.finished` 投递 | `./scripts/smoke-review-ready.sh` 中 `session.end` 段 | evidence JSON 中含 `utterance_count` 与 session 状态 |
| B5 | scaffold review worker pipeline | session 结束后 worker 异步消费 `session.finished`，落 stub review | `./scripts/smoke-review-ready.sh` 中轮询 review | `review_status=ready` 或 `generator=stub-v1` |
| B6 | add review polling endpoint | app 端轮询 `GET /sessions/:id/review`，能拿到三态之一 | 同上 | review polling 段 status 可为 `pending` 或 `ready` |
| B7 | add degraded text messages endpoint | `degradedText` 状态下 app 调用 `POST /sessions/:id/messages`，语音可用时返回 409 | `swift test --filter SessionAPIClient` / smoke 脚本 | 单测覆盖；UI 上 `degradedText` 阶段文案"网络不稳定，当前会话需要恢复后再继续"可见 |

### 1.2 iOS I1–I6（6 项）

| 编号 | issue 标题 | 在 app 中具体验证什么 | 必跑脚本 / 命令 | 通过证据 |
|---|---|---|---|---|
| I1 | add root store and DI baseline | 启动后 AppState、Root Store、Factory 容器全量运转；不依赖 Combine | `swift test`（基础包测试） | 216 项 swift 测试全绿 |
| I2 | wire SpeechSession reducer boundary | 触发说话键 → `session.start`；状态机按 `idle → connecting → recording → ... → ended` 推进；不在 UI 层直接读后端状态 | `swift test --filter SpeechSessionMachine` | 状态机测试全绿 |
| I3 | add socket transport and frame decoding | 断网 / 重连可在 UI 显示，丢帧规则被单测覆盖 | `swift test --filter SpeakingRoomTransportBridge` | transport 测试全绿 |
| I4 | add session API client | 上述 B1/B2/B6/B7 接口归一化；错误模型一致 `{code,message,request_id}` | `swift test --filter SessionAPIClient` | 测试全绿 |
| I5 | scaffold speaking room screen | "说的房间"页面骨架稳定，能承载状态机各 phase；loading / failed / retry 可见 | `./Scripts/smoke-iphone17pro.sh` | runbook 输出 + launch 测试 3 项绿 |
| I6 | add review polling and skeleton state | "回顾"页面骨架态：未就绪可见 loading；可手动重试 | 同上 | 回溯页可从工作台模块"回顾"进入；骨架态文案"正在等待转录、评价与炼化内容"显示 |

### 1.3 第一波整体主链路端到端（合 1 张图）

```text
游客登录 (B1)
  -> 创建 session (B2)
  -> 网关握手 (B3)
  -> 说的房间 UI 壳 (I5) + SpeechSession (I2)
  -> SocketTransport (I3) + APIClient (I4)
  -> 用户开口说一句，按结束 (I4 session.end)
  -> 落库 + 投递 session.finished (B4 + B5)
  -> 轮询 review 骨架 (I6 + B6 + worker B5)
  -> 失败态可重试，文本降级末级兜底 (B7)
```

**在 app 中一次验证即可同时覆盖 B1–B7 + I1–I6**：打开 app → 说一句话 → 等待回顾出现骨架态 → 等待出现 ready / failed / fallback 任一确定态。

---

## 二、第二波可在 App 中验证的 issue

> 第二波把所有"占位"做实。本节按 PRD W5–W6 的能力面分组，从 app 用户视角倒推验证要点。

### 2.1 回顾与炼化闭环（对应 I7 / I8 / B9 / B10）

> **重要前提**：B8（AI 真实评价 + 炼化实现）当前仍 OPEN，所以这里看到的"评价 / 炼化"在 B8 落地前是骨架或 stub 占位；调用 `POST /corpus/blocks/batch-accept` 仍可走，因为该接口已由 B10 CLOSED 完成。

#### 2.1.1 I7：完整回顾页（review 页）

| 项 | 在 app 中看到的现象 | 通过条件 |
|---|---|---|
| 渐进到达时序 | 进入回顾页可见加载骨架；后端响应后逐层浮现 | `.loading` → `.pending` → `.ready`；中间态不卡死、不退化为空白 |
| 三态三态正确 | 未就绪 / 已就绪 / 失败可在同一会话里复现 | `.pending` 显示"正在等待..."；`.failed` 显示"暂不可用"+ 重试按钮；`.ready` 显示完整内容 |
| 双栏对照 | ready 时显示 "You / Better" 列表 | 列表项存在且非空 |
| 转录话轮 | ready 时显示说话者列表 | 列表项存在且非空 |
| **评价与炼化真实数据** | **依赖 B8；B8 OPEN 期间，回顾 ready 阶段只能看到 stub** | 验收时**显式记录"B8 未闭合"**，不视为 I7 缺陷 |

#### 2.1.2 I8：炼化卡交互与入库

| 项 | 在 app 中看到的现象 | 通过条件 |
|---|---|---|
| 单卡入库 | 炼化卡右侧"加入语料库"按钮可点击 | 点击后按钮变 disabled 并改文为"已加入"，且不重复入库 |
| 一键全入库 | ready 阶段顶部存在批量入库入口（如设计存在） | 点击后所有卡进入"已加入"态；网络仅一次 batch-accept |
| 空态 | 评价返回空 list 时显示明确空态 | 不闪退 |
| 入库结果反映到语料库 | 完事切到"语料库"页面，可见新加入的块 | 内容、来源元信息可见 |

#### 2.1.3 B9 / B10 后端契约（不可直接点验但属本次验收依赖）

| 编号 | issue 标题 | 在 app 中看到的现象 | 通过条件 |
|---|---|---|---|
| B9 | extend review endpoint to full model | I7 ready 阶段一次性拿到 overview / transcript / dual / refine | swagger / smoke 脚本 response 字段完整 |
| B10 | add corpus module and phrase_blocks migration | 语料库页能拉到服务端块的列表 + 入库可成功 | `./scripts/smoke-corpus.sh` 全绿 |

### 2.2 语料库基础版（对应 I9 / I13 / B10）

| 项 | 在 app 中看到的现象 | 通过条件 |
|---|---|---|
| 首次进入 | 进入语料库 Tab，可看到本地缓存 + 远端数据 | 不白屏 |
| 双维度筛选 | 顶部存在"场景 / 功能"筛选项 | 选中后列表即时过滤 |
| 关键词搜索 | 顶部搜索框输入关键词 | 命中块浮现；清空后恢复 |
| 状态灯 | 每行行末或中部显示"新入库 / 训练中 / 已自动化" | 颜色 / 文案与 PRD 一致 |
| 本地优先 | 弱网（飞行模式）切到语料库 | 仍可看到上次缓存；恢复网络后看到新增 |
| 最小收藏 | 每行右侧有 ☆ 切换 | 点击切换无明显延迟，重启 app 后仍保留 |
| **删除 / 置顶** | I9 范围未包含 | **验收时按"未做"记录，不视为缺陷** |

### 2.3 每日一读（对应 I10 / B11）

详见 `docs/09_I10_smoke_test_runbook.md`，验收时按文档中 P0 / P1 case 表跑一遍：

| P0 必跑 | app 中现象 | 通过条件 |
|---|---|---|
| Case A 今日文章展示 | 进入 daily-read 可见 skeleton → ready | 一分钟内出 ready 或 fallbackPreset |
| Case B AI 朗读播放 | ready 后点播放，1–3s 内开始播放 | 进度条推进 |
| Case C 暂停 / 恢复 | 暂停后从断点继续 | 暂停后未归零 |
| Case D 锁屏持续播放 | 锁屏后音频继续 + 锁屏控件可见 | `Info.plist` UIBackgroundModes 已含 audio |
| Case E 切后台继续播放 | 切后台音频不中断 | 同上 |
| Case F 来电中断 | 来电时立刻暂停 | 通话结束不抢播 |
| Case G 网络断开 fallback | 开飞行模式进入 | 切到 failed 或 fallbackPreset，可重试 |
| Case I 跟读提交 | 点开始/停止跟读 | 状态切到"正在提交..." → "今日已跟读"；按钮变 disabled |
| **Case K V1.1 硬约束不出分** | UI 任何处显示评分都视为红线 bug | 必须通过 |

### 2.4 即时反馈命中检测（对应 I11 / B12）

| 项 | 在 app 中看到的现象 | 通过条件 |
|---|---|---|
| 对话中命中 | 在说的房间看到顶部出现"地道表达 +1"轻量徽章 | 仅展示，**不弹窗、不震动、不打断语音流** |
| 徽章累计 | 多次命中应在同一位置轮换 / 折叠显示 | 不刷屏（命中合并或去重） |
| 命中未命中 | 完全无命中时不显示徽章 | UI 无残影 |
| 主链路不受影响 | badge 检测 800ms 超时/失败 | 音频流、转录、session.end 流程全部不受影响 |

### 2.5 真实语音链路（对应 I12 / B13 / B14）

| 项 | 在 app 中看到的现象 | 通过条件 |
|---|---|---|
| 真实音频采集 | app 真机/模拟器采集到语音 | 时域波形或转录可见 |
| AI 语音播放 + 打断 | 说话时 AI 立刻停播 | 目标 200ms 内声音消失 |
| 后台播放配置 | 锁屏 / 切后台后播放不被中断 | `Info.plist` UIBackgroundModes 含 audio |
| 三级降级 | 火山不可达 → 模块化 → 纯文本 fallback | 状态机不崩，至少能切到 `degradedText` |
| **供应商凭证** | 客户端 App 中**搜不到任何 API Key 字串** | 代码审查 + 进程环境变量截图 |

---

## 三、验收前的快速烟囱测试

### 3.1 iOS Host 烟囱（5 分钟）

```bash
cd "/Users/apple/Developments/FluentWork App/fluentwork-ios"
./Scripts/smoke-iphone17pro.sh
```

通过条件：

1. 退出码 `0`
2. 输出含 `iPhone 17 Pro` + `launch succeeded`
3. `launch/bootstrap` 至少 3 个测试全绿
4. 日志落在 `.tmp/smoke-iphone17pro/`

### 3.2 Backend 全链路烟囱（3 分钟）

```bash
cd "/Users/apple/Developments/FluentWork App/fluentwork-backend"
./scripts/smoke-review-ready.sh
```

通过条件：

1. stdout 含 `wave1 review-ready smoke PASS`
2. evidence JSON 中 `review_status: "ready"`
3. 含 `generator` 字段；当前若启用了方舟开关，可能显示 `ark-review-refine-v1`
4. 失败时优先查 `app-server` / worker / DSN

### 3.3 Backend 语料库烟囱（2 分钟）

```bash
cd "/Users/apple/Developments/FluentWork App/fluentwork-backend"
./scripts/smoke-corpus.sh
```

通过条件：

1. stdout 含 `corpus smoke PASS`
2. 含 `phrase_block_id` 与场景 / 功能字段

### 3.4 Backend 每日一读烟囱（2 分钟）

```bash
cd "/Users/apple/Developments/FluentWork App/fluentwork-backend"
./scripts/smoke-daily-read.sh
```

通过条件：

1. stdout 含 `daily-read smoke PASS`
2. 命中分支为 `ready` 或正确回退到 `fallbackPreset`

---

## 四、逐 issue 测试验收 checklist（可直接复制到 PR 评论 / Issue 评论）

### 4.1 第一波 checklist

```
B1 - 游客身份签发：                  [ ] 通过  [ ] 不通过
B1 - merge 幂等：                     [ ] 通过  [ ] 不通过
B2 - session 创建：                   [ ] 通过  [ ] 不通过
B3 - 网关握手 / WSS schema：          [ ] 通过  [ ] 不通过
B4 - session.end 落库：               [ ] 通过  [ ] 不通过
B5 - worker 异步闭环：                [ ] 通过  [ ] 不通过
B6 - review 三态轮询：                [ ] 通过  [ ] 不通过
B7 - 文本降级占位：                   [ ] 通过  [ ] 不通过
I1 - Root Store / DI：                [ ] 通过  [ ] 不通过
I2 - SpeechSession 边界：             [ ] 通过  [ ] 不通过
I3 - SocketTransport / 帧解码：       [ ] 通过  [ ] 不通过
I4 - SessionAPIClient：               [ ] 通过  [ ] 不通过
I5 - 说的房间 UI 壳：                 [ ] 通过  [ ] 不通过
I6 - review 轮询骨架态：              [ ] 通过  [ ] 不通过
```

### 4.2 第二波 checklist

```
B9  - review 接口全量返回：           [ ] 通过  [ ] 不通过
B10 - corpus 模块 + phrase_blocks：   [ ] 通过  [ ] 不通过
B11 - content 模块 + 每日一读：       [ ] 通过  [ ] 不通过
B12 - B7 命中检测链路：               [ ] 通过  [ ] 不通过
B13 - voice-gateway 火山端到端：      [ ] 通过  [ ] 不通过
B14 - 端到端注入 POC：                [ ] 通过  [ ] 不通过
I7  - 完整回顾页：                    [ ] 通过  [ ] 不通过（依赖 B8）
I8  - 炼化卡交互与入库：              [ ] 通过  [ ] 不通过
I9  - 语料库页本地优先：              [ ] 通过  [ ] 不通过
I10 - 每日一读页：                    [ ] 通过  [ ] 不通过
I11 - B7 徽章展示：                   [ ] 通过  [ ] 不通过
I12 - 真实 AudioEngine 替换：         [ ] 通过  [ ] 不通过
I13 - 语料库同步策略定案：            [ ] 通过  [ ] 不通过
```

### 4.3 自动化与活体证据复核

```
iOS swift test (216 项):              [ ] 全绿   [ ] 不全绿
iOS iPhone 17 Pro smoke:              [ ] 全绿   [ ] 不全绿
backend go test ./...:                [ ] 全绿   [ ] 不全绿
backend ./scripts/smoke-review-ready: [ ] PASS   [ ] FAIL
backend ./scripts/smoke-corpus:       [ ] PASS   [ ] FAIL
backend ./scripts/smoke-daily-read:   [ ] PASS   [ ] FAIL
```

---

## 五、已知限制（OPEN 项，会影响 app 验收口径）

下面这一节必须保留，是为了避免"看着像 bug 但其实是 OPEN 项"。验收时遇到下列场景，按"已知限制"处理，不要把它归类为上面二、三、四节的任何一条。

1. **B8（评价 + 炼化真实现）仍 OPEN**
   - 影响：I7 ready 阶段只能看到 stub / 评价占位；review JSON 中 `generator` 仍会是 stub 实现
   - 处理：验收 I7 时只验骨架 / 字段 / 时序，不要对内容质量做评分
2. **B15（离线评估集 + Prompt 回归基线）仍 OPEN**
   - 影响：纯回归脚本层缺失，不影响 app
3. **meta Epic 4–8 + PREREQ 仍 OPEN**
   - 影响：仅在 meta 层定义，不影响 app
4. **范围欠账（不属任何已 CLOSED issue）**
   - `C4` 创建练习弹层、`A1-A2` 素材模块、`I1-I4` 发音评测、`E1-E5` 闪测、`H1-H4` 话题建议、`I9` 语料库删除 / 置顶
   - 处理：验收时按"未做"记录

---

## 六、验收流程建议

为了一次验收把所有上面的项跑完，建议按下面顺序：

1. **准备阶段（10 分钟）**
   - 启动 backend dev 栈（`./scripts/dev-up.sh`）
   - 在 iOS 模拟器或真机 build + run Host
   - 把 iOS 环境变量 `LOCAL_HOST` 指向 backend（如真机）

2. **逐项验收（30–60 分钟，按节奏）**
   - 按第一波 → 第二波的顺序遍历 §1.3 端到端主链路
   - 在 §4 的 checklist 上勾选
   - 任何 ❑ 选"否"的项落到最后的"问题清单"

3. **收口（10 分钟）**
   - 把 §4 各 checklist 的结果汇总，附到 PR 评论或 Wiki
   - 若发现新 bug，回写对应 issue / 仓的 issue，并按"阻塞 / 非阻塞"分类
   - 把属于 §5"已知限制"的现象移除，不要混入问题清单

---

## 七、对接到门禁文档时的回写规则

- **本清单与 `51_FluentWork第一波能力验证与第二波薄弱点检查门禁.md` 的关系**：
  - `51` 是机器 / owner / 评审者侧门禁（自动化 + 结构审查 + 活体 runbook）
  - 本清单 (`53`) 是**用户手点 app 时的验收清单**，与 `51` 互补不重复
  - 二者通过 §3 的烟囱测试共享证据（iOS swift test / smoke runbook 输出落盘即可引用）
- **本清单与 `62_FluentWork第一波关闭记录.md` 的关系**：
  - `62` 已经记录了第一波已具备证据，本清单是 `62` 证据在 app 用户视角的展开

---

## 八、最终口径

> **第一波 + 第二波已 CLOSED 的实现 issue，可在 app 中按本文 §1.3、§2.1–2.5 的能力面、§4 的 checklist 逐项验收；唯一会显著影响验收口径的是仍 OPEN 的 B8（评价 + 炼化真实 LLM），其他 OPEN 项不阻塞 app 端验收。本轮跑完后再决定是否进入下一步开发。**
