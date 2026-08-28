# FluentWork 第二波跨仓任务结构与 Issue 草案

**版本**：V1.0
**日期**：2026-08
**定位**：定义 FluentWork 第二波跨仓任务如何在 `meta`、`backend`、`ios` 三仓落位，沿用第一波的三层结构（`meta` Epic → 代码仓 issue → 执行清单）
**上游依据**：`20_FluentWork产品需求文档PRD.md`（W5-W6 排期与模块 C1-C3 / D1-D2 / B7 / F1-F2 / C5）、`31_FluentWork后端技术方案文档.md`、`32_FluentWork-iOS App端技术设计文档.md`、`44_FluentWork第一波跨仓任务结构与Skills使用边界.md`
**变更说明**：第一波主链路（session → WSS → 一轮对话 → session.end → review 骨架）已完成，本文定义第二波的任务边界与 issue 草案
**状态**：执行文档，可直接作为第二波任务建立的依据

---

## 一、结论

第二波不再扩张主链路骨架，而是把第一波打通的链路"做实、做满"：

> **第二波的目标是把"说 → 回顾 → 入库"这条完整流水线打通并对齐北极星口径，对应 PRD 的 W5-W6 排期。**

任务结构沿用第一波口径：

1. `meta` 放 **跨仓 Epic 与依赖关系**
2. `fluentwork-backend` 放 **后端实现 tickets（编号从 `B8` 延续）**
3. `fluentwork-ios` 放 **iOS 实现 tickets（编号从 `I7` 延续）**

---

## 二、第二波的目标边界

### 2.1 进入第二波的内容（PRD W5-W6）

1. `C1-C3` 回顾页完整版：转录话轮、三层评价、双栏对照
2. `D1-D2` 话术块炼化：自动提炼 3-5 个话术块 + 可编辑 / 丢弃 / 入库
3. `B7` 即时反馈：对话中命中已入库话术块 → 轻量徽章 + 下轮自然确认
4. `F1-F2` 语料库基础版：双维度筛选 + 搜索 + 状态灯
5. `C5` 每日一读：每日生成 + AI 朗读 + 跟读（不出分）

出口标准（对齐 PRD 第十章）：

1. W5 出口：说 → 读 → 入库打通，命中反馈可见
2. W6 出口：话术块可浏览搜索；每日一读按时生成

### 2.2 第二波不进入的内容

1. `E1-E5` 闪测（E1-E3 排期 W7）
2. `H1-H4` 话题建议（W7）
3. `I1-I4` 发音评测（V1.1）
4. `billing` 商业化实现（V1.1，表结构与接口预留按既有口径）
5. `C4` 创建练习弹层与 `A1-A2` 素材模块：属于 W3-W4 排期但第一波未承接，记为**遗留项**，不在第二波扩张，后续单独定波次

---

## 三、第二波 Epic 建议

在 `meta` 中建立 5 个 Epic。

## Epic 4：回顾与炼化闭环

目标：

把第一波的 review 骨架做实——`session.end` 后异步生成真实评价与炼化结果，iOS 完整回顾页消费并支持话术块入库。

包含：

1. backend `ai-worker` 评价 + 炼化真实现（合并一次旗舰模型调用）
2. `GET /sessions/:id/review` 扩展为全量返回模型
3. iOS 完整回顾页（渐进加载 + 双栏对照）
4. iOS 炼化卡交互与入库动作

验收标准：

1. 会话结束后评价与炼化结果在 P90 ≤ 15s 内就绪，超时告警
2. 回顾页可展示三层评价与双栏对照，每条评价引用原句
3. 炼化卡可编辑、丢弃、单独入库与一键入库
4. 任务幂等键 `session_id + task_type`，失败重试 1 次不产生脏写

## Epic 5：语料库基础版

目标：

让入库的话术块有地方安放、查看与追踪，形成"练 → 沉淀"的资产面。

包含：

