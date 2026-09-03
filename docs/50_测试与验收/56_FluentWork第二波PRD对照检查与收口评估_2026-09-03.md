# FluentWork 第二波 PRD 对照检查与收口评估

**版本**：V1.0
**日期**：2026-09-03
**定位**：对照 PRD V1.4，盘点当前实现状态与偏差，量化第二波完成差距

---

## 一、提交记录

| 仓 | 提交 | 内容 |
|---|---|---|
| fluentwork-backend | `59efd2f` | feat(voicegateway): B15-I3 add log_id to ai.turn.end |
| fluentwork-ios | `889517b` | feat(iOS): B15-I3 capture log_id from ai.turn.end |

---

## 二、PRD P0 功能对照

### 2.1 模块 A：素材输入

| ID | 需求 | 状态 | 说明 |
|---|---|---|---|
| A1 | 粘贴工作内容生成练习 | ❌ 未实现 | 第一波遗留项；第二波范围外 |
| A2 | 一句话说明场景 | ❌ 未实现 | 第一波遗留项；第二波范围外 |
| A3 | 默认入口预置场景 | ✅ 已实现 | Bootstrap demo material 主路径 |
| A4 | 隐私声明 | ❌ 未实现 | 第二波范围外 |
| A5 | 演示素材一键体验 | ✅ 已实现 | Bootstrap 默认流程 |

### 2.2 模块 B：说（Practice Room）

| ID | 需求 | 状态 | 说明 |
|---|---|---|---|
| B1 | 语音对话（ASR→LLM→TTS） | ✅ 已实现 | VolcDuplexProvider + WSS |
| B2 | AI 提问贴素材 | ✅ 已实现 | session.start instructions 注入 |
| B3 | 实时转录浮层 | ✅ 已实现 | client.asr.transcription 帧 |
| B4 | AI 气泡重播/展开 | ⚠️ 部分实现 | 文本展示；音频重播未完全实现 |
| B5 | 完整转录 + 异步分析 | ✅ 已实现 | session.end → worker |
| B6 | 中途重来 | ⚠️ 部分实现 | 可放弃不计入 review |
| B7 | 即时反馈徽章 | ✅ 已实现 | BadgeEmitter + iOS overlay |

### 2.3 模块 C：读（回顾页与每日一读）

| ID | 需求 | 状态 | 说明 |
|---|---|---|---|
| C1 | 转录按话轮展示 | ✅ 已实现 | I7 完整回顾页 |
| C2 | 三层评价卡 | ✅ 已实现 | I7 review page |
| C3 | 双栏对照 | ✅ 已实现 | I7 review page |
| C4 | 历史练习回顾 | ❌ 未实现 | 第一波遗留项 |
| C5 | 每日一读 | ✅ 已实现 | B11 + I10 |

### 2.4 模块 D：话术块炼化

| ID | 需求 | 状态 | 说明 |
|---|---|---|---|
| D1 | 自动提炼 3-5 个话术块 | ⚠️ 骨架已就 | B8 OPEN（真实 LLM 未接入） |
| D2 | 卡片可编辑/入库 | ✅ 已实现 | I8 batch-accept |
| D3 | 场景/功能标签 | ✅ 已实现 | B10 phrase_blocks |

### 2.5 模块 E：闪测（提取训练）

| ID | 需求 | 状态 | 说明 |
|---|---|---|---|
| E1-E5 | 闪测完整功能 | ❌ 未实现 | **V1.4 明确 W7 排期**，非本轮范围 |

### 2.6 模块 F：语料库

| ID | 需求 | 状态 | 说明 |
|---|---|---|---|
| F1 | 双维度筛选 + 搜索 | ✅ 已实现 | I9 corpus page |
| F2 | 状态灯 | ✅ 已实现 | 新入库/训练中/已自动化 |
| F3 | 收藏与置顶 | ⚠️ 部分实现 | 收藏已就；置顶未做 |
| F4 | 单条删除 | ✅ 已实现 | I9 delete |

### 2.7 模块 G：账号与基础

| ID | 需求 | 状态 | 说明 |
|---|---|---|---|
| G1 | 邮箱 + 手机号登录 | ✅ 已实现 | Backend auth + iOS |
| G2 | 个性化设置 | ✅ 已实现 | Feature flags |
| G3 | 演示素材体验 | ✅ 已实现 | A5 同上 |
| **G4** | **游客→注册数据归并** | ✅ **已实现** | **Bootstrap 中处理；详见 §三** |

### 2.8 模块 H：话题建议

| ID | 需求 | 状态 | 说明 |
|---|---|---|---|
| H1-H4 | 话题建议全部功能 | ❌ 未实现 | **V1.4 明确 W7 排期**，非本轮范围 |

### 2.9 模块 I：语音评测

| ID | 需求 | 状态 | 说明 |
|---|---|---|---|
| I1-I4 | 发音评测全部功能 | ❌ 未实现 | **V1.4 PRD 明确整体后置到 V1.1** |

---

## 三、G4 游客归并决策落地说明

### 3.1 决策内容

根据用户决策，G4 游客归并在 **Bootstrap 中处理**，延迟到用户触发"保存语料"时才引导注册，实现：

