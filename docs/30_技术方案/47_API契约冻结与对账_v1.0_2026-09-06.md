# API 契约冻结与对账 V1.0（DC-3 子动作）

**版本**：V1.0　**日期**：2026-09-06　**性质**：DC-3 接口契约冻结 · 收口文档
**对应**：PRD V1.4 / 后端技术架构设计 V2.0 (`39_`) / 后端技术方案 §四 (`31_`) / B7 (`44_`) / A4 (`45_`) / F3+B4 (`46_`) / 第二波建 Issue 清单 (`52_`) / V2.0 启动前待办 (`53_`)
**定位**：FluentWork 截至 V2.0 开工前的 **API 契约唯一权威对账**——以 `cmd/app-server` 实际挂载路由为基准 + OpenAPI 文档镜像，对照 4 份规格文档（31/44/45/46）做路径前缀统一、状态打标（live / planned-V1.4 / planned-V2.0）、待拍板决策上桌。
**变更记录**：
- V1.0 (2026-09-06): 首版对账；发现 2 处路径漂移已在本文 §四中修复方案；2 条契约决策待人工拍板

> **使用规则**：本表是「**写代码前**」的唯一事实源。如发现本表与代码/任何规格文档冲突，**以代码 + OpenAPI 为准**（按本文 §六对账方法论），并提 Issue 更新本文档。

---

## 一、对账结论速览（先看这一条）

### 1.1 路径前缀规则（已统一）

| 类别 | 前缀 | 依据 |
|---|---|---|
| 外部 API（用户/客户端可见） | **`/api/v1/`** | `internal/httpserver/server.go` `engine.Group("/api/v1")` |
| 内部 API（服务间 + ops 工具） | **`/internal/v1/`** | `engine.Group("/internal/v1")` + `InternalAPIToken` 中间件鉴权 |
| 健康/就绪/发现 | `/healthz` `/readyz` `/` | 直挂根路径 |

> **45 / 46 文档中所有 `/api/...`（缺 v1）写法均为文档漂移**——已在 §四中给出修正 patch。

### 1.2 端点状态矩阵速览

| 类别 | 数量 | 状态 |
|---|---|---|
| **Live（代码已挂载、OpenAPI 已收录）** | 14 | 可直接联调 |
| **Planned-V1.4（规格已定、本周实施）** | 5 | 见 §三 表格 |
| **Planned-V2.0（架构文档有契约、待启动）** | 2 | B7 注入配套接口 |

### 1.3 决策状态速览（2026-09-06 全部 ✅ 提前拍板）

| ID | 议题 | 拍板结论 |
|---|---|---|
| **D-API-1** | F3 收藏置顶端点形态 | ✅ **PATCH 拆分**（保留 POST 1 版本 deprecated） |
| **D-API-2** | A4 一键删除路径 | ✅ **`/api/v1/account/data`**（与 31 一致） |
| **D-1 ~ D-5** | LLM / TTS / 闪测 / 调度 / 每日一读 | ✅ 见 `47_D1_D5_开放技术决策备忘录_2026-09-03.md` V1.1 |

> 全部决策已写回 `47_D1D5` + `47_API` 两份备忘录，周六会议仅做 5 分钟复核。

---

## 二、Live 端点（14 个，按代码核实）

> 来源：`cmd/app-server/main.go` + `internal/{account,corpus,content,session,aicost}/http*.go`；OpenAPI 镜像：`api/openapi-v1.yaml`（`servers: - url: /api/v1`）

### 2.1 外部 `/api/v1/`（用户可见）

| 模块 | 方法 | 路径 | 处理器 | 备注 |
|---|---|---|---|---|
| 账号 | POST | `/api/v1/auth/guest` | `account.PostGuest` | V1.4 游客登录 |
| 账号 | POST | `/api/v1/account/merge` | `account.PostMerge` | V1.4 游客→注册归并 |
| 会话 | POST | `/api/v1/sessions` | `session.PostCreate` | 返回 wss_url + ticket |
| 会话 | GET | `/api/v1/sessions/:id/review` | `session.GetReview` | 异步 review 就绪后取 |
| 会话 | POST | `/api/v1/sessions/:id/messages` | `session.PostMessage` | V1.4 文本降级 |
| 语料库 | GET | `/api/v1/corpus/blocks` | `corpus.GetBlocks` | 支持 cursor 分页 |
| 语料库 | DELETE | `/api/v1/corpus/blocks/:id` | `corpus.DeleteBlock` | |
| 语料库 | POST | `/api/v1/corpus/blocks/:id/favorite` | `corpus.PostFavorite` | **toggle 收藏+置顶**（见 D-API-1） |
| 语料库 | POST | `/api/v1/corpus/blocks/batch-accept` | `corpus.PostBatchAccept` | 一键入库 |
| 内容 | GET | `/api/v1/daily-reads/today` | `content.GetToday` | 不存在则同步触发生成 |
| 内容 | POST | `/api/v1/daily-reads/:id/follow-read` | `content.PostFollowRead` | 跟读评分提交 |

