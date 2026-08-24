# Skills Policy

## 目标

规定 FluentWork 如何使用共享 skills、repo-specific skills 和外部 skills。

## 当前决策

1. 不单独创建共享 skills 仓；
2. 共享真源保存在 `fluentwork-meta/agents/`；
3. 各仓自己持有 `CLAUDE.md` / `AGENTS.md`；
4. 共享规则集中，仓库入口分散。

## 共享 skills

以下内容允许作为共享规则：

1. AI 协作与角色分工；
2. Git / PR / review gate 纪律；
3. 外部 skills 的允许范围；
4. CI 中对 agent 文件的检查方式。

## repo-specific skills

以下内容必须放在各仓本地：

1. iOS 仓的 SwiftUI / AudioEngine / SpeechSession 限制；
2. Backend 仓的 gateway / migration / data safety 限制；
3. Infra 仓的 deploy / secrets / rollback 限制；
4. Meta 仓的文档治理与编号规范。

## CI 边界

CI 可以做：

1. 检查 `CLAUDE.md` / `AGENTS.md` 是否存在；
2. 检查模板是否同步；
3. 检查共享规则引用是否正确。

CI 不做：

1. 加载交互式 skills runtime；
2. 依赖本地 agent 环境；
3. 把 skills 当 required check 的执行主体。