1. `phrase_blocks` 表与 migration
2. backend `corpus` 模块：列表 / 编辑 / 删除 / 收藏 / 批量入库
3. iOS 语料库页：本地优先缓存 + 双维度筛选 + 搜索 + 状态灯

验收标准：

1. `GET /corpus/blocks` 支持场景 / 功能双维度筛选与关键词搜索，cursor 分页
2. 入库后话术块带调度字段（`next_due_at` / `state`），状态灯口径为 新入库（灰）/ 训练中（黄）/ 已自动化（绿）
3. iOS 弱网场景可浏览本地缓存
4. 锚点字段 `anchor_user_said` 按 3.2 要点 4 口径明文落库参与检索

## Epic 6：每日一读

目标：

按 `C5` 建立每日一读生成管线与客户端消费页，跟读只录音不出分。

包含：

1. `daily_reads` 表与 migration（`uk_user_date` 唯一键防重）
2. 凌晨 02:00 批处理骨架 + 新用户当天首次打开的同步兜底生成
3. 语料不足时预置内容兜底
4. iOS 每日一读页：今日文章 + AI 朗读 + 跟读录音 + 后台播放

验收标准：

1. `GET /daily-reads/today` 在内容未就绪时行为明确（轮询就绪）
2. 批处理对活跃用户（近 7 天有练习）生成，重复生成被唯一键拦截
3. 跟读评分接口占位存在但恒不出分（随发音评测 V1.1 启用）
4. iOS 后台播放配置验证通过

## Epic 7：即时反馈命中检测（B7）

目标：

在对话中实时检测用户命中已入库话术块，给出轻量正反馈并累计实战使用次数。

包含：

1. `voice-gateway` 在 `user.speech.end` 时旁路触发命中检测
2. `app-server` 内部命中检测接口（800ms 超时，失败跳过）
3. `badge` 控制帧下发 + 下一话轮 system 注入
4. iOS 徽章展示（展示类，不进状态机）

验收标准：

1. 命中检测不阻塞语音主链路，超时或失败直接跳过
2. 命中即下发轻量徽章（"地道表达 +1"），`real_use_count` 累计
3. 新帧纳入 `wss-control-frames-v1.json` 契约与契约测试
4. iOS 徽章为展示类 Action，不进入 SpeechSession reducer

## Epic 8：语音链路火山做实（Phase 0 并行轨）

目标：

偿还第一波"双端 stub"欠账：网关接真实火山语音链路，iOS 替换占位音频引擎，并完成 B7 注入能力 POC，为 `B12` 定路径、为压测 Go/No-Go 阀门提供结论。

包含：

1. `voice-gateway` 接入火山端到端语音（替换 stub `ai.text.delta`）
2. 三级降级切换逻辑（端到端 → 模块化编排 → 纯文本）
3. iOS 真实 AudioEngine 替换 `PlaceholderAudioEngine`（采集 / Opus 编码 / 播放 / 打断）
4. 端到端文本注入能力 POC（V8 有效注入窗口测量，档位结论回写 `meta`）

验收标准：

1. 客户端与网关可完成真实语音对话，首响 P90 ≤ 1.5s（Go/No-Go 指标）
2. 接入第一天音频回环自测通过，音频参数与火山要求严格对齐
3. 三级降级可切换，非核心链路故障不影响主链路
4. POC 输出有效注入窗口长度，确定 B7 档位（①/②/③），`B12` 技术路径冻结
5. 火山凭证与 API Key 全程不出服务端，客户端不持有供应商凭证
6. 打断（barge-in）本地立即停播，目标 200ms 内声音消失

前置提醒（非工程，商务 / 采购面）：

1. 免费额度算账 → 供应商确认记录（W0 遗留项）
2. 企业协议"API 数据不用于训练"条款确认（提审材料引用）
3. 并发配额写进合同，与火山商务确认压测窗口

---

## 四、第二波 backend tickets 草案

下面这些 tickets 建议建立在 `fluentwork-backend`，编号延续第一波。

## B8. ai-worker 评价 + 炼化真实现