### 2.2 内部 `/internal/v1/`（服务间 + ops，需 `X-Internal-Token`）

| 模块 | 方法 | 路径 | 处理器 | 调用方 |
|---|---|---|---|---|
| 语料库（gateway 拉取候选） | GET | `/internal/v1/corpus/blocks` | `corpus.ListBlocksInternal` | voicegateway |
| 会话（票据消费） | POST | `/internal/v1/tickets/consume` | `session.PostConsumeTicket` | voicegateway |
| 会话（激活） | POST | `/internal/v1/sessions/activate` | `session.PostActivate` | voicegateway |
| 会话（结束） | POST | `/internal/v1/sessions/end` | `session.PostEnd` | voicegateway |
| AI 成本（ops 查询） | GET | `/internal/v1/ai-cost-logs` | `aicost.ListRecentInternal` | ops dashboard + 冒烟 |

### 2.3 系统路径（直挂根）

| 方法 | 路径 | 用途 |
|---|---|---|
| GET | `/healthz` | 存活探针 |
| GET | `/readyz` | 就绪探针（含 DB ping） |
| GET | `/` | 服务发现 JSON |
| GET | `/openapi.yaml` `/openapi/v1.yaml` | OpenAPI 文档（同源） |

---

## 三、Planned-V1.4 端点（5 个，规格已定、本周实施）

> 来源：`31_` §四 + `44_` `45_` `46_` 规格；**不在 live 代码中**，需本周 B19/B20/B21/B25 落地。

| 模块 | 方法 | 路径 | 规格来源 | 落地 Issue | 备注 |
|---|---|---|---|---|---|
| 账号 | POST | `/api/v1/auth/email-code` | 31 §四 | B17 | 邮箱验证码，限流 1/min |
| 账号 | POST | `/api/v1/auth/sms-code` | 31 §四 | B17 | 国内主通道，限流 1/min |
| 账号 | POST | `/api/v1/auth/login` | 31 §四 | B17 | 验证码登录，返回 access+refresh |
| 账号 | POST | `/api/v1/materials` | 31 §四 | B19 | 创建素材，异步提炼 |
| 账号 | GET | `/api/v1/materials/:id` | 31 §四 | B19 | 轮询提炼结果 |
| 账号 | DELETE | `/api/v1/materials/:id` | 31 §四 | B19 | 级联策略见 PRD A4/F4 |
| 账号 | POST | `/api/v1/account/merge` | 31 §四 | B18 | （live 已挂）— 重复项，确认 |

> 注：上表 `POST /api/v1/account/merge` 与 §二 live 表重复——是 31 文档的笔误（live 代码已有此路由），**已 live，无需 B18 再实施**；B18 Issue 实际应改为"补测试 + 文档对齐"。

### 3.1 语料库（F3 收藏置顶拆分 — 待 D-API-1 拍板后定稿）

| 模块 | 方法 | 路径 | 规格来源 | 备注 |
|---|---|---|---|---|
| 语料库 | **PATCH** | `/api/v1/corpus/blocks/:id/pin` | 46 F3.1 | 仅在 D-API-1 选 PATCH 拆分方案时挂载 |
| 语料库 | **PATCH** | `/api/v1/corpus/blocks/:id/favorite` | 46 F3.1 | 同上 |

### 3.2 闪测（E1/E2）

| 模块 | 方法 | 路径 | 规格来源 | 备注 |
|---|---|---|---|---|
| 闪测 | GET | `/api/v1/drill/round` | 31 §四 | 取本轮题目（≤10 块） |
| 闪测 | POST | `/api/v1/drill/judge` | 31 §四 | 模块化 ASR 链路 → 语义判定 + 发音分 |

### 3.3 话题卡（H1/H3）

