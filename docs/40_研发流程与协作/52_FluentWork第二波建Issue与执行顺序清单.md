# FluentWork 第二波建 Issue 与执行顺序清单

**版本**：V1.0
**日期**：2026-08
**定位**：把第二波 Epic、repo issue 和 PR 级推进顺序串成可执行清单，供实际在 GitHub 建 issue 与开始编码时直接使用
**上游依据**：`51_FluentWork第二波跨仓任务结构与Issue草案.md`、`50_FluentWork第一波建Issue与执行顺序清单.md`
**变更说明**：第一波收尾后，补齐第二波从任务结构到实际建 issue / 开发推进之间的执行顺序

---

## 一、目的

`51` 号文档完成了第二波的三层拆分准备：

1. `meta` Epic（5 个）
2. `backend` / `ios` 一级 issue（`B8-B14` / `I7-I12`）
3. 各 issue 的范围与验收重点

这份清单只做一件事：

> **告诉你第二波应该按什么顺序建 issue，按什么顺序推进 PR，以及第一波遗留的 issue 状态如何收尾。**

它不新增新的任务边界，只把已有内容串起来。

---

## 二、第一波 issue 收尾动作（已完成）

在建第二波 issue 之前，先把第一波已完成但仍是 OPEN 的 issue 关闭：

### `fluentwork-meta`

1. `EPIC: 说的房间契约冻结`
2. `EPIC: 说的房间主链路打通`
3. `EPIC: review 异步闭环接通`

### `fluentwork-backend`

1. `B3: scaffold voice-gateway WSS protocol`
2. `B4: persist session end data`
3. `B5: scaffold review worker pipeline`
4. `B6: add review polling endpoint`

### `fluentwork-ios`

1. `I5: scaffold speaking room screen`
2. `I6: add review polling and skeleton state`

关闭口径：

1. 关闭评论附上对应合入的 commit / PR 记录
2. 显式注明：review worker 当前为 stub，"评价内容真实现"由第二波 `B8` 承接，回链 `EPIC: 回顾与炼化闭环`
3. 状态注记：上述关闭动作已全部执行完毕（三仓对应 issue 均已 CLOSED）

---

## 三、第二波正式启动前，先做第一波阻断遗留清账

在建第二波实现票、开写第二波代码之前，先完成下面三项：

1. iOS 第一波 `iPhone 17 Pro` 模拟器 smoke runbook
2. backend 第一波 `session.end -> worker -> review ready` live smoke runbook
3. `meta` 第一波关闭记录

说明：

1. 这三项不是第二波功能范围，而是第一波验证欠账；
2. 它们解决的是“第一波是否真的过关”的问题，不解决，不应视为第二波正式开始；
3. 详细计划见 `docs/50_测试与验收/52_FluentWork第一波遗留问题清账与第二波启动前计划.md`。

---

## 四、先建哪些 Issue

推荐先建 issue，再开始写代码。建 issue 的顺序建议如下：

### Step 1：先在 `meta` 建 5 个 Epic

顺序：

1. `EPIC: 回顾与炼化闭环`
2. `EPIC: 语料库基础版`
3. `EPIC: 每日一读`
4. `EPIC: 即时反馈命中检测`
5. `EPIC: 语音链路火山做实`

原因：

1. 先有跨仓真源，第二波实现票才有统一挂点；
2. 命中检测依赖语料库数据，Epic 顺序同时表达依赖方向；
3. 语音链路火山做实是第一波欠账，作为 Phase 0 并行轨单独挂 Epic，不混入功能面 Epic。

### Step 2：再在 `fluentwork-backend` 建第二波 issue

顺序：

1. `B8: implement review evaluation and refine in ai-worker`
2. `B9: extend review endpoint to full model`
3. `B10: add corpus module and phrase_blocks migration`
4. `B11: add content module and daily reads`
5. `B12: add B7 hit detection path`
6. `B13: connect voice-gateway to Volcano realtime speech`
7. `B14: run end-to-end text injection POC`
8. `B15: build offline evaluation dataset and prompt regression baseline`

原因：

1. `B8` / `B9` 是 iOS 回顾页（`I7`）的前置；
2. `B10` 是入库与命中检测的共同数据基础；
3. `B12` 放最后，等语料数据就位；
4. `B13` / `B14` 属 Phase 0 并行轨：`B14` 不依赖网关接入，应最先开始；`B12` 的启动以 `B14` 结论回写为前提；
5. `B15` 是 `B8` 的质量前置（见 61 号文档难点 2）：`B8` 提测前，回归基线必须可跑且全绿。

### Step 3：最后在 `fluentwork-ios` 建第二波 issue

顺序：