范围：

1. 替换第一波 review worker stub：评价与炼化合并一次旗舰模型调用
2. JSON Schema 校验，失败重试 1 次
3. 幂等键 `session_id + task_type`
4. 每次 AI 调用落 `ai_cost_logs`
5. 全部完成后 `session.status = reviewed`

依赖：

- 第一波 `B5`（worker 骨架）

验收重点：

1. 评价生成 P90 ≤ 15s，超时告警
2. 重复消费直接跳过
3. 失败后会话仍可回看转录，评价区留给前端"重试"态

## B9. review 接口全量返回模型

范围：

1. `GET /sessions/:id/review` 扩展：转录话轮 + 三层评价 + 双栏对照 + 炼化卡
2. 保持第一波三态（未就绪 / 已就绪 / 失败）契约不回退

依赖：

- `B8`

验收重点：

1. 返回模型与 `openapi-v1.yaml` 同步
2. 三层评价每条引用原句
3. 炼化卡包含 意图 / 英文块 / 对比锚点 三元组

## B10. corpus 模块与 phrase_blocks

范围：

1. migration：`phrase_blocks` 表（含 `is_favorite` / `pinned_at` / 调度字段 / FULLTEXT）
2. `GET /corpus/blocks`（双维度筛选 + 关键词搜索 + cursor 分页）
3. `PUT /corpus/blocks/:id`、`DELETE /corpus/blocks/:id`
4. `POST /corpus/blocks/:id/favorite`
5. `POST /corpus/blocks/batch-accept`（供炼化卡一键入库）

依赖：

- `B9`（炼化卡数据是入库入口）

验收重点：

1. 批量入库幂等，入库即进入调度队列字段
2. 状态迁移（new / training / automated）单测覆盖
3. 软删除口径与 3.2 要点 1 一致

## B11. content 模块与每日一读

范围：

1. migration：`daily_reads` 表（`uk_user_date` 唯一键）
2. `GET /daily-reads/today`：不存在则同步触发生成，轮询就绪
3. 凌晨 02:00 批处理骨架（活跃用户口径：近 7 天有练习）
4. 语料不足时预置内容兜底
5. `POST /daily-reads/:id/follow-read` 占位（不出分）

依赖：

- `B10`（生成内容衍生自语料）

验收重点：

1. 重复生成被唯一键拦截
2. 同步兜底生成约 5-10s，前端轮询行为明确
3. 生成失败不影响已有内容（熔断原则）

## B12. B7 命中检测链路

范围：

1. `voice-gateway` 在 `user.speech.end` 旁路调用 `app-server` 内部命中检测
2. 800ms 超时，失败跳过，不阻塞音频流
3. 命中 → `badge` 控制帧下发 + 下一话轮 system 注入
4. `real_use_count` 累计
5. `wss-control-frames-v1.json` 增加 badge 帧并补契约测试

依赖：

- `B10`（命中检测需要已入库话术块）

验收重点：

1. 命中检测故障不影响"说 → 转录落库"主链路
2. 契约测试覆盖新帧
3. 网关与单体边界不被模糊（检测执行在单体，网关只旁路触发）

## B13. voice-gateway 火山端到端接入与三级降级（Phase 0）

范围：

1. 替换 stub：`session.start` 接火山端到端实时语音真实对话
2. 三级降级切换逻辑（端到端 → 模块化编排 → 纯文本，自动 + 可人工）
3. 三段延迟埋点分开发上报（端上 / 网关转发 / 模型链路）
4. 首响 P90 ≤ 1.5s 压测验证（PRD W1-W2 出口标准）

依赖：

- 第一波 `B3`（WSS 协议骨架，已完成）
- 供应商前置（额度 / 协议 / 凭证就位）

验收重点：

1. 真实语音对话回环可跑通（坑清单 #3：接入第一天音频回环自测）
2. 降级切换不破坏会话状态机语义，纯文本降级与第一波 `B7` 文本接口口径衔接
3. API Key 与火山凭证全程不出服务端（两次握手口径不变）

