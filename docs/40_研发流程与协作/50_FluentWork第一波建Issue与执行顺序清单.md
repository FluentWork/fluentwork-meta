# FluentWork 第一波建 Issue 与执行顺序清单

**版本**：V1.0
**日期**：2026-08
**定位**：把第一波 Epic、repo issue 和 PR 级任务串成可执行顺序，供实际在 GitHub 建 issue 与开始编码时直接使用
**上游依据**：`45_FluentWork-meta第一波Epic草案.md`、`46_FluentWork-backend第一波Issue草案.md`、`47_FluentWork-ios第一波Issue草案.md`、`49_FluentWork重点Issue-PR级任务清单.md`
**变更说明**：补齐从任务结构到实际建 issue / 开发推进之间的执行顺序

---

## 一、目的

前面的文档已经完成了三层拆分：

1. `meta` Epic
2. `backend` / `ios` 一级 issue
3. 重点 issue 的 PR 级任务

这份清单只做一件事：

> **告诉你第一波应该按什么顺序建 issue，按什么顺序推进 PR。**

它不新增新的任务边界，只把前面已有内容串起来。

---

## 二、先建哪些 Issue

推荐先建 issue，再开始写代码。

建 issue 的顺序建议如下：

### Step 1：先在 `meta` 建 3 个 Epic

顺序：

1. `EPIC: 说的房间契约冻结`
2. `EPIC: 说的房间主链路打通`
3. `EPIC: review 异步闭环接通`

原因：

1. 先有跨仓真源，backend / ios 的实现票才有统一挂点；
2. 后续新增 repo issues 时不容易漂浮；
3. 依赖关系能先固定下来。

### Step 2：再在 `fluentwork-backend` 建第一波 issue

顺序：

1. `B1: add guest auth and merge endpoints`
2. `B2: add session creation endpoint`
3. `B3: scaffold voice-gateway WSS protocol`
4. `B4: persist session end data`
5. `B5: scaffold review worker pipeline`
6. `B6: add review polling endpoint`
7. `B7: add degraded text messages endpoint`

原因：

1. backend 是第一波主链路的先导面；
2. iOS 的 `I3` / `I4` 需要依赖 B1、B2、B3、B6、B7 的契约。

### Step 3：最后在 `fluentwork-ios` 建第一波 issue

顺序：

1. `I1: add root store and DI baseline`
2. `I2: wire SpeechSession reducer boundary`
3. `I3: add socket transport and frame decoding`
4. `I4: add session API client`
5. `I5: scaffold speaking room screen`
6. `I6: add review polling and skeleton state`

原因：

1. iOS 虽然依赖 backend 契约，但 `I1` 可以先独立起；
2. `I2-I5` 更适合在 backend 主协议有雏形后接线；
3. `I6` 应晚于 review 查询接口。

---

## 三、Issue 建立时的统一字段

每个 issue 建议都带下面这些字段：

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

## 四、真正开始编码时的顺序

如果你准备正式开始推进第一波，建议按下面顺序：

### Phase A：契约和协议立起来

1. `B1`
2. `B2`
3. `B3-PR1`
4. `I1`
5. `I3-PR1`
6. `B3-PR2`
7. `I3-PR2`
8. `I4`

目标：

1. 游客身份、session 创建、WSS 握手都先站起来；
2. iOS 先有可消费的连接和 API 层。

### Phase B：说的房间主链路接通

1. `B3-PR3`
2. `I3-PR3`
3. `I2`
4. `I5-PR1`
5. `I5-PR2`
6. `B4`

目标：

1. 客户端和 gateway 能完成至少一轮最小对话；
2. 会话结束可以落库。

### Phase C：review 异步闭环接通

1. `B5-PR1`
2. `B5-PR2`
3. `B5-PR3`
4. `B5-PR4`
5. `B6`
6. `I6`

目标：

1. review 异步任务可跑；
2. iOS 可轮询 review 结果。

### Phase D：补错误态、心跳和降级占位

1. `I5-PR3`
2. `I5-PR4`
3. `I3-PR4`
4. `B3-PR4`
5. `B7`

目标：

1. 把主链路从“能跑”补到“能被失败和异常承载”；
2. 给文本降级预留落点。

---

## 五、每个仓同时开的任务上限

为了避免超小团队把上下文打散，建议控制并行度：

### `meta`

同时进行：

1. 维护 Epic 状态
2. 不额外开新的大批任务

并行上限：

- `1`

### `fluentwork-backend`

同时进行：

1. 一个协议 / 接口类 PR
2. 一个异步 / review 类 PR

并行上限：

- `2`

### `fluentwork-ios`

同时进行：

1. 一个基座 / transport 类 PR
2. 一个 UI 壳 / 状态接线类 PR

并行上限：

- `2`

总原则：

> **backend 与 ios 可并行，但每个仓内部不要同时开太多横切任务。**

---

## 六、什么时候再新增 Epic

只有满足下面任一条件，才建议继续新增 `meta` Epic：

1. 出现新的跨仓闭环
2. 需要新的跨仓验收标准
3. 单靠 repo issue 已经无法表达依赖关系

不建议因为这些原因新增 Epic：

1. 某个 repo issue 太大
2. 想把同一仓的几个技术任务包起来
3. 只是为了让 issue 看起来更整齐

换句话说：

> **repo 内问题继续拆 issue；跨仓闭环才升 Epic。**

---

## 七、什么时候再用 Matt 风格 tools

当前最推荐的用法是：

1. 先按本文件把 issue 建起来
2. 再只对下面这些点继续使用 Matt 风格 tools：
   - `B3-PR2`
   - `B3-PR4`
   - `I3-PR4`
   - `I5-PR4`

原因：

1. 这些点更偏状态传播、协议校验、错误处理
2. 它们最容易在一个 PR 里越写越大
3. 现在已经有稳定边界，继续细化时不容易跑偏

不建议继续用它来做的事：

1. 重新拆 `meta` Epic
2. 重写一级 issue 标题
3. 改写已经冻结的跨仓范围

---

## 八、建议的下一步动作

如果你接下来继续让我推进，最顺的顺序是：

1. 先把这批文档推到远端
2. 再在 `meta` 建 3 个 Epic
3. 再在 `backend` / `ios` 建第一波 issue
4. 最后只对最难的 4 个 PR 级任务继续下钻

---

## 九、最终口径

FluentWork 第一波的实际推进顺序应统一为：

1. 先 `meta` Epic
2. 再 `backend` / `ios` issue
3. 再 PR 级任务
4. 最后才在必要处继续细分

这套顺序的目的不是把文档写得更漂亮，而是让：

1. 任务边界稳定
2. backend / ios 不互相等待过久
3. 每个 PR 都能被清楚评审和回滚
