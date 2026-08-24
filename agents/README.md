# FluentWork Agents

本目录是 FluentWork 多仓共享的 agent / skills 治理真源。

## 目录

```text
agents/
├── README.md
├── shared/
│   ├── ai-collaboration.md
│   ├── git-and-pr-rules.md
│   ├── review-gate.md
│   ├── skills-policy.md
│   └── matt-pocock-skills.md
└── templates/
    ├── CLAUDE.md.template
    └── AGENTS.md.template
```

## 约束

1. 这里是共享真源，不直接替代各仓根目录的 `CLAUDE.md` / `AGENTS.md`；
2. `fluentwork-ios`、`fluentwork-backend`、`fluentwork-infra`、`fluentwork-meta` 各仓仍需保留自己的入口文件；
3. 共享规则在这里维护，各仓入口文件只补本仓特有内容；
4. CI 后续只校验入口文件与模板的一致性，不在 CI 中运行整套 skills runtime。
