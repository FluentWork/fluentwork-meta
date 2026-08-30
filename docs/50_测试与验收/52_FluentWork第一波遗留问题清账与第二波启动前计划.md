# FluentWork 第一波遗留问题清账与第二波启动前计划

**版本**：V1.0  
**日期**：2026-08  
**定位**：在正式开始第二波实现前，先把第一波留下的阻断项与非阻断项分层收口，形成明确清账顺序、验收标准与启动门槛  
**上游依据**：`51_FluentWork第一波能力验证与第二波薄弱点检查门禁.md`、`52_FluentWork第二波建Issue与执行顺序清单.md`、`61_FluentWork第二波关键难点进度与缺口分析.md`、第一波入口文档  
**状态**：执行中（P0-1 / P0-2 / P0-3 产物已落地，见 `62_FluentWork第一波关闭记录.md`）

---

## 一、结论

在当前口径下，**第二波不应直接进入功能开发**。

原因不是第一波功能没做完，而是第一波还有两类遗留：

1. **阻断启动的质量遗留**：第一波主链路已经实现，但还没有完成足够的能力放行验证
2. **不阻断启动的范围遗留**：例如练习弹层、素材模块，属于未纳入第一波的产品范围，不应混入第二波启动门槛

因此，正式开始第二波前，必须先清掉阻断启动的质量遗留；范围遗留继续挂账，不插入第二波。

---

## 二、什么算“第一波遗留问题”

第一波遗留不能混成一类。这里统一分成两层：

### 2.1 阻断第二波启动的遗留

这类问题不解决，第二波会在“验证欠账未清”的状态下带病开工。

1. iOS 没有固定的 `iPhone 17 Pro` 模拟器 smoke runbook / gate
2. backend 没有固定的 `session.end -> worker -> review ready` live smoke runbook
3. 三仓还没有把第一波“做了什么”和“如何证明过关”收成一份启动前清账记录

### 2.2 不阻断第二波启动的遗留

这类问题是范围欠账，不应冒充质量门禁。

1. `C4` 创建练习弹层
2. `A1-A2` 素材模块
3. 更重型的 release gate / 真机验证自动化

处理原则：

> **阻断项先清账；范围欠账继续挂 backlog，不挤占第二波启动窗口。**

---

## 三、当前盘点

### 3.1 已具备的第一波资产

1. iOS `swift test` 已通过，且有状态机、transport、session/review API、导航链路测试
2. backend `go test ./...` 已通过，且有 account / session / review worker / gateway 契约测试
3. backend 已能本地 smoke 到 `guest -> session -> review pending -> text degrade`
4. `gstack /review`、CI required checks、owner 审批三层门禁已存在

### 3.2 仍缺的第一波放行证据

1. **iOS 活体验证缺口**：没有一条稳定 runbook 证明第一波链路能在 `iPhone 17 Pro` 模拟器上跑通
2. **backend 活体验证缺口**：没有一条稳定 runbook 证明 `session.end` 后 worker 真能把 review 推到 `ready`
3. **跨仓收口缺口**：没有一份“第一波启动关闭证明”，让第二波明确建立在已验证基线上

---

## 四、第二波正式启动前必须完成的清账项

以下三项为**启动门槛**。未完成前，不应宣称第二波已正式开始。

| 编号 | 清账项 | 仓库 | 产出物 | 验收标准 |
|---|---|---|---|---|
| P0-1 | iOS 第一波模拟器 smoke runbook | `fluentwork-ios` | runbook 文档 + 可执行命令 | 在 `iPhone 17 Pro` 模拟器上可重复跑通最小链路 |
| P0-2 | backend 第一波 `review ready` live smoke runbook | `fluentwork-backend` | runbook 脚本或文档 | 可重复证明 `session.end -> worker -> review ready` |
| P0-3 | 第一波关闭记录 | `fluentwork-meta` | 清账记录 / 验收摘要 | 明确列出通过证据与未纳入第二波的挂账项 |

---

## 五、建议执行顺序

### Step 0：先做清账，不开第二波实现票

顺序：

1. `P0-1` iOS 模拟器 smoke runbook
2. `P0-2` backend `review ready` live smoke
3. `P0-3` `meta` 关闭记录

原因：

1. 这三项直接对应第一波当前最真实的验证缺口
2. 不先补，第二波所有功能票都建立在“第一波已完成”的口头假设上
3. 清账成本低于第二波开工后返工成本

### Step 1：清账完成后，再进入第二波 Phase 0

1. `B14`
2. `B15`
3. `I13`

原因：

1. 这三项是第二波真正的“先定案 / 先验证”前置
2. 它们不应与第一波清账混在一起，但必须在功能开发前完成

### Step 2：再进入功能面 Phase A-D

1. Phase A：`B8` / `B9` / `I7`
2. Phase B：`B10` / `I8` / `I9`
3. Phase C：`B11` / `I10`
4. Phase D：`B12` / `I11`

---

## 六、每项清账的具体要求

### 6.1 P0-1 iOS 模拟器 smoke runbook

目标：

证明第一波 iOS 不是只有 package tests，而是真的能在指定模拟器上完成最小链路验证。

最低内容：

1. 固定机型：`iPhone 17 Pro`
2. 固定启动方式：`xcodebuild test`、UI smoke，或本地 runbook 脚本
3. 至少证明：
   - app 启动成功
   - bootstrap ready
   - 可进入第一波主路径对应页面
   - review 骨架或 mock 首波能力可见

通过标准：

1. 本地可重复执行
2. 输出可截图或日志化
3. 后续可升为 workflow

### 6.2 P0-2 backend `review ready` live smoke

目标：

证明第一波 backend 不只是“review pending + text degrade”能通，而是 worker 异步闭环也能活体到 `ready`。

最低内容：

1. guest auth
2. create session
3. 触发 `session.end`
4. 轮询 review 到 `ready`
5. 输出 `generator` / `status` / 关键字段样例

通过标准：

1. 不依赖手工改库
2. 可本地重复执行
3. 明确失败时看哪里：app-server / worker / queue / db

### 6.3 P0-3 第一波关闭记录

目标：

给第二波一个“从哪里起步”的硬基线。

最低内容：

1. 第一波通过了哪些门禁
2. 证据落在哪些测试 / runbook / 文档
3. 哪些遗留属于非阻断挂账
4. 结论句：第二波从什么前提开始

---

## 七、第二波启动定义

从现在起，第二波“正式开始”的定义收紧为：

```text
第一波阻断遗留清账完成
  -> Phase 0 前置（B14 / B15 / I13）起步
  -> Phase A-D 功能开发
```

以下情况都不算“第二波正式开始”：

1. 只建了第二波 issue，但第一波阻断遗留未清
2. 直接进入 `B8` / `I7` / `B10` 开发，但第一波 smoke gate 还没补
3. 用第一波已有单测替代 live smoke 证据

---

## 八、更新后的统一口径

> **第二波启动前，先清第一波阻断性遗留：iOS 模拟器 smoke、backend review ready live smoke、跨仓关闭记录。只有这些验证欠账清完，第二波才进入 Phase 0 与后续功能开发；练习弹层、素材模块等范围遗留继续挂账，不插入第二波。**
