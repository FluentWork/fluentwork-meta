# FluentWork 第一波跨仓任务结构与 Skills 使用边界

**版本**：V1.0
**日期**：2026-08
**定位**：定义 FluentWork 第一波跨仓任务如何在 `meta`、`backend`、`ios` 三仓落位，并明确 Matt Pocock 风格 skills 在 Epic / ticket 拆解中的使用边界
**上游依据**：`10_FluentWork项目启动书.md`、`11_FluentWork团队分工文档.md`、`31_FluentWork后端技术方案文档.md`、`32_FluentWork-iOS App端技术设计文档.md`、`43_三仓协作与Review Workflow配置说明.md`
**变更说明**：补齐从治理文档进入第一波实际开发时，跨仓任务如何组织、如何引用技能体系、哪些内容属于真源的问题
**状态**：执行文档，可直接作为第一波任务建立的依据

---

## 一、结论

第一波开发任务不应全部堆在 `meta`，也不应直接分散到两个代码仓各写各的。

推荐结构是：

1. `meta` 放 **跨仓 Epic 与依赖关系**
2. `fluentwork-backend` 放 **后端实现 tickets**
3. `fluentwork-ios` 放 **iOS 实现 tickets**

也就是说：

> **`meta` 管方向、范围、先后顺序与跨仓依赖；代码仓管可直接开发和提 PR 的实现任务。**

---

## 二、第一波开发的目标边界

第一波不是“把全部 MVP 都拆掉”，而是只聚焦下面这条主链路：

1. 用户进入说的房间
2. 客户端创建 session
3. 客户端通过票据连上 WSS
4. AI 与用户完成至少一轮对话
5. 会话结束后落库
6. review 异步生成并可被客户端拉取

第一波**不进入**的内容：

1. corpus 语料库全量能力
2. drill 闪测
3. 每日一读
4. 话题卡
5. billing 商业化实现
6. 提审与 TestFlight 细节

---

## 三、为什么 `meta` 只放 Epic，不放全部 tickets

原因有三点：

1. `meta` 是跨仓真源，适合记录“这波要完成什么闭环”，不适合承接大量 repo 内碎任务；
2. backend / ios 的 tickets 最终都要直接对应代码改动、测试、PR 和 owner 审批，放回代码仓更自然；
3. 如果把实现 tickets 全堆在 `meta`，后续很容易出现 issue 与 PR 脱节、责任边界模糊、状态同步成本过高。

因此本轮采用：

- `meta`: Epic + 跨仓任务地图
- `backend` / `ios`: repo tickets

---

## 四、Matt Pocock 风格 skills 是否冲突

**不冲突，但不能替代 `meta` 的任务真源。**

### 4.1 可以怎么用

Matt Pocock 风格 skills 适合用于：

1. 把一个 repo 内的实现工作继续细分；
2. 把大 ticket 拆成更小的 coding tasks；
3. 为单仓任务生成测试、验收点、代码范围建议；
4. 辅助编写 repo 内的实现说明。

### 4.2 不该怎么用

不建议让它直接负责：

1. 定义跨仓 Epic 的边界；
2. 决定 iOS 和 backend 谁先做什么；
3. 替代项目治理文档成为第一真源；
4. 在多个仓之间各自维护一套不同版本的任务结构。

### 4.3 本项目的正确边界

在 FluentWork 当前阶段，正确做法是：

1. 先在 `meta` 固定第一波跨仓 Epic 和依赖顺序；
2. 再在 `backend` / `ios` 中使用 Matt Pocock 风格 skills 去细化单仓 tickets；
3. 单仓细化后的结果必须回链到 `meta` 的 Epic。

统一口径：

> **skills 可以帮助拆 repo 内 tasks，但 `meta` 才是跨仓任务结构的真源。**

---

## 五、第一波 Epic 放在 `meta` 的建议

建议在 `meta` 中只建立 3 个 Epic。

## Epic 1：说的房间契约冻结

目标：

冻结第一波开发所需的 iOS / backend 契约，避免两边并行时反复返工。

