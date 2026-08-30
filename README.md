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
7. `docs/30_技术方案/31_FluentWork后端技术方案文档.md`
8. `docs/30_技术方案/32_FluentWork-iOS App端技术设计文档.md`
9. `docs/30_技术方案/33_FluentWork-Prompt工程与语料库设计文档.md`
10. `docs/50_测试与验收/50_FluentWork端到端注入能力验证文档.md`
11. `docs/50_测试与验收/51_FluentWork第一波能力验证与第二波薄弱点检查门禁.md`
12. `docs/50_测试与验收/52_FluentWork第一波遗留问题清账与第二波启动前计划.md`
13. `docs/60_评审与复盘/60_评审记录-W0.md`
14. `docs/60_评审与复盘/62_FluentWork第一波关闭记录.md`
15. `docs/50_测试与验收/50_FluentWork端到端注入能力验证文档.md`（B14；含执行状态）


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

- add issue templates
- add PR template
- add CODEOWNERS for meta
- expand document and workflow checks
