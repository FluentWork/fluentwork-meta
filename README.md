# FluentWork Meta

`fluentwork-meta` is the governance and specification repository for FluentWork.

## Purpose

This repository stores the project's shared source of truth:

- product requirements
- UX and interaction design
- architecture and ADRs
- roadmap and milestone planning
- review records and release gates
- GitHub workflow conventions for the FluentWork organization

## Suggested Structure

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

## Working Rules

1. Product decisions are recorded before implementation starts.
2. Architecture and scope changes must be reflected in docs.
3. Every execution repository should link back to the relevant spec or issue.
4. Release and review criteria should be maintained here before rollout.

## Related Repositories

- `fluentwork-ios`
- `fluentwork-backend`
- `fluentwork-infra`

## Next Initialization Targets

- add issue templates
- add PR template
- add ADR index
- sync current local docs into this repository
