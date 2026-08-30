# FluentWork 第一波能力验证与第二波薄弱点检查门禁

**版本**：V1.0  
**日期**：2026-08  
**定位**：把第一波当前能力验证、现行检查链路、第二波薄弱点的放行门禁收口为一份可执行清单，避免实现完成后没有独立检查环节  
**上游依据**：`agents/shared/review-gate.md`、`43_三仓协作与Review Workflow配置说明.md`、`30/31/32/33` 号技术方案文档、`51/52/61` 第二波文档、各代码仓第一波/第二波入口文档  
**状态**：执行文档

---

## 一、这份文档解决什么问题

当前三仓已经有：

1. 本地 `gstack /review` 审查门禁
2. CI 的 build / test / lint / config 检查
3. 高风险目录的人类 owner 审批

但还缺两件事：

1. **按波次、按薄弱点绑定的验收门禁**
2. **把“做完了”与“验证过了”分开的证据清单**

这会带来一个常见问题：

> 代码已经实现，但没有独立的“反证环节”去确认失败态、弱网、幂等、降级、时序是否真的被检查过。

本文的目标就是把这个缺口补上。

---

## 二、当前是如何进行“辩证检查”的

当前链路不是单一的“跑测试就算过”，而是四层证据叠加：

```text
实现代码
  -> 本地自检（build / test / lint / contract）
  -> gstack /review（看结构、边界、回归风险）
  -> 活体验证 / runbook（看真实链路能否跑通）
  -> 人类 owner 审批（做最终取舍）
```

具体口径：

1. **自动化证据层**  
   - iOS：`swift test`
   - backend：`go test ./...`、`./scripts/dev-check.sh`
2. **结构审查层**  
   - commit 前必须跑 `gstack /review`，pre-commit 以 `GSTACK_REVIEWED=1` 声明
3. **活体验证层**  
   - backend 已有 `./scripts/dev-up.sh` 本地 smoke 入口
   - iOS 当前只有 package test，尚未形成挂到 `iPhone 17 Pro` 模拟器的固定 smoke run
4. **owner 判断层**  
   - 高风险目录仍由人类 owner 做最后审批，不由机器替代

### 当前缺口

当前链路已经具备“审查 + CI + owner”的骨架，但还**不算完整的辩证检查**，原因有三：

1. 检查项更多是“仓库级”，还不是“波次能力级”
2. 第一波缺少一份统一的放行证据清单
3. 第二波薄弱点虽然被识别出来了，但还没全部转成阻断式门禁

---

## 三、第一波当前验证现状（2026-08-30 实测）

下面这部分以当前工作区实测为准。

### 3.1 iOS

已拿到的证据：

1. `swift test` 实测通过，当前通过 **69 项测试**
2. 覆盖面已经包含：
   - `SpeechSessionMachine` 状态迁移与异常分支
   - `SpeakingRoomTransportBridge` 的事件桥接
   - `SessionAPIClient` 的游客鉴权 / session 创建 / review 轮询 / 文本降级
   - `LaunchToNavigationEndToEndTests` 的 bootstrap → 导航 → feature projection
3. 本机存在 `iPhone 17 Pro` 模拟器

当前缺口：

1. CI 里只有 `swift-package-test`，**没有真正的 iPhone 17 Pro 模拟器 smoke gate**
2. 还没有把“第一波最小链路在模拟器可点击跑通”写成固定 runbook 或 required evidence

### 3.2 Backend

已拿到的证据：

1. `go test ./...` 实测通过
2. 本地 smoke 实测通过：
   - `POST /auth/guest`
   - `POST /sessions`
   - `GET /sessions/:id/review` 返回 `pending`
   - `POST /sessions/:id/messages` 文本降级返回 `stub-text-v1`
3. 单测 / 契约测试已覆盖：
   - guest 签发与 merge 幂等
   - session 创建、票据消费、过期 / 重放保护
   - gateway WSS 握手与最小会话循环
   - `session.end` → job → stub `review_json` → `review ready`

当前缺口：