## B14. 端到端注入能力 POC（Phase 0，最先开始）

范围：

1. 按《`50_FluentWork端到端注入能力验证文档`》执行 V8 有效注入窗口测量与 T9 用例
2. 输出档位结论（① 当轮确认 / ② 下轮开局确认 / ③ 仅徽章不注入）
3. 结论回写 `meta` 验证文档，冻结 `B12` 技术路径

依赖：

- 火山凭证就位（可直连火山 API，不依赖 `B13`）

验收重点：

1. 实测数据支撑三档之一，不悬而未决（W0 遗留项 2）
2. `B12` 启动前结论已回写 `meta`

顺序说明：`B14` 不依赖网关接入，是 Phase 0 中唯一可以立即开始的任务。

---

## 五、第二波 iOS tickets 草案

下面这些 tickets 建议建立在 `fluentwork-ios`，编号延续第一波。

## I7. 完整回顾页

范围：

1. 渐进加载的两层到达时序：骨架 → 转录 / 评价 → 炼化卡
2. 三层评价卡展示（每条引用原句）
3. 双栏对照与差异高亮
4. 用户话轮回听入口

依赖：

- 第一波 `I6`（review 轮询骨架）
- `B9`

验收重点：

1. 两层到达时序有单测（对齐 32 号文档 `C5` 任务单）
2. review 失败态保留"重试"入口
3. View 不直接依赖 transport / audio 底层

## I8. 炼化卡交互与入库

范围：

1. 炼化卡编辑 / 丢弃 / 单独入库
2. 一键全部入库（接 `POST /corpus/blocks/batch-accept`）
3. 入库后的最小确认反馈

依赖：

- `I7`
- `B10`

验收重点：

1. 入库动作幂等，重复点击不重复入库
2. 空卡 / 全部丢弃后的空态明确

## I9. 语料库页本地优先

范围：

1. `CorpusStore` / `SyncEngine`：本地缓存 + 增量同步
2. 场景 / 功能双维度筛选 + 关键词搜索
3. 状态灯（灰 / 黄 / 绿）

依赖：

- `B10`

验收重点：

1. 弱网本地优先场景测试通过（对齐 32 号文档 `C6` 任务单）
2. 高频列表渲染不把每帧数据塞进全局 Store
3. 收藏 / 置顶 / 删除交互留下波，本期只接线最小收藏

## I10. 每日一读页

范围：

1. 今日文章展示 + 生成未就绪时的轮询态
2. AI 朗读播放
3. 跟读模式：只录音与示范对比，不出分
4. 后台播放配置
5. 历史归档最小占位

依赖：

- `B11`

验收重点：

1. 后台播放配置验证通过（对齐 32 号文档 `C7` 任务单）
2. 跟读评分字段不消费（V1.1 启用）

## I11. B7 徽章展示接线

范围：

1. 消费 `badge` 控制帧
2. 轻量徽章展示（"地道表达 +1"），不弹窗不震动
3. 展示类 Action，不进入 SpeechSession 状态机 reducer

依赖：

- `B12`
- 第一波 `I3`（frame decoding）

验收重点：

1. 徽章展示不阻塞语音流
2. 事件表口径与 32 号文档一致（`badgeHit → .feedback(.badgeHit)` 展示类）

## I12. 真实 AudioEngine 替换占位实现（Phase 0）

范围：

1. `AVAudioEngine` 采集（16kHz 单声道 tap）+ Opus 编码（火山 SDK 优先）+ 上行
2. AI 语音播放 + 打断本地立即停播（barge-in，不等服务端确认）
3. `AVAudioSession` 配置（`.playAndRecord` + `.voiceChat`）
4. 接入第一天音频回环自测，参数与火山要求严格对齐
5. 替换 DI 中的 `PlaceholderAudioEngine`

依赖：

- 第一波 `I3`（frame 编解码，已完成）
- backend `B13`（真链路可联调）

验收重点：

