# FluentWork Meta

## Repo Role

`fluentwork-meta` is the governance and source-of-truth repository for FluentWork.

It owns:

1. product and architecture documents
2. project governance and workflow rules
3. shared agent policy and templates
4. milestone planning and review records

## Shared Source Of Truth

Shared agent rules live in:

- `agents/shared/ai-collaboration.md`
- `agents/shared/git-and-pr-rules.md`
- `agents/shared/review-gate.md`
- `agents/shared/skills-policy.md`
- `agents/shared/matt-pocock-skills.md`

This repository is itself the source of truth for those shared files.

## Repo-Specific Constraints

1. Keep repository root minimal.
2. Governance changes must stay aligned with existing project docs.
3. Do not rename or move core docs casually; preserve the numbering scheme.
4. When shared agent rules change, update templates in `agents/templates/` in the same change.
5. Do not treat this repo as an implementation repo for iOS, backend, or infra code.

## High-Risk Areas

1. `docs/10_项目治理/`
2. `agents/shared/`
3. `agents/templates/`
4. future `.github/` workflow and template files

## Expected Workflow

1. Read current governance docs before editing.
2. Prefer minimal diffs over broad rewrites.
3. When a rule changes, update the corresponding plan or policy doc.
4. Keep shared rules centralized here; code repos should remain thin.
5. External skills may assist, but FluentWork governance wins on conflicts.