1. 本地 live smoke 还没有覆盖“真实 `session.end` 后 worker 跑到 `review ready`”的固定 runbook
2. `./scripts/dev-up.sh` 目前只 smoke 到 guest auth，没有把第一波全链路做成一条命令

### 3.3 结论

第一波当前不是“没有验证”，而是：

1. **自动化基础已经存在**
2. **后端有最小活体 smoke**
3. **iOS 缺少模拟器活体验证门禁**
4. **跨仓还缺一份统一的放行清单**

所以第一波的真实状态应表述为：

> **具备单测、契约测试和部分活体验证，但还没有形成“波次能力放行 gate”。**

---

## 四、第一波放行门禁（从现在开始执行）

本节定义第一波能力的关闭门禁。未满足时，不应宣称“第一波已完整验证”。

### 4.1 iOS 第一波门禁

| 编号 | 门禁 | 阻断级别 | 所需证据 |
|---|---|---|---|
| W1-IOS-1 | `swift test` 全绿 | 阻断 | 测试日志或 CI 记录 |
| W1-IOS-2 | `SpeechSession` / transport / degraded text 相关测试覆盖保持存在 | 阻断 | 对应测试文件仍在，新增行为有测试更新 |
| W1-IOS-3 | `iPhone 17 Pro` 模拟器 smoke 跑通最小链路 | 阻断 | 模拟器运行记录：启动 app、bootstrap ready、可进入说的房间或消费 mock 首波能力 |
| W1-IOS-4 | 若改动高频路径（波形 / 流式文本 / interrupt），需补性能或时序说明 | 条件阻断 | PR 说明或专项记录 |

**现状**：

- `W1-IOS-1`、`W1-IOS-2`：已具备
- `W1-IOS-3`：**未形成固定门禁**
- `W1-IOS-4`：按高风险改动触发

### 4.2 Backend 第一波门禁

| 编号 | 门禁 | 阻断级别 | 所需证据 |
|---|---|---|---|
| W1-BE-1 | `./scripts/dev-check.sh` 或等价 `go test ./... + go build ./...` 全绿 | 阻断 | 终端日志或 CI 记录 |
| W1-BE-2 | 本地 smoke 跑通 `guest -> session -> review pending -> text degrade` | 阻断 | runbook 输出 |
| W1-BE-3 | `session.end -> worker -> review ready` 至少有一条可重复证据 | 阻断 | `service_test` 或本地 runbook |
| W1-BE-4 | gateway WSS 握手与最小回路契约测试通过 | 阻断 | `voicegateway` / `voiceproto` 测试记录 |

**现状**：

- `W1-BE-1`、`W1-BE-2`、`W1-BE-4`：已具备
- `W1-BE-3`：当前主要靠单测，不是 live runbook

### 4.3 跨仓第一波门禁

| 编号 | 门禁 | 阻断级别 | 所需证据 |
|---|---|---|---|
| W1-X-1 | commit 前完成 `gstack /review` | 阻断 | `GSTACK_REVIEWED=1` |
| W1-X-2 | PR required checks 全绿 | 阻断 | GitHub checks |
| W1-X-3 | 高风险目录保留 owner 审批 | 阻断 | PR approval |
| W1-X-4 | PR 描述写明验收证据，而不是只写改了什么 | 非阻断，逐步收紧 | PR 模板 / 描述 |

---

## 五、第二波薄弱点检查门禁

这一节把第二波的“薄弱点”转成阻断式检查项。没有证据，不应放行为“完成”。

### 5.1 回顾页失败语义与渐进到达门禁（`B8` / `B9` / `I7`）

| 门禁 | 要求 | 所需证据 |
|---|---|---|
| W2-R1 | `review pending / ready / failed` 三态都可解释，不出现空白页 | 后端响应样例 + iOS 状态/UI 测试 |
| W2-R2 | 渐进到达时序明确：骨架 -> 转录/评价 -> 炼化卡 | iOS 两层到达时序单测 |
| W2-R3 | Schema 校验失败、重试耗尽后，转录仍可回看 | backend 测试 + PR 说明 |

