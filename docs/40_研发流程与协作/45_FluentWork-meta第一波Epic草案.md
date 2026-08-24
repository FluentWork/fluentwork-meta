# FluentWork meta 第一波 Epic 草案

**版本**：V1.0
**日期**：2026-08
**定位**：提供可直接录入 GitHub 的第一波跨仓 Epic 草案，作为 `fluentwork-meta` 的任务真源
**上游依据**：`44_FluentWork第一波跨仓任务结构与Skills使用边界.md`、`31_FluentWork后端技术方案文档.md`、`32_FluentWork-iOS App端技术设计文档.md`
**变更说明**：将第一波跨仓任务结构转写为 `meta` 仓可直接使用的 Epic 草案

---

## 一、使用说明

本文件中的每个 Epic 都建议建立在 `fluentwork-meta`，用于承接：

1. 跨仓目标
2. 范围边界
3. 依赖顺序
4. backend / ios 子任务链接

每个 Epic 建立后：

1. 在 `backend` / `ios` 仓建立对应 implementation issues；
2. 将子 issue 链接回本 Epic；
3. 由本 Epic 跟踪跨仓完成状态，而不是直接对应单个 PR。

---

## Epic 1：说的房间契约冻结

**建议标题**

`EPIC: 说的房间契约冻结`

**建议 labels**

`meta` `epic` `spec` `backend` `ios`

**建议正文**

```md
## 目标

冻结第一波开发所需的 iOS / backend 契约，避免两边并行时反复返工。

## 范围

- REST 契约冻结
- WSS schema 冻结
- iOS SpeechSession 事件与状态边界冻结
- 游客身份与 `merge` 策略冻结
- `session.end -> review` 最小闭环口径冻结

## 验收标准

- `POST /sessions`、`POST /auth/guest`、`POST /account/merge`、`GET /sessions/:id/review`、`POST /sessions/:id/messages` 口径明确
- WSS 控制帧与音频帧字段明确
- iOS 与 backend 都能引用同一版 schema / 文档

## 不包含

- corpus / drill / 每日一读
- billing 商业化实现
- 完整视觉精修

## 下挂子任务

- backend: B1, B2, B3, B7
- ios: I2, I3, I4

## 上游依据

- `31_FluentWork后端技术方案文档.md`
- `32_FluentWork-iOS App端技术设计文档.md`
- `44_FluentWork第一波跨仓任务结构与Skills使用边界.md`
```

---

## Epic 2：说的房间主链路打通

**建议标题**

`EPIC: 说的房间主链路打通`

**建议 labels**

`meta` `epic` `backend` `ios` `core-loop`

**建议正文**

```md
## 目标

打通“创建 session -> WSS 连接 -> 一轮对话 -> 会话结束”的最小可运行链路。

## 范围

- backend `app-server` session 建立
- backend `voice-gateway` WSS 主链路
- iOS `SocketTransport`
- iOS `APIClient`
- iOS 房间状态接线与基础 UI 壳

## 验收标准

- 客户端可完成一次 session 创建与握手
- 客户端与网关可完成至少一轮消息收发
- 会话结束后服务端能记录最小 session 数据

## 不包含

- 完整回顾页
- review 异步消费
- corpus / drill / 每日一读

## 下挂子任务

- backend: B2, B3, B4
- ios: I1, I2, I3, I4, I5

## 依赖

- 依赖 Epic 1 完成或至少冻结主协议边界

## 上游依据

- `31_FluentWork后端技术方案文档.md`
- `32_FluentWork-iOS App端技术设计文档.md`
- `44_FluentWork第一波跨仓任务结构与Skills使用边界.md`
```

---

## Epic 3：review 异步闭环接通

**建议标题**

`EPIC: review 异步闭环接通`

**建议 labels**

`meta` `epic` `backend` `ios` `async`

**建议正文**

```md
## 目标

把 `session.end` 后的异步任务、review 查询接口和 iOS 轮询消费打通，形成最小回顾闭环。

## 范围

- backend `ai-worker` 骨架
- `session.finished` 事件与幂等键
- `GET /sessions/:id/review`
- iOS review 轮询与骨架态
- 失败重试与最小错误态

## 验收标准

- 会话结束能触发异步任务
- review 未完成时客户端可轮询
- review 完成后客户端能拿到最小回顾数据

## 不包含

- 完整回顾页视觉打磨
- 发音评测
- APNs 静默推送增强

## 下挂子任务

- backend: B5, B6
- ios: I6

## 依赖

- 依赖 Epic 2 打通主链路

## 上游依据

- `31_FluentWork后端技术方案文档.md`
- `32_FluentWork-iOS App端技术设计文档.md`
- `44_FluentWork第一波跨仓任务结构与Skills使用边界.md`
```

---

## 二、后续新增 Epic 的规则

第一波之后若继续新增 Epic，统一遵守下面规则：

1. 只在 `meta` 建跨仓 Epic，不把 repo 内碎任务继续堆进 `meta`
2. 一个 Epic 必须能回答：
   - 目标是什么
   - 不包含什么
   - 依赖谁
   - 下挂哪些 backend / ios issues
3. Epic 数量保持克制，优先按“闭环”而不是按“模块名”来拆
4. 只有存在跨仓依赖、跨仓验收或跨仓阻塞时，才值得新建 Epic

推荐的后续 Epic 候选：

1. `EPIC: corpus 最小入库闭环`
2. `EPIC: drill 闪测最小可用闭环`
3. `EPIC: 每日一读生成与消费闭环`
4. `EPIC: 提审与发布门禁闭环`

---

## 三、最终口径

`meta` 仓的 Epic 不是实现任务列表，而是：

1. 第一波跨仓开发的导航层
2. backend / ios issue 的汇聚点
3. 依赖顺序和范围控制的真源
