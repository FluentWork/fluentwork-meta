# Git And PR Rules

## Git 纪律

1. 默认基于 issue 或明确任务工作；
2. 每个提交聚焦单一主题；
3. 不擅自 amend 用户已有提交；
4. 不擅自 `reset --hard`、`checkout --`、`push --force`；
5. 未经用户要求，不主动提交 secrets、证书、环境密钥或大体积产物。

## 分支约定

1. `main`：可发布；
2. `develop`：日常集成；
3. `feature/<issue-id>-<topic>`：功能分支；
4. `release/<version>`：预发布；
5. `hotfix/<topic>`：线上修复。

## PR 最低要求

1. 关联 issue；
2. 说明上游文档；
3. 写清验收点；
4. 给出测试结果；
5. 标明风险点；
6. 若涉及第二审查 AI，记录其主要 findings。

## 合入前检查

1. 本地自检已完成；
2. CI 必须通过；
3. OpenCodeReview：若存在 `high` finding，必须先修复再合；无 `high` 方可合入（详见 `review-gate.md`）；
4. CODEOWNERS 要求满足；
5. 高风险目录需 owner 审批；
6. 文档、测试、实现三者口径一致。