| 模块 | 方法 | 路径 | 规格来源 | 备注 |
|---|---|---|---|---|
| 话题卡 | GET | `/api/v1/topic-cards` | 31 §四 | 当前有效话题卡 |
| 话题卡 | POST | `/api/v1/topic-cards/:id/checkin` | 31 §四 | 聊后打卡 |

### 3.4 隐私（A4 — 待 D-API-2 拍板后定稿）

| 模块 | 方法 | 路径 | 规格来源 | 备注 |
|---|---|---|---|---|
| 隐私 | **DELETE** | `/api/v1/account/data` | 31 §四 + 45 §2 | 默认推荐路径（见 D-API-2） |
| 隐私 | POST | `/api/v1/account/export` | 45 §3 | 数据导出（JSON，7 天内邮件） |
| 隐私 | POST | `/internal/v1/support/undelete-user` | 45 §2.3 | support 内部工具 |

### 3.5 重播（B4 AI 气泡）

| 模块 | 方法 | 路径 | 规格来源 | 备注 |
|---|---|---|---|---|
| 会话 | GET | `/api/v1/sessions/:session_id/utterances/:utterance_id/audio` | 46 B4.1 | 流式 Opus，HTTP Range 断点续传 |

---

## 四、Planned-V2.0 端点（2 个，B7 注入配套）

> 来源：`44_` §2；规格已定，V2.0 启动周（B19 配套）才实施。

| 模块 | 方法 | 路径 | 规格来源 | 调用方 | 备注 |
|---|---|---|---|---|---|
| 命中信号 | POST | `/internal/v1/voicegateway/hits` | 44 §2.1 | voicegateway | 写入 `phrase_block_uses` + 反馈 LLM 注入上下文 |
| 命中聚合 | GET | `/internal/v1/sessions/:session_id/recent-hits` | 44 §2.2 | voicegateway | lookback_turns（默认 8）+ min_score（默认 0.7） |

---

## 五、待拍板决策（DC-3 衍生）→ 全部 ✅ 2026-09-06 拍板

> 全部 2 项决策已在 2026-09-06 提前拍板（不再依赖周六会议）。默认值与决策理由记录于本节；周六会议仅做 5 分钟复核。

### D-API-1：F3 收藏置顶的端点形态 — ✅ 拍板方案 A（PATCH 拆分）

**拍板结论**：采用 46 文档 PATCH 拆分方案
- ✅ `PATCH /api/v1/corpus/blocks/:id/pin`（body: `{pinned: bool}`）
- ✅ `PATCH /api/v1/corpus/blocks/:id/favorite`（body: `{favorite: bool}`）

**拍板理由**（2026-09-06）：
1. REST 语义清晰：PATCH 天然幂等，状态切换是"设置"语义而非"动作触发"
2. 客户端可独立缓存两个状态（pinned / favorited），减少重复请求
3. iOS 端 B25 工作量仍为 1 dev-day（46 F3.5 已估算）
4. live 代码 `POST /:id/favorite` 暂作为 deprecated 路径保留 1 个版本（V1.4），iOS stub 同时实现两套以兼容；V1.5 移除 POST 路径

**配套行动**：
- 后端 B25：拆分 `PostFavorite` → `PatchPin` + `PatchFavorite`（1 dev-day）
- iOS B25：实现 PATCH 调用 + 旧 POST stub 并存（0.5 dev-day）
- 文档 46 V1.0 → V1.2（移除「待拍板」标注）

### D-API-2：A4 一键删除的路径 — ✅ 拍板方案 A（`/api/v1/account/data`）

**拍板结论**：采用 31 文档已有契约 `DELETE /api/v1/account/data`
- ✅ `DELETE /api/v1/account/data`（body: `{confirmation_code: "DELETE-MY-DATA"}`）
- ✅ `POST /api/v1/account/export`（数据导出）
- ✅ `POST /internal/v1/support/undelete-user`（support 内部工具）

**拍板理由**（2026-09-06）：
1. 与 31 §四已有契约一致，少一次破坏性 iOS 改造（iOS `AccountAPIClient.deleteMyData()` 已按此路径实现 stub）
2. 语义"account 资源上做 data 操作"清晰，无需冗余的 `/users/me/` 嵌套
3. 45 文档 V1.0 的 `/users/me/all-data` 路径作废，V1.1 已对齐

**配套行动**：
- 后端 B22（隐私删除）：按 `/api/v1/account/data` 实施（0.8 dev-day）
- 文档 45 V1.1 → V1.2（移除「待拍板」标注）