包含：

1. REST 契约冻结
2. WSS schema 冻结
3. iOS SpeechSession 事件与状态边界冻结
4. 游客身份与 `merge` 策略冻结
5. `session.end -> review` 的最小闭环口径冻结

验收标准：

1. `POST /sessions`、`POST /auth/guest`、`POST /account/merge`、`GET /sessions/:id/review`、`POST /sessions/:id/messages` 口径明确
2. WSS 控制帧与音频帧字段明确
3. iOS 与 backend 都能引用同一版 schema / 文档

## Epic 2：说的房间主链路打通

目标：

打通“创建 session -> WSS 连接 -> 一轮对话 -> 会话结束”的最小可运行链路。

包含：

1. backend `app-server` session 建立
2. backend `voice-gateway` WSS 主链路
3. iOS `SocketTransport`
4. iOS `APIClient`
5. iOS 房间状态接线与基础 UI 壳

验收标准：

1. 客户端可完成一次 session 创建与握手
2. 客户端与网关可完成至少一轮消息收发
3. 会话结束后服务端能记录最小 session 数据

## Epic 3：review 异步闭环接通

目标：

把 `session.end` 后的异步任务、review 查询接口和 iOS 轮询消费打通，形成最小回顾闭环。

包含：

1. backend `ai-worker` 骨架
2. `session.finished` 事件与幂等键
3. `GET /sessions/:id/review`
4. iOS review 轮询与骨架态
5. 失败重试与最小错误态

验收标准：

1. 会话结束能触发异步任务
2. review 未完成时客户端可轮询
3. review 完成后客户端能拿到最小回顾数据

---

## 六、第一波 backend tickets 建议

下面这些 tickets 建议建立在 `fluentwork-backend`。

## B1. 游客身份与账号基础接口

范围：

1. `POST /auth/guest`
2. `POST /account/merge`
3. 基础 token / device_id 幂等处理

依赖：

- Epic 1

验收重点：

1. 游客身份可签发
2. `merge` 幂等
3. 会话与业务表可挂在游客身份下

## B2. session 创建接口

范围：

1. `POST /sessions`
2. 创建 session 记录
3. 返回 `wss_url + ticket + session_id`

依赖：

- B1

验收重点：

1. session 票据可校验
2. ticket 有时效
3. 接口错误模型统一

## B3. voice-gateway WSS 协议骨架

范围：

1. WSS 握手
2. ticket 校验
3. 控制帧 / 音频帧 schema
4. ping / reconnect 基础支持

依赖：

- Epic 1
- B2

验收重点：

1. 客户端能成功建连
2. 协议字段与文档一致
3. 至少支持一轮最小消息交互

## B4. 会话结束落库与最小持久化

范围：

1. `session.end`
2. utterance / session 最小记录
3. session 状态更新

依赖：

- B3

验收重点：

1. 会话结束后数据能落库
2. 重复结束事件不造成脏写
3. 后续 review 任务可读取必要上下文

## B5. review 异步任务骨架

范围：

1. `session.finished` 事件
2. Redis Stream / worker 消费骨架
3. 幂等键 `session_id + task_type`

依赖：

- B4

验收重点：

1. 会话结束能投递任务
2. worker 能消费并更新状态
3. 失败路径可重试一次

## B6. review 查询接口

范围：

1. `GET /sessions/:id/review`
2. review 未就绪 / 已就绪 / 失败三态
3. 最小返回模型

依赖：

- B5

验收重点：

1. iOS 可轮询
2. 状态模型稳定
3. 错误码口径明确

## B7. 文本降级接口占位

范围：

1. `POST /sessions/:id/messages`
2. 只提供最小可用占位
3. 在语音可用时返回 409

依赖：

- Epic 1

验收重点：

1. 降级口径存在
2. iOS 可先接线
3. 不阻塞第一波主链路

---

## 七、第一波 iOS tickets 建议

下面这些 tickets 建议建立在 `fluentwork-ios`。

## I1. C0 状态管理基座

范围：