1. `PlaceholderAudioEngine` 从 DI 移除（可保留为测试用实现）
2. 打断目标 200ms 内声音消失（端上指标，不依赖服务端）
3. 不修改 `SpeechSession` 状态机本体，接线继续走 Middleware
4. 说的房间高频路径（波形 / 流式文本）帧率抽查无明显劣化（32 号文档 1.4.6）

---

## 六、跨仓依赖顺序

第二波推荐按这个顺序推进：

1. `meta` 建 Epic 4、5、6、7、8
2. **Phase 0（并行轨）：`B14` 注入 POC 先行，`B13` 网关接入与 `I12` 音频引擎并行**
3. backend 建 `B8`、`B9`、`B10`
4. ios 建 `I7`、`I8`、`I9`
5. backend 建 `B11`、`B12`（`B12` 启动前 `B14` 结论必须已回写 `meta`）
6. ios 建 `I10`、`I11`

换句话说：

> **Phase 0 把语音链路做实并与 Phase A 并行，不改变原有波次的任务边界；功能面先把回顾与炼化做实，再建语料库承接入库，然后每日一读独立推进，最后接命中反馈——命中检测既依赖已入库的语料数据，也依赖注入 POC 的档位结论。**

---

## 七、必须遵守的禁区与边界

继承第一波禁区，并补充第二波特有边界：

1. 不修改人写的 `SpeechSession` 状态机本体
2. 不把每帧高频数据塞进全局 Store
3. 不模糊 `app-server` 与 `voice-gateway` 边界——命中检测的实际执行在单体，网关只旁路触发
4. 非核心链路（每日一读、命中检测）故障不能影响"说 → 转录落库"主链路
5. 评价必须引用原句，炼化必须来自用户真实卡壳点——Prompt 口径不在代码仓私下定案，冲突先回写 `meta`
6. 跟读不出分是本期硬口径，不得提前引入评分展示
7. `phrase_blocks` 锚点字段按后端方案 3.2 要点 4 口径处理，不得自行加密或脱敏
8. 火山凭证与 API Key 全程不出服务端；客户端不持有供应商凭证，不直连火山服务

---

## 八、Issue 命名与链接建议

### `meta`

- `EPIC: 回顾与炼化闭环`
- `EPIC: 语料库基础版`
- `EPIC: 每日一读`
- `EPIC: 即时反馈命中检测`
- `EPIC: 语音链路火山做实`

### `backend`

- `B8: implement review evaluation and refine in ai-worker`
- `B9: extend review endpoint to full model`
- `B10: add corpus module and phrase_blocks migration`
- `B11: add content module and daily reads`
- `B12: add B7 hit detection path`
- `B13: connect voice-gateway to Volcano realtime speech`
- `B14: run end-to-end text injection POC`

### `ios`

- `I7: add full review page`
- `I8: add refine cards accept flow`
- `I9: add corpus page with local-first store`
- `I10: add daily read page`
- `I11: add badge feedback display`
- `I12: replace placeholder audio engine with real pipeline`

链接规则（沿用第一波）：

1. backend / ios ticket 必须回链到所属 `meta` Epic
2. PR 必须回链到 repo ticket
3. `meta` Epic 不直接对应代码 PR，而是对应一组 repo tickets 的完成状态

---

## 九、最终口径

FluentWork 第二波跨仓任务结构的标准口径应统一为：

1. 第二波只做 PRD W5-W6：回顾完整版 + 炼化 + 语料库基础版 + 每日一读 + 即时反馈；另以 Phase 0 并行轨偿还语音链路火山实接欠账；
2. `meta` 只放跨仓 Epic（5 个），实现 tickets 留在代码仓并从 `B8` / `I7` 延续编号；
3. 推进顺序：Phase 0 语音做实并行 → 回顾做实 → 入库承接 → 每日一读 → 命中反馈；
4. 闪测、话题建议、发音评测、创建练习弹层不进入第二波；
5. 所有禁区与熔断原则继承第一波口径。
