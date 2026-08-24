# AGENTS

## Repository

- Name: `fluentwork-meta`
- Role: governance, documentation, shared agent policy, and project planning

## Shared Rules

This repository is the shared source of truth for FluentWork agent policy.

Primary shared files:

1. `agents/shared/ai-collaboration.md`
2. `agents/shared/git-and-pr-rules.md`
3. `agents/shared/review-gate.md`
4. `agents/shared/skills-policy.md`
5. `agents/shared/matt-pocock-skills.md`

## Local Rules

1. Keep document numbering stable.
2. Keep root files minimal and intentional.
3. Update templates when shared rules change.
4. Do not put implementation-specific iOS or backend rules here unless they are truly cross-repo governance.

## Required Behaviors

1. Read upstream governance docs before editing.
2. Keep changes scoped to the active policy or planning topic.
3. Preserve traceability between decision docs and execution plans.
4. Avoid unnecessary renames and path churn.
5. Surface cross-repo implications when changing shared rules.

## High-Risk Paths

1. `docs/10_项目治理/`
2. `agents/shared/`
3. `agents/templates/`

## CI Boundary

CI should validate configuration and document consistency. CI should not run a full interactive skills runtime here.