### 5.2 语料库本地优先门禁（`I13` / `I9` / `B10`）

| 门禁 | 要求 | 所需证据 |
|---|---|---|
| W2-C1 | `I13` 定案先于 `I9` 动码 | 定案文档回写记录 |
| W2-C2 | 覆盖游客归并、删除语义、收藏同步、游标推进 | 定案文档 + 测试用例映射 |
| W2-C3 | `batch-accept` 幂等、软删除、最小收藏口径固定 | backend 测试 |
| W2-C4 | 弱网本地优先场景有测试，不只测 happy path | iOS 场景测试 |

### 5.3 每日一读生成与兜底门禁（`B11` / `I10`）

| 门禁 | 要求 | 所需证据 |
|---|---|---|
| W2-D1 | `uk_user_date` 或等价唯一键防重复生成 | migration / 测试 |
| W2-D2 | 生成未就绪时有明确轮询态 | iOS 页面状态测试 |
| W2-D3 | 生成失败时有兜底内容或最近成功内容，不出现空页 | backend / iOS 联合证据 |
| W2-D4 | 跟读不出分，评分字段不消费 | 接口与 UI 证据 |

### 5.4 徽章反馈不扰动主链路门禁（`B12` / `I11`）

| 门禁 | 要求 | 所需证据 |
|---|---|---|
| W2-B1 | 命中检测 800ms 超时 / 失败直接跳过，不污染主链路 | backend 集成测试 |
| W2-B2 | 同一话轮重复命中去重或合并展示 | 规则说明 + iOS 测试 |
| W2-B3 | 徽章展示不进入 `SpeechSession` reducer | iOS 测试 / 代码审查证据 |
| W2-B4 | 徽章展示不弹窗、不震动、不阻塞语音流 | UI / 交互记录 |

### 5.5 真实语音链路门禁（`B14` / `B13` / `I12`）

| 门禁 | 要求 | 所需证据 |
|---|---|---|
| W2-V1 | `B14` 先出注入档位结论，再允许 `B12` 开工 | POC 报告 |
| W2-V2 | `B13` 接入前供应商三项前置已闭环 | 执行记录 |
| W2-V3 | 首响 P90、打断 200ms、后台播放边界有量化记录 | 测量结果 |
| W2-V4 | `VoiceProvider` 防腐层存在，供应商类型不外溢 | 代码审查 + 架构说明 |

### 5.6 Prompt 与成本账门禁（`B15` / `B8`）

| 门禁 | 要求 | 所需证据 |
|---|---|---|
| W2-P1 | `B15` 回归基线可跑且全绿，才允许 `B8` 提测 | 回归日志 |
| W2-P2 | 每次 AI 调用同步写 `ai_cost_logs` | backend 测试 |
| W2-P3 | 裁判与人工基准未校准前，不把 LLM-as-judge 当唯一真相 | 评估说明 |

---

## 六、执行要求

从本文生效起，按下面规则执行：

1. 入口文档负责讲范围，**本文负责讲放行门禁**
2. 任何波次完成说明，必须同时回答两件事：
   - 做了什么
   - 用什么证据证明它过了门禁
3. 对第二波薄弱点，不允许只写“已知风险”；必须写成：
   - 阻断门禁
   - 所需证据
   - 归属 issue
4. 若某门禁暂时无法自动化，必须至少有：
   - 本地 runbook
   - 手工检查清单
   - 记录位置

---

## 七、当前建议的补强动作

按优先级：

1. 给 iOS 增加 `iPhone 17 Pro` 模拟器 smoke runbook，并尽快升到 workflow
2. 给 backend 增加一条“`session.end -> worker -> review ready`”的一键 smoke runbook
3. PR 描述补“验收证据”字段，避免只写改动摘要
4. 第二波每张高风险票都带上本文对应门禁编号，避免实现与检查脱节

---

## 八、最终口径

> **FluentWork 当前已经有代码审查、CI 与 owner 审批，但还没有把第一波能力验证和第二波薄弱点检查全部收口成波次级 gate。本文生效后，第一波按能力证据放行，第二波按薄弱点门禁放行。**
