# AI Collaboration

## 目标

统一 FluentWork 多仓中的 AI 协作方式，避免不同仓库出现互相冲突的 agent 行为。

## 共享原则

1. AI 是正式产能，但不是最终拍板者；
2. 共享规则集中在 `fluentwork-meta/agents/`；
3. 各仓根目录必须有自己的 `CLAUDE.md` 和 `AGENTS.md`；
4. 各仓入口文件只写本仓特有约束，不重复抄整份共享规则；
5. 所有 agent 产出都要能落到 PR、测试、文档或变更记录中。

## 角色分工

1. `Trae`：治理文档、多文档梳理、计划与决策记录；
2. `Claude Code`：主力实现、改代码、补测试、修 CI；
3. `第二审查 AI`：PR 风险审查、漏测与回归提示；
4. `人类 owner`：关键目录审批、架构仲裁、发布决策。

## 通用要求

1. 优先最小差异修改；
2. 改动代码时同步补测试或明确说明不能补的原因；
3. 改动行为、流程、接口时同步更新文档；
4. 不得擅自绕过 branch protection、CODEOWNERS、required checks；
5. 未经明确授权，不做 destructive git 操作。
