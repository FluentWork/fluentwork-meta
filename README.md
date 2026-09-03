# FluentWork Meta

`fluentwork-meta` is the governance and specification repository for FluentWork.

GitHub 组织已建立：

- `https://github.com/FluentWork`

已初始化仓库：

- `https://github.com/FluentWork/fluentwork-meta`
- `https://github.com/FluentWork/fluentwork-ios`
- `https://github.com/FluentWork/fluentwork-backend`
- `https://github.com/FluentWork/fluentwork-infra`

当前本地仓库 remote：

- `git@github.com:FluentWork/fluentwork-meta.git`

## Purpose

This repository stores the project's shared source of truth:

- product requirements
- UX and interaction design
- architecture and ADRs
- roadmap and milestone planning
- review records and release gates
- shared agent policy and templates
- GitHub workflow conventions for the FluentWork organization

## 目录

```text
docs/
├── 10_项目治理/
├── 20_产品设计/
├── 30_技术方案/
├── 40_研发流程与协作/
├── 50_测试与验收/
├── 60_评审与复盘/
└── 70_外部研究/
```

## 阅读顺序

1. `docs/10_项目治理/10_FluentWork项目启动书.md`
2. `docs/10_项目治理/11_FluentWork团队分工文档.md`
3. `docs/10_项目治理/12_FluentWork-AI协作开源研发与CI-CD方案.md`
4. `docs/20_产品设计/20_FluentWork产品需求文档PRD.md`
5. `docs/20_产品设计/21_FluentWork界面设计文档.md`
6. `docs/30_技术方案/30_FluentWork技术方案设计文档.md`
7. `docs/30_技术方案/31_FluentWork后端技术方案文档.md`（V1.2 设计基线）
8. `docs/30_技术方案/32_FluentWork-iOS App端技术设计文档.md`
9. `docs/30_技术方案/33_FluentWork-Prompt工程与语料库设计文档.md`
10. `docs/30_技术方案/34_FluentWork火山引擎选型与开通清单.md`
11. `docs/30_技术方案/39_FluentWork后端技术架构设计V2_0_2026-09-03.md`（**V2.0 架构新增 LLM 编排 / TTS / 闪测三模块**）
12. `docs/30_技术方案/44_B7_LLM注入完整接口契约_2026-09-03.md`（**B19 前置契约 + phrase_block_uses 新表 + 10 测试用例**）
13. `docs/30_技术方案/45_A4_隐私声明与级联删除完整规格_2026-09-03.md`（**第二波 OPEN 项清账；文案 + 级联删除 + 8 测试用例**）
14. `docs/30_技术方案/46_F3_B4_收藏置顶与重播完整规格_2026-09-03.md`（**F3 置顶排序规则 + B4 重播 UI 规格 + 波形预计算**）
15. `docs/40_研发流程与协作/47_D1_D5_开放技术决策备忘录_2026-09-03.md`（**D-1~D-5 决策备忘录；周六上午决策会议预备材料**）
16. `docs/50_测试与验收/57_FluentWork评估集100条样本设计_2026-09-03.md`（**B8/B18 验证必需；100 条分层样本 + 4 阶段建立计划**）
17. `docs/40_研发流程与协作/53_FluentWork_V2_0启动前待办清单_2026-09-03.md`（**V2.0 实施前的硬阻塞、软阻塞、Issue 拆分**）
18. `docs/40_研发流程与协作/54_FluentWork_V2_0周末作战地图与Issue草案集_2026-09-03.md`（**9/5-9/6 集中攻克：5 Epic × 16 Issue 草案 + 测试用例 + 验收标准**）
13. `docs/50_测试与验收/50_FluentWork端到端注入能力验证文档.md`
14. `docs/50_测试与验收/51_FluentWork第一波能力验证与第二波薄弱点检查门禁.md`
15. `docs/50_测试与验收/52_FluentWork第一波遗留问题清账与第二波启动前计划.md`
16. `docs/50_测试与验收/56_FluentWork第二波PRD对照检查与收口评估_2026-09-03.md`（**第二波收口评估，含 V2.0 缺口分析**）
17. `docs/60_评审与复盘/65_FluentWork_说读炼化闪测语料库_闭环状态收口报告_2026-09-03.md`（**五环节完成度 + 阻塞根因 + 后续计划**）
18. `docs/60_评审与复盘/60_评审记录-W0.md`
19. `docs/60_评审与复盘/62_FluentWork第一波关闭记录.md`


## Working Rules

1. Product decisions are recorded before implementation starts.
2. Architecture and scope changes must be reflected in docs.
3. Every execution repository should link back to the relevant spec or issue.
4. Release and review criteria should be maintained here before rollout.
5. Root files should remain minimal and intentional.

## 当前说明

- 正式文档统一进入 `docs/` 对应分类目录。
- 核心文档已完成第二轮编号重命名，统一采用 `数字前缀_主题名.md`。
- 共享 agent 规则与模板统一放在 `agents/` 下维护。
- GitHub 组织和 4 个基础仓库已经初始化完成。
- 当前治理与规格文档以 `fluentwork-meta` 为真源。

## Related Repositories

- `fluentwork-ios`
- `fluentwork-backend`
- `fluentwork-infra`

## Next Initialization Targets

- [ ] V2.0 五项开放技术决策拍板（D-1 ~ D-5，详见 `40_研发流程与协作/53_`）
- [ ] 火山 Ark LLM + TTS 凭证申请（W2 末必须到位）
- [ ] 18 个 V2.0 一级 Issue 在三仓建立（参考 `40_研发流程与协作/52_` 模式）
- [ ] V2.0 数据库迁移脚本入库（drill_records 等）
- [ ] add issue templates
- [ ] add PR template
- [ ] add CODEOWNERS for meta
- [ ] expand document and workflow checks