1. `I7: add full review page`
2. `I8: add refine cards accept flow`
3. `I9: add corpus page with local-first store`
4. `I10: add daily read page`
5. `I11: add badge feedback display`
6. `I12: replace placeholder audio engine with real pipeline`
7. `I13: draft corpus local-first sync strategy decision doc`

原因：

1. `I7` 依赖 `B9`，但渐进加载骨架可先行；
2. `I9` 的本地缓存层可在契约冻结后独立起步；
3. `I11` 必须晚于 `B12` 的帧契约；
4. `I12` 属 Phase 0，与 `B13` 并行，联调依赖网关真链路；
5. `I13` 是 `I9` 的前置（见 61 号文档难点 3）：同步策略定案回写后再动码。

---

## 五、Issue 建立时的统一字段

沿用第一波口径，每个 issue 都带下面这些字段：

1. `Goal`
2. `Scope`
3. `Acceptance Criteria`
4. `Out of Scope`
5. `Dependencies`
6. `Upstream Epic`
7. `References`

统一要求：

1. `meta` Epic 不直接绑 PR
2. `backend` / `ios` issue 必须回链到 `meta` Epic
3. PR 必须回链到 repo issue，而不是只链 `meta`

---

## 六、真正开始编码时的顺序

### 启动前 Gate（阻断）

1. 第一波阻断遗留三项已清账
2. `meta` 已有第一波关闭记录
3. 只有满足上述条件，后面的 Phase 0 与功能面开发才算第二波正式启动

### Phase 0（并行轨）：语音链路火山做实

1. `B14`（注入 POC，不依赖网关，最先开始）
2. `B13` + `I12`（网关接入与 iOS 音频引擎并行）

目标：

1. 注入窗口档位（①/②/③）在 `B12` 启动前回写 `meta`；
2. 真实语音对话可跑通，首响 P90 ≤ 1.5s 的 Go/No-Go 结论落定。

说明：

1. Phase 0 不改变 Phase A → D 的任务边界与编号，只占用各仓既有并行额度（见第六节）；
2. Phase 0 与 Phase A 可同时开始：`B14` 与 `B8` 并行，`I12` 与 `I7` 骨架并行。

### Phase A：回顾做实

1. `B8`
2. `B9`
3. `I7`

目标：

1. 评价与炼化结果真实生成并落库；
2. iOS 完整回顾页可消费全量模型。

### Phase B：入库闭环

1. `B10`
2. `I8`
3. `I9`

目标：

1. 炼化卡可入库，语料库可浏览检索；
2. "说 → 读 → 入库"流水线完整贯通（W5 出口）。

### Phase C：每日一读

1. `B11`
2. `I10`

目标：

1. 每日一读按时生成并可消费；
2. 跟读只录音不出分（W6 出口）。

### Phase D：即时反馈

1. `B12`
2. `I11`

目标：

1. 命中检测不阻塞主链路；
2. 徽章展示可见，命中反馈闭环完成。

---

## 七、每个仓同时开的任务上限

沿用第一波口径，避免超小团队把上下文打散：

### `meta`

- 维护 Epic 状态，不额外开新的大批任务
- 并行上限：`1`

### `fluentwork-backend`

- 一个接口 / 模块类 PR + 一个异步 / 生成类或语音链路类 PR（Phase 0 期 `B13`/`B14` 占其一）
- 并行上限：`2`

### `fluentwork-ios`

- 一个页面 / 状态接线类 PR + 一个本地缓存 / 同步类或音频链路类 PR（Phase 0 期 `I12` 占其一）
- 并行上限：`2`

总原则：

> **backend 与 ios 可并行，但每个仓内部不要同时开太多横切任务；Phase 0 与 Phase A 并行时不额外扩容。**

---

## 八、什么时候再新增 Epic

沿用第一波口径：只有出现新的跨仓闭环、新的跨仓验收标准，或单靠 repo issue 已无法表达依赖关系时，才新增 `meta` Epic。

已知候选（不在第二波）：

1. 闪测 + 调度引擎（W7，需要服务端调度引擎与客户端卡流两个面的跨仓验收）
2. 话题建议批处理（W7）
3. 创建练习弹层与素材模块（第一波遗留项）

---

## 九、最终口径

FluentWork 第二波的实际推进顺序应统一为：

1. 先收尾第一波 OPEN issue
2. 再清掉第一波阻断遗留（模拟器 smoke、`review ready` live smoke、关闭记录）
3. 再 `meta` 建 5 个 Epic（含 Phase 0 语音做实）
4. 再 `backend` / `ios` 建第二波 issue（含 `B13`/`B14`/`B15`/`I12`/`I13`）
5. 编码按启动前 Gate → Phase 0（并行）+ Phase A → B → C → D 推进，每仓并行不超过 2 个 PR

这套顺序的目的与第一波相同：

1. 任务边界稳定
2. backend / ios 不互相等待过久
3. 每个 PR 都能被清楚评审和回滚