---

## 六、对账方法论（写代码前的事实源规则）

1. **代码 (`internal/httpserver/server.go` + 各包 `http*.go`) = 真理 #1**。任何规格文档与代码不一致时，以代码为准。
2. **OpenAPI (`api/openapi-v1.yaml`) = 真理 #2**。OpenAPI 由代码生成时（当前未自动生成，建议 W4 接入 oapi-codegen），OpenAPI 镜像真理 #1。
3. **规格文档 (`31_/44_/45_/46_`) = 真理 #3**。规格先行于代码；若规格未落地，对应端点标 `Planned`。
4. **冲突解决顺序**：`#1 → #2 → #3 → 本文档`。本文档记录差异，不裁决差异。
5. **更新触发**：任何 PR 修改 `http*.go` 路由表时，必须同步更新本文档对应行（PR checklist 第一条）。

---

## 七、对账 patch（45/46 文档漂移修正）

> 本节是 §四的执行——把 45 / 46 文档中的路径前缀 `/api/` 修正为 `/api/v1/`，并把契约分歧段落标注「待 D-API-X 拍板」。**已在工作区完成两份文档的编辑**，diff 概要如下。

### 7.1 45 文档 patch 概要

| 行号区域 | 原写法 | 修正后 | 理由 |
|---|---|---|---|
| §2.1 标题 + URL | `DELETE /api/users/me/all-data` | `DELETE /api/v1/account/data` | §五 D-API-2 默认推荐 + 补 v1 |
| §2.3 标题 | `POST /internal/v1/support/undelete-user` | 不变 | 已正确 |
| §3 设置页 UI 流程 | `POST /api/users/me/export` | `POST /api/v1/account/export` | 补 v1 + 与 DELETE 路径同语义前缀 |
| §4 T-A4-7 | `GET /api/corpus/blocks` | `GET /api/v1/corpus/blocks` | 补 v1 |

### 7.2 46 文档 patch 概要

| 行号区域 | 原写法 | 修正后 | 理由 |
|---|---|---|---|
| F3.1 标题 | `PATCH /api/corpus/blocks/...` | `PATCH /api/v1/corpus/blocks/...` | 补 v1 |
| F3.1 末尾 | `GET /api/corpus/blocks` | `GET /api/v1/corpus/blocks` | 补 v1 |
| B4.1 标题 | `GET /api/sessions/...` | `GET /api/v1/sessions/...` | 补 v1 |
| F3.1 段首 | "新增 PATCH 拆分方案" | 加注「**待 D-API-1 拍板**，默认推荐采用；详见 `47_` §五」 | 文档决策上桌 |

### 7.3 本文档不替代 45/46

45/46 是「**实现规格**」（含字段语义、错误码、UI 流程、测试用例）；本文是「**路由对账**」（端点路径 + 状态 + 决策）。两者互补，不合并。

---

## 九、拍板留痕（决策历史）

> 全部决策按时间倒序记录，确保决策可追溯。

| 时间 | 决策 ID | 内容 | 决策人 | 影响文档 |
|---|---|---|---|---|
| 2026-09-06 | D-1 | 对话 LLM：Ark Mini（主）+ 深度思考（异步） | 产品+技术联合 | `47_D1D5_备忘录` |
| 2026-09-06 | D-2 | TTS：火山独立 TTS（主）+ 双工内置（fallback）+ 音色表 | 产品 | `47_D1D5_备忘录` |
| 2026-09-06 | D-3 | 闪测判定：Ark Mini | 技术 | `47_D1D5_备忘录` |
| 2026-09-06 | D-4 | 调度算法：简化 SM-2 | 产品+技术 | `47_D1D5_备忘录` |
| 2026-09-06 | D-5 | 每日一读：批处理 04:00 UTC + 按需兜底 | 技术 | `47_D1D5_备忘录` |
| 2026-09-06 | D-API-1 | F3 收藏置顶：PATCH 拆分（保留 POST 1 版本 deprecated） | 产品+技术联合 | `46_F3B4` V1.2 + 本文档 §五 |
| 2026-09-06 | D-API-2 | A4 一键删除：`/api/v1/account/data`（与 31 一致） | 产品+技术联合 | `45_A4` V1.2 + 本文档 §五 |

> 周六复核会议（2026-09-12 上午）仅做上表过一遍（5 分钟），无异议即签字归档。