1. TGReduxKit 接入
2. Factory 容器
3. `AppState`
4. Root Store
5. TestStore 基建

依赖：

- iOS 技术文档既有定案

验收重点：

1. 全工程零 Combine
2. 根 Store 可运行
3. reducer / middleware 测试基线存在

## I2. SpeechSession 契约接线骨架

范围：

1. 接入人写状态机接口
2. 事件到 Action 的映射
3. SideEffect 协议边界

依赖：

- Epic 1
- I1

验收重点：

1. 不修改人写状态机本体
2. reducer 与 middleware 责任清晰
3. 事件表与技术文档一致

## I3. SocketTransport

范围：

1. WSS 建连
2. ping / reconnect
3. 控制帧与音频帧编解码
4. 序列号丢帧纯函数

依赖：

- Epic 1
- I1

验收重点：

1. 能消费 backend WSS schema
2. 丢帧规则有单测
3. 失败态可上抛到 Store

## I4. APIClient 会话接口

范围：

1. `POST /auth/guest`
2. `POST /sessions`
3. `GET /sessions/:id/review`
4. `POST /account/merge`
5. `POST /sessions/:id/messages`

依赖：

- Epic 1
- I1

验收重点：

1. 错误归一化
2. ticket / session 创建链路可跑通
3. review 轮询接口已可用

## I5. 说的房间主链路 UI 壳

范围：

1. 房间页面基础布局
2. 状态切换
3. 说话键 / 转录浮层 / 基础消息区占位
4. 不做完整视觉精修

依赖：

- I2
- I3
- I4

验收重点：

1. 能承载主链路状态流转
2. 错误态与 loading 态可见
3. 不直接依赖底层音频或 transport

## I6. review 轮询与骨架态

范围：

1. 回顾页最小 state
2. 轮询 `GET /sessions/:id/review`
3. 骨架态 / ready / failed 三态

依赖：

- I4
- Epic 3

验收重点：

1. review 未完成时前端行为明确
2. 完成后可展示最小结果
3. 重试路径存在

---

## 八、跨仓依赖顺序

第一波推荐按这个顺序推进：

1. `meta` 建 Epic 1、2、3
2. backend 建 B1、B2、B3
3. ios 建 I1、I2、I3、I4
4. backend 建 B4、B5、B6
5. ios 建 I5、I6

换句话说：

> **先冻结契约，再起后端 session / gateway 骨架，再起 iOS 消费层，最后接 review 闭环。**

---

## 九、Issue 命名与链接建议

为了避免多仓混乱，建议用下面的口径：

### `meta`

- `EPIC: 说的房间契约冻结`
- `EPIC: 说的房间主链路打通`
- `EPIC: review 异步闭环接通`

### `backend`

- `B1: add guest auth and merge endpoints`
- `B2: add session creation endpoint`
- `B3: scaffold voice-gateway WSS protocol`
- `B4: persist session end data`
- `B5: scaffold review worker pipeline`
- `B6: add review polling endpoint`
- `B7: add degraded text messages endpoint`

### `ios`

- `I1: add root store and DI baseline`
- `I2: wire SpeechSession reducer boundary`
- `I3: add socket transport and frame decoding`
- `I4: add session API client`
- `I5: scaffold speaking room screen`
- `I6: add review polling and skeleton state`

链接规则：

1. backend / ios ticket 必须回链到所属 `meta` Epic
2. PR 必须回链到 repo ticket
3. `meta` Epic 不直接对应代码 PR，而是对应一组 repo tickets 的完成状态

---

## 十、最终口径

FluentWork 第一波跨仓任务结构的标准口径应统一为：

1. `meta` 只放跨仓 Epic，不放全部实现 tickets；
2. `backend` / `ios` 各自持有 repo 内 tickets；
3. Matt Pocock 风格 skills 可以辅助单仓 ticket 继续细分，但不能替代 `meta` 的跨仓真源；
4. 第一波只打“session -> WSS -> 一轮对话 -> session.end -> review”这条主链路；
5. 语料库、闪测、每日一读、billing 不进入第一波。
