# FluentWork backend 第一波 Issue 草案

**版本**：V1.0
**日期**：2026-08
**定位**：提供可直接录入 `fluentwork-backend` GitHub Issues 的第一波后端实现草案
**上游依据**：`45_FluentWork-meta第一波Epic草案.md`、`31_FluentWork后端技术方案文档.md`、`44_FluentWork第一波跨仓任务结构与Skills使用边界.md`
**变更说明**：将第一波 backend tickets 转写为可直接粘贴到 GitHub 的 issue 草案

---

## 一、使用说明

本文件中的 issue 建议建立在 `fluentwork-backend`，并统一：

1. 回链到对应的 `meta` Epic
2. 标出上游文档
3. 标出验收标准
4. 控制在单个 PR 可完成的范围

推荐 labels：

- `backend`
- `issue`
- `spec`
- `core-loop`
- `async`

---

## B1：add guest auth and merge endpoints

**建议标题**

`B1: add guest auth and merge endpoints`

**建议 labels**

`backend` `issue` `auth` `core-loop`

**建议正文**

```md
## Goal

为第一波主链路提供游客身份与数据归并能力，确保演示主路径不被注册流程阻断。

## Scope

- `POST /auth/guest`
- `POST /account/merge`
- device_id 幂等处理
- 游客身份下的最小 token / account 基线

## Acceptance Criteria

- 可以签发游客身份
- `merge` 调用幂等
- session 等业务数据可挂在游客身份下
- 错误模型统一为 `{code, message, request_id}`

## Out of Scope

- 正式邮箱 / 手机验证码登录的完整实现
- billing 权益逻辑

## Upstream Epic

- `EPIC: 说的房间契约冻结`

## References

- `31_FluentWork后端技术方案文档.md`
- `44_FluentWork第一波跨仓任务结构与Skills使用边界.md`
```

---

## B2：add session creation endpoint

**建议标题**

`B2: add session creation endpoint`

**建议 labels**

`backend` `issue` `session` `core-loop`

**建议正文**

```md
## Goal

提供第一波说的房间主链路所需的 session 创建接口。

## Scope

- `POST /sessions`
- 创建 session 最小记录
- 返回 `wss_url + ticket + session_id`
- ticket 时效与校验信息

## Acceptance Criteria

- 客户端可通过接口成功创建 session
- 返回值包含 `session_id`、`ticket`、`wss_url`
- ticket 有明确有效期
- 接口错误模型统一

## Dependencies

- B1

## Upstream Epic

- `EPIC: 说的房间契约冻结`
- `EPIC: 说的房间主链路打通`
```

---

## B3：scaffold voice-gateway WSS protocol

**建议标题**

`B3: scaffold voice-gateway WSS protocol`

**建议 labels**

`backend` `issue` `gateway` `wss` `core-loop`

**建议正文**

```md
## Goal

搭建第一波 voice-gateway WSS 协议骨架，支持最小会话握手与消息交互。

## Scope

- WSS 握手
- ticket 校验
- 控制帧 schema
- 音频帧 schema
- ping / reconnect 基础支持

## Acceptance Criteria

- 客户端能完成最小建连
- schema 与文档一致
- 至少支持一轮最小消息交互
- 非法 ticket 会被拒绝

## Dependencies

- B2

## Upstream Epic

- `EPIC: 说的房间契约冻结`
- `EPIC: 说的房间主链路打通`
```

---

## B4：persist session end data

**建议标题**

`B4: persist session end data`

**建议 labels**

`backend` `issue` `session` `persistence`

**建议正文**

```md
## Goal

在会话结束时完成 session 与最小 utterance 数据持久化，为 review 异步任务提供输入。

## Scope

- `session.end` 处理
- session 状态更新
- utterance / session 最小记录
- 重复结束事件幂等保护

## Acceptance Criteria

- 会话结束后数据能落库
- 重复结束事件不造成脏写
- review 任务可读取必要上下文

## Dependencies

- B3

## Upstream Epic

- `EPIC: 说的房间主链路打通`
```

---

## B5：scaffold review worker pipeline

**建议标题**

`B5: scaffold review worker pipeline`

**建议 labels**

`backend` `issue` `worker` `async`

**建议正文**

```md
## Goal

为 review 异步闭环建立最小 worker 管线。

## Scope

- `session.finished` 事件
- Redis Stream / worker 消费骨架
- 幂等键 `session_id + task_type`
- 一次失败重试路径

## Acceptance Criteria

- 会话结束能投递任务
- worker 能消费并更新最小状态
- 失败后最多重试一次

## Dependencies

- B4

## Upstream Epic

- `EPIC: review 异步闭环接通`
```

---

## B6：add review polling endpoint

**建议标题**

`B6: add review polling endpoint`

**建议 labels**

`backend` `issue` `review` `async`

**建议正文**

```md
## Goal

提供 iOS 轮询 review 状态所需的查询接口。

## Scope

- `GET /sessions/:id/review`
- review 未就绪 / 已就绪 / 失败三态
- 最小返回模型

## Acceptance Criteria

- iOS 可稳定轮询
- 三态口径明确
- 错误码口径明确

## Dependencies

- B5

## Upstream Epic

- `EPIC: review 异步闭环接通`
```

---

## B7：add degraded text messages endpoint

**建议标题**

`B7: add degraded text messages endpoint`

**建议 labels**

`backend` `issue` `fallback` `session`

**建议正文**

```md
## Goal

提供文本降级链路所需的最小接口占位，避免 iOS 接线时没有服务端落点。

## Scope

- `POST /sessions/:id/messages`
- 语音可用时返回 409
- 文本降级时提供最小占位响应

## Acceptance Criteria

- 降级口径存在
- iOS 可先按契约接线
- 不阻塞第一波主链路

## Dependencies

- Epic 1 契约冻结

## Upstream Epic

- `EPIC: 说的房间契约冻结`
```

---

## 二、后续新增 backend issue 的规则

第一波之后继续新增 backend issue 时，统一遵守：

1. 一个 issue 尽量对应一个 PR
2. issue 名称继续使用 `B<number>` 前缀
3. 只有确实跨 PR 才升级为 backend 内部 mini-epic
4. 优先围绕主链路闭环拆，不按目录平均切碎

推荐的后续编号区间：

1. `B8-B12`：corpus 最小入库闭环
2. `B13-B16`：drill 闪测主链路
3. `B17-B20`：每日一读批处理与消费
4. `B21+`：发布、回滚、观测增强

---

## 三、最终口径

`fluentwork-backend` 的第一波 issues 应该聚焦：

1. guest / session / gateway / review 这四个核心面
2. 先让最小闭环能跑，再补扩展模块
3. 所有 issue 都必须回链到 `meta` Epic，而不是各自漂浮