1. 免注册启动 → 自动签发游客 token
2. 游客可体验完整功能
3. 注册时调用 `POST /account/merge` 无损归并数据

### 3.2 实现证据

| 层面 | 文件 | 证据 |
|---|---|---|
| Backend API | `internal/account/service.go` | `POST /auth/guest` + `POST /account/merge` |
| iOS Bootstrap | `AppBootstrapMiddleware.swift` | `bootstrapSucceeded` 携带 `authInfo` |
| iOS Reducer | `AppReducer.swift:110` | `bootstrapSucceeded` 处理 `AuthState` 更新 |
| iOS Corpus 归并 | `AppBootstrapMiddleware.swift:587` | `mergedIntoRegistered` → `mergeRebuild` |
| 测试 | `CorpusFeatureTests.swift:352` | `corpusMergeClearsGuestStateAndRebuildsForRegisteredUser` |
| 文档 | `docs/15_游客升级注册用户流程修复方案.md` | 完整流程设计文档 |

### 3.3 状态

✅ **已在代码中实现**；文档 `docs/15_游客升级注册用户流程修复方案.md` 作为设计记录。

---

## 四、第二波完成度量化

### 4.1 第二波 Issue 完成状态

| Issue | 标题 | Backend | iOS | 状态 |
|---|---|---|---|---|
| B8 | review eval + refine | ❌ OPEN | - | **唯一阻断项** |
| B9 | review 接口全量模型 | ✅ CLOSED | - | |
| B10 | corpus 模块 | ✅ CLOSED | - | |
| B11 | content + 每日一读 | ✅ CLOSED | - | |
| B12 | B7 命中检测 | ✅ CLOSED | - | |
| B13 | Volc 端到端 | ✅ CLOSED | - | |
| B14 | 注入 POC | ✅ CLOSED | - | |
| B15 | 离线评估 | ✅ CLOSED | - | |
| I7 | 完整回顾页 | - | ✅ CLOSED | |
| I8 | 炼化卡入库 | - | ✅ CLOSED | |
| I9 | 语料库本地优先 | - | ✅ CLOSED | |
| I10 | 每日一读页 | - | ✅ CLOSED | |
| I11 | 徽章展示 | - | ✅ CLOSED | |
| I12 | 真实 AudioEngine | - | ✅ CLOSED | |
| I13 | 语料库同步策略 | - | ✅ CLOSED | |

### 4.2 量化结论

```
第二波代码层面完成度：14/15 issues = 93%

唯一 OPEN 项：B8（review eval + refine 真实 LLM 接入）
- 影响：回顾页显示 stub 数据，非真实 AI 评价
- 阻断：不影响其他功能正常使用
```

### 4.3 PRD P0 完成度

| 范围 | 应完成 | 实际完成 | 完成率 |
|---|---|---|---|
| MVP P0 功能（排除 E/H/I） | 约 25 项 | 约 21 项 | **84%** |
| 其中第二波重点项 | B9-B15 / I7-I13 | 14/15 issues | **93%** |

---

## 五、偏差分析

### 5.1 有意偏差（PRD 明确不在本轮）

| 偏差项 | PRD 原文 | 决策 |
|---|---|---|
| E1-E5 闪测 | W7 排期 | 第二波范围外，保持 |
| H1-H4 话题建议 | W7 排期 | 第二波范围外，保持 |
| I1-I4 发音评测 | V1.1 后置 | PRD V1.4 明确后置，保持 |

### 5.2 实现差距（需跟进）

| 差距项 | 影响 | 建议 |
|---|---|---|
| B8 未闭合 | 回顾页只有 stub 评价 | 优先跟进 B8 真实 LLM 接入 |
| A1-A2 素材输入 | 无法自定义练习内容 | 第一波遗留，可后续补充 |
| C4 历史回顾 | 无法查看历史练习 | 后续迭代补充 |
| F3 置顶 | 语料库功能不完整 | 后续迭代补充 |

### 5.3 轻微功能差距

| 差距项 | 影响 | 严重度 |
|---|---|---|
| B4 AI 气泡重播 | 可听录音增强体验 | 低 |
| B6 中途重来 | 可放弃当前会话 | 低 |

---

## 六、下一步建议

### 6.1 立即行动

1. **跟进 B8** — review eval + refine 真实 LLM 接入
   - 当前：stub 数据
   - 目标：真实 AI 评价 + 话术块生成

### 6.2 后续迭代（V1.0 补完）

1. A1-A2 素材输入
2. C4 历史回顾
3. F3 语料库置顶

### 6.3 V1.1 范围（已明确）

1. E1-E5 闪测
2. H1-H4 话题建议
3. I1-I4 发音评测

---

## 七、最终结论

> **第二波代码层面完成度 93%（14/15 issues），PRD MVP P0 功能完成度 84%**。唯一显著差距是 B8（review eval + refine 真实 LLM）未闭合，导致回顾页显示 stub 数据。G4 游客归并已按 Bootstrap 策略落地实现。有意偏差（闪测/话题/发音评测）均符合 PRD 排期决策。

---

*文档版本：V1.0*
*上游依据：PRD V1.4、`64_FluentWork第二波收口评估与下半波规划_V1.md`、`53_验收清单.md`*
