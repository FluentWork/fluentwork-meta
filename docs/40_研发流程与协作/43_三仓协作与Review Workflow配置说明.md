# 三仓协作与 Review Workflow 配置说明

**版本**：V1.1  
**日期**：2026-08  
**定位**：说明 `fluentwork-ios`、`fluentwork-backend`、`fluentwork-infra` 当前如何协作，以及本地 OpenCodeReview commit 门禁如何统一落地  
**上游依据**：`12_FluentWork-AI协作开源研发与CI-CD方案.md`、`agents/shared/review-gate.md`

---

## 一、当前三仓分工

## 1.1 fluentwork-ios

定位：SwiftUI 客户端主仓（状态管理、页面、服务接入、测试）

当前 workflow 重点：

1. `agent-config-check`
2. `ios-ci`

本地审查：`.githooks/pre-commit` → `Scripts/ocr-local-review.sh`

## 1.2 fluentwork-backend

定位：Go 后端主仓（API、gateway、worker、migration、契约）

当前 workflow 重点：

1. `agent-config-check`
2. `backend-ci`

本地审查：`.githooks/pre-commit` → `scripts/ocr-local-review.sh`

## 1.3 fluentwork-infra

定位：CI/CD、deploy、环境模板、workflow 与观测配置仓

当前 workflow 重点：

1. `agent-config-check`
2. `infra-ci`

本地审查：`.githooks/pre-commit` → `scripts/ocr-local-review.sh`

---

## 二、当前协作链路

```text
本地实现
  -> 本地自检
  -> 本地 OpenCodeReview（git pre-commit，critical/high 拦截）
  -> （可选）gstack /review 做更深结构审查
  -> push 分支
  -> GitHub PR
  -> CI checks
  -> 人类 owner 审批
```

口径：

1. **OpenCodeReview CLI（`ocr`）是 commit 门禁**，三仓统一；
2. **GitHub 不再运行** `opencode-review` workflow；
3. `gstack /review` 仍可用于本地更深审查，但不替代 OCR 门禁。

---

## 三、本地 OCR 门禁落地（统一）

每个代码仓：

1. `.githooks/pre-commit`：调用 `ocr-local-review.sh`
2. `ocr-local-review.sh`：`ocr review --format json --audience agent` → `ocr-fail-on-high.sh`
3. `setup-git-hooks.sh`：对本 clone 执行 `git config core.hooksPath .githooks`
4. `.opencodereview/rule.json`：仓内审查规则
5. 紧急旁路：`SKIP_OCR=1`（必须在 commit/PR 说明原因；不得用于掩盖安全类 high）

clone 后第一次：

```bash
# backend / infra
./scripts/setup-git-hooks.sh

# ios
./Scripts/setup-git-hooks.sh
```

手动预跑：

```bash
./scripts/ocr-local-review.sh   # or ./Scripts/...
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

说明：历史文件 `.github/workflows/opencode-review.yml` 已移除；审查改在本地 pre-commit。

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

OpenCodeReview **不是** GitHub required check；severity 门禁在本地 commit 阶段执行。

---

## 六、LLM / provider 配置（本地）

本地 `ocr` 使用开发者本机配置（`ocr config` / `~/.opencodereview/config.json`）。

组织级 GitHub secrets（`OPENCODEREVIEW_LLM_*`）不再被三仓 workflow 消费；可保留作将来恢复 CI 审查或个人参考，但当前链路不依赖它们。

---

## 七、推荐维护方式

1. **门禁规则**：`fluentwork-meta/agents/shared/review-gate.md`
2. **Git/PR 纪律**：`fluentwork-meta/agents/shared/git-and-pr-rules.md`
3. **各仓入口**：`AGENTS.md` / `CLAUDE.md` / `README.md`
4. **脚本契约**：三仓保持同名脚本语义一致（iOS 目录名为 `Scripts/`）

---

## 八、最终口径

> **commit 前本地跑 OpenCodeReview（critical/high 拦截）；CI 只做 build/test/lint；人类 owner 继续审批高风险路径；gstack /review 可选加强，不替代 OCR 门禁。**
