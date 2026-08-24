# FluentWork iOS 第一波 Issue 草案

**版本**：V1.0
**日期**：2026-08
**定位**：提供可直接录入 `fluentwork-ios` GitHub Issues 的第一波 iOS 实现草案
**上游依据**：`45_FluentWork-meta第一波Epic草案.md`、`32_FluentWork-iOS App端技术设计文档.md`、`44_FluentWork第一波跨仓任务结构与Skills使用边界.md`
**变更说明**：将第一波 iOS tickets 转写为可直接粘贴到 GitHub 的 issue 草案

---

## 一、使用说明

本文件中的 issue 建议建立在 `fluentwork-ios`，并统一：

1. 回链到对应 `meta` Epic
2. 明确依赖哪些 backend 契约
3. 把技术文档中的禁区带进 issue
4. 保持单个 issue 对应单个可审查 PR

推荐 labels：

- `ios`
- `issue`
- `spec`
- `core-loop`
- `ui-shell`

---

## I1：add root store and DI baseline

**建议标题**

`I1: add root store and DI baseline`

**建议 labels**

`ios` `issue` `architecture` `core-loop`

**建议正文**

```md
## Goal

建立第一波 iOS 开发所需的状态管理和依赖注入基座。

## Scope

- TGReduxKit 接入
- Factory 容器
- `AppState`
- Root Store
- TestStore 基建

## Acceptance Criteria

- 全工程零 Combine
- Root Store 可运行
- reducer / middleware 测试基线存在
- 说的房间与工作台至少能被 `scope`

## Out of Scope

- 页面视觉打磨
- 具体业务 API 接线

## Upstream Epic

- `EPIC: 说的房间主链路打通`

## References

- `32_FluentWork-iOS App端技术设计文档.md`
```

---

## I2：wire SpeechSession reducer boundary

**建议标题**

`I2: wire SpeechSession reducer boundary`

**建议 labels**

`ios` `issue` `state-machine` `core-loop`

**建议正文**

```md
## Goal

把人写的 SpeechSession 状态机接口以正确边界接入根 Store，不修改状态机本体。

## Scope

- 接入人写状态机接口
- 事件到 Action 的映射
- SideEffect 协议边界
- Middleware 与 reducer 责任划分

## Acceptance Criteria

- 不修改人写状态机本体
- reducer 与 middleware 责任清晰
- 事件表与技术文档一致

## Dependencies

- I1
- 契约冻结 Epic

## Upstream Epic

- `EPIC: 说的房间契约冻结`
- `EPIC: 说的房间主链路打通`
```

---

## I3：add socket transport and frame decoding

**建议标题**

`I3: add socket transport and frame decoding`

**建议 labels**

`ios` `issue` `transport` `wss`

**建议正文**

```md
## Goal

建立说的房间所需的 SocketTransport 与帧编解码层。

## Scope

- WSS 建连
- ping / reconnect
- 控制帧与音频帧编解码
- 序列号丢帧纯函数

## Acceptance Criteria

- 能消费 backend WSS schema
- 丢帧规则有单测
- 失败态可上抛到 Store

## Dependencies

- I1
- 契约冻结 Epic

## Upstream Epic

- `EPIC: 说的房间契约冻结`
- `EPIC: 说的房间主链路打通`
```

---

## I4：add session API client

**建议标题**

`I4: add session API client`

**建议 labels**

`ios` `issue` `api` `session`

**建议正文**

```md
## Goal

接入第一波主链路所需的 session 相关 REST 接口。

## Scope

- `POST /auth/guest`
- `POST /sessions`
- `GET /sessions/:id/review`
- `POST /account/merge`
- `POST /sessions/:id/messages`

## Acceptance Criteria

- 错误归一化
- ticket / session 创建链路可跑通
- review 轮询接口已可用
- 文本降级接口已留接线点

## Dependencies

- I1
- backend B1 / B2 / B6 / B7 契约就绪

## Upstream Epic

- `EPIC: 说的房间契约冻结`
- `EPIC: 说的房间主链路打通`
```

---

## I5：scaffold speaking room screen

**建议标题**

`I5: scaffold speaking room screen`

**建议 labels**

`ios` `issue` `ui-shell` `core-loop`

**建议正文**

```md
## Goal

为说的房间主链路提供最小可运行 UI 壳。

## Scope

- 房间页面基础布局
- 状态切换
- 说话键 / 转录浮层 / 基础消息区占位
- loading / failed / retry 最小错误态

## Acceptance Criteria

- 能承载主链路状态流转
- 错误态与 loading 态可见
- View 不直接依赖底层音频或 transport

## Out of Scope

- 完整视觉精修
- 波形 / 动效细节打磨

## Dependencies

- I2
- I3
- I4

## Upstream Epic

- `EPIC: 说的房间主链路打通`
```

---

## I6：add review polling and skeleton state

**建议标题**

`I6: add review polling and skeleton state`

**建议 labels**

`ios` `issue` `review` `async`

**建议正文**

```md
## Goal

为第一波回顾闭环接入 review 轮询和最小骨架态。

## Scope

- 回顾页最小 state
- 轮询 `GET /sessions/:id/review`
- skeleton / ready / failed 三态
- 最小重试路径

## Acceptance Criteria

- review 未完成时前端行为明确
- 完成后可展示最小结果
- 重试路径存在

## Dependencies

- I4
- backend B6
- review 异步 Epic

## Upstream Epic

- `EPIC: review 异步闭环接通`
```

---

## 二、后续新增 iOS issue 的规则

第一波之后继续新增 iOS issue 时，统一遵守：

1. issue 名称继续使用 `I<number>` 前缀
2. 保持“状态管理 / transport / API / UI / sync”这五类边界，不把职责混成一个大票
3. 任何依赖 backend 契约的 issue，必须在正文里写清依赖项
4. 人写禁区相关内容不能被普通 UI / Store issue 隐式改动

推荐的后续编号区间：

1. `I7-I10`：corpus 最小浏览与同步
2. `I11-I14`：drill 闪测主链路
3. `I15-I18`：每日一读与后台播放
4. `I19+`：设置页、订阅入口、体验增强

---

## 三、Matt Pocock 风格 skills 在 iOS issue 中怎么用

在 `fluentwork-ios` 中，Matt Pocock 风格 skills 适合继续做这些事：

1. 把 `I5` 这种 UI 壳 issue 继续细成子步骤
2. 把 `I3` 里的 transport 拆成测试任务与实现任务
3. 为单个 issue 生成更细的测试清单

但不应用来：

1. 改写 `meta` Epic 边界
2. 改写技术文档中已经冻结的人写禁区
3. 在没有更新 `meta` 的前提下自行扩展跨仓范围

统一口径：

> **skills 用于继续细化 repo issue，不用于重新定义跨仓结构。**

---

## 四、最终口径

`fluentwork-ios` 的第一波 issues 应该聚焦：

1. Root Store / DI 基座
2. SpeechSession 边界接线
3. SocketTransport
4. Session APIClient
5. 说的房间 UI 壳
6. review 轮询最小闭环

先把这 6 个点站稳，再进入 corpus / drill / 每日一读。
