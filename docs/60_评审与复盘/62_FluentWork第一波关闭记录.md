# FluentWork 第一波关闭记录

**版本**：V1.0  
**日期**：2026-08-30  
**定位**：给第二波一个硬基线——第一波通过了哪些门禁、证据落在哪里、哪些遗留继续挂账  
**上游依据**：`51_FluentWork第一波能力验证与第二波薄弱点检查门禁.md`、`52_FluentWork第一波遗留问题清账与第二波启动前计划.md`、`61_FluentWork第二波关键难点进度与缺口分析.md`

---

## 一、结论句

> **第一波阻断性验证欠账已清：iOS `iPhone 17 Pro` 模拟器 smoke、backend `session.end -> worker -> review ready` live smoke、以及本关闭记录均已落地。第二波从该基线进入 Phase 0（`B14` / `B15` / `I13`），再进入功能面 Phase A–D。练习弹层、素材模块等范围遗留继续挂账，不插入第二波启动门槛。**

---

## 二、第一波做了什么（能力边界）

### 已打通

1. 游客鉴权与 session 创建 / 票据握手
2. voice-gateway WSS 协议骨架与最小会话循环（stub）
3. `session.end` 落库 + `session.finished` outbox
4. review worker stub 生成 + `GET /sessions/:id/review` 三态轮询
5. iOS Root Store / Transport / Speaking Room 壳 + review 骨架入口
6. 文本降级末级兜底（`channel: text`）

### 明确未做实（留给第二波，不是本关闭记录的阻断项）

1. 真实火山语音链路与首响指标（`B13` / `I12` / Epic 语音）
2. 真实评价 + 炼化 LLM（`B8` / `B9` / `I7`）
3. 语料库 / 每日一读 / badge 命中（`B10-B12` / `I8-I11`）

---

## 三、门禁通过证据

### 3.1 自动化证据

| 仓 | 证据 | 状态 |
|---|---|---|
| `fluentwork-ios` | `swift test`（含状态机 / transport / session API / launch→navigation） | 已具备 |
| `fluentwork-backend` | `go test ./...` / `./scripts/dev-check.sh` | 已具备 |
| 三仓 | commit 前 `gstack /review` + CI required checks + owner 审批 | 已具备 |

### 3.2 活体 / runbook 证据（本次清账补齐）

| 编号 | 清账项 | 产出 | 状态 |
|---|---|---|---|
| P0-1 / W1-START-1 | iOS `iPhone 17 Pro` 模拟器 smoke | `fluentwork-ios/Scripts/smoke-iphone17pro.sh` + `docs/06_第一波iPhone17Pro_Smoke_Runbook.md` | 已落地 |
| P0-2 / W1-START-2 | backend `review ready` live smoke | `fluentwork-backend/scripts/smoke-review-ready.sh` + `cmd/smoke-review-ready` + `docs/03_第一波_review_ready_Smoke_Runbook.md` | 已落地 |
| P0-3 / W1-START-3 | 第一波关闭记录 | 本文档 | 已落地 |

### 3.3 执行命令与实测（2026-08-30）

```bash
# iOS — 实测 PASS：Host launch on iPhone 17 Pro + launch/bootstrap 3 tests green
cd fluentwork-ios && ./Scripts/smoke-iphone17pro.sh

# backend — 实测 PASS：review_status=ready, generator=stub-v1
cd fluentwork-backend && ./scripts/smoke-review-ready.sh
```

---

## 四、非阻断挂账（不插入第二波启动）

1. `C4` 创建练习弹层与素材输入
2. `A1-A2` 素材模块与提炼
3. 更重型 release gate / 真机验证自动化
4. CI 将模拟器 smoke 升为 required check（runbook 稳定后再做）

处理原则：继续挂 backlog，不冒充质量门禁，不挤占第二波 Phase 0 窗口。

---

## 五、第二波从哪里起步

统一启动定义：

```text
第一波阻断遗留清账完成（本记录）
  -> Phase 0 前置：B14 / B15 / I13
  -> Phase A–D 功能开发
```

推荐下一步（不在本关闭记录范围内，仅指路）：

1. `B14` 端到端文本注入 POC（最先开始，`B12` 门禁）
2. `B15` 离线评估集与 Prompt 回归基线（`B8` 提测门禁）
3. `I13` 语料库本地优先同步策略定案（`I9` 前置）

详细顺序见：

1. `52_FluentWork第一波遗留问题清账与第二波启动前计划.md`
2. `52_FluentWork第二波建Issue与执行顺序清单.md`
3. `61_FluentWork第二波关键难点进度与缺口分析.md`
