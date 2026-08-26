# 三仓协作与 Review Workflow 配置说明

**版本**：V1.2  
**日期**：2026-08  
**定位**：说明 `fluentwork-ios`、`fluentwork-backend`、`fluentwork-infra` 当前如何协作，以及本地 gstack `/review` commit 门禁如何统一落地  
**上游依据**：`12_FluentWork-AI协作开源研发与CI-CD方案.md`、`agents/shared/review-gate.md`

---

## 一、当前三仓分工

## 1.1 fluentwork-ios

定位：SwiftUI 客户端主仓（状态管理、页面、服务接入、测试）

当前 workflow 重点：

1. `agent-config-check`
2. `ios-ci`

本地审查：`.githooks/pre-commit` → `Scripts/gstack-review-gate.sh`

## 1.2 fluentwork-backend

定位：Go 后端主仓（API、gateway、worker、migration、契约）

当前 workflow 重点：

1. `agent-config-check`
2. `backend-ci`

本地审查：`.githooks/pre-commit` → `scripts/gstack-review-gate.sh`

## 1.3 fluentwork-infra

定位：CI/CD、deploy、环境模板、workflow 与观测配置仓

当前 workflow 重点：

1. `agent-config-check`
2. `infra-ci`

本地审查：`.githooks/pre-commit` → `scripts/gstack-review-gate.sh`

---

## 二、当前协作链路

```text
本地实现
  -> 本地自检
  -> gstack /review（Cursor skill）
  -> git commit（pre-commit 要求 GSTACK_REVIEWED=1）
  -> push 分支
  -> GitHub PR
  -> CI checks（build / test / lint / config）
  -> 人类 owner 审批
```

口径：

1. **gstack `/review` 是主审查**；本地 pre-commit 以 attestation 强制声明已跑过；
2. **GitHub 不运行** `opencode-review` 或其它 code-review workflow；
3. OpenCodeReview CLI 仅可选手工，不参与默认门禁。

---

## 三、本地 gstack `/review` 门禁落地（统一）

每个代码仓：

1. `.githooks/pre-commit`：调用 `gstack-review-gate.sh`
2. `gstack-review-gate.sh`：要求 `GSTACK_REVIEWED=1`（或紧急 `SKIP_GSTACK_REVIEW=1`）
3. `setup-git-hooks.sh`：对本 clone 执行 `git config core.hooksPath .githooks`
4. 技能本身在 Cursor/Claude 中交互执行；bash 无法代跑 skill

clone 后第一次：

```bash
# backend / infra
./scripts/setup-git-hooks.sh

# ios
./Scripts/setup-git-hooks.sh
```

正常提交：

```bash
# 先在 Cursor 中跑 gstack /review，再：
GSTACK_REVIEWED=1 git commit -m "..."
```

紧急旁路（须在 commit/PR 说明原因）：

```bash
SKIP_GSTACK_REVIEW=1 git commit -m "..."
```

---

## 四、当前 workflow 文件位置

## 4.1 iOS

- `.github/workflows/agent-config-check.yml`
- `.github/workflows/ios-ci.yml`

## 4.2 Backend

- `.github/workflows/agent-config-check.yml`
- `.github/workflows/backend-ci.yml`

## 4.3 Infra

- `.github/workflows/agent-config-check.yml`
- `.github/workflows/infra-ci.yml`

说明：历史文件 `.github/workflows/opencode-review.yml` 已移除；组织策略为 **CI 不做 code review**。

---

## 五、当前 main 分支门禁

三个代码仓当前都启用了：

1. `Require a pull request before merging`
2. `Require approvals: 1`
3. `Dismiss stale pull request approvals when new commits are pushed`
4. `Require branches to be up to date before merging`
5. `Require conversation resolution before merging`
6. `Require linear history`

各仓 required checks 如下：

1. `fluentwork-ios`：`agent-config-check` / `repo-structure-check` / `swift-package-test`
2. `fluentwork-backend`：`agent-config-check` / `repo-structure-check` / `go-build-and-test`
3. `fluentwork-infra`：`agent-config-check` / `repo-structure-check` / `workflow-lint`

Code review **不是** GitHub required check；审查在本地 commit / 合入前由 gstack `/review` 执行。

---

## 六、LLM / provider 配置（本地）

gstack `/review` 使用开发者本机 Cursor/Claude 会话；不依赖组织级 `OPENCODEREVIEW_LLM_*` secrets。

历史 OCR 相关 secrets 可保留作归档，当前链路不依赖它们。

---

## 七、推荐维护方式

1. **门禁规则**：`fluentwork-meta/agents/shared/review-gate.md`
2. **Git/PR 纪律**：`fluentwork-meta/agents/shared/git-and-pr-rules.md`
3. **各仓入口**：`AGENTS.md` / `CLAUDE.md` / `README.md`
4. **脚本契约**：三仓保持同名脚本语义一致（iOS 目录名为 `Scripts/`）

---

## 八、最终口径

> **commit 前跑 gstack `/review` 并以 `GSTACK_REVIEWED=1` 提交；CI 只做 build/test/lint；人类 owner 继续审批高风险路径；CI 不跑 code review。**
