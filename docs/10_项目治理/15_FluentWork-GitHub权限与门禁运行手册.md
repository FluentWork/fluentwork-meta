# FluentWork GitHub 权限与门禁运行手册

**版本**：V1.0
**日期**：2026-08
**定位**：把 GitHub 组织权限、仓库门禁、分支保护与例外处理收口为可直接执行的运行手册
**上游依据**：`10_FluentWork项目启动书.md`、`12_FluentWork-AI协作开源研发与CI-CD方案.md`、`13_FluentWork仓库初始化与CI-CD执行计划.md`、`43_三仓协作与Review Workflow配置说明.md`
**变更说明**：补齐治理总方案之后缺失的执行级 runbook，统一“谁能改、怎么审、什么能放行、出了例外谁拍板”的口径
**状态**：执行文档，可直接用于组织与仓库配置

---

## 一、目的

本手册只回答 4 个问题：

1. GitHub 组织中谁拥有什么权限；
2. 四个仓库的 `main` 应被什么门禁保护；
3. 什么情况下允许例外放行；
4. 当门禁、权限或 required checks 要调整时，应按什么顺序操作。

本手册不讨论业务功能实现，不替代技术方案或测试计划，只定义 GitHub 治理面的运行规则。

---

## 二、适用范围

适用于以下仓库：

1. `fluentwork-meta`
2. `fluentwork-ios`
3. `fluentwork-backend`
4. `fluentwork-infra`

默认分支统一为 `main`。

---

## 三、权限模型

## 3.1 Team 划分

GitHub 组织至少保留 4 个 team：

1. `core`
2. `ios`
3. `backend`
4. `infra`

其中：

- `core` 是最终治理 owner；
- `ios`、`backend`、`infra` 分别承接各自代码仓的日常 owner 职责；
- AI 工具不属于 team 成员，不直接授予组织级权限。

## 3.2 组织级权限边界

### `core`

可执行：

1. 修改组织级 branch protection；
2. 修改仓库级 required checks；
3. 修改组织级 secrets / variables；
4. 修改 team 权限；
5. 执行发布审批；
6. 在阻断主线时执行受控 bypass。

### `ios` / `backend` / `infra`

可执行：

1. 审核本仓 PR；
2. 维护本仓目录 owner 规则；
3. 修改本仓非组织级 workflow；
4. 发起 required checks 调整申请。

不可直接执行：

1. 修改组织级权限模型；
2. 修改其他业务仓的 owner 规则；
3. 擅自降低 `main` 分支保护强度。

## 3.3 仓库 owner 归属建议

| 仓库 | 日常 owner team | 最终治理 owner |
|---|---|---|
| `fluentwork-meta` | `core` | `core` |
| `fluentwork-ios` | `ios` | `core` |
| `fluentwork-backend` | `backend` | `core` |
| `fluentwork-infra` | `infra` | `core` |

---

## 四、分支保护基线

## 4.1 `main` 的统一要求

四个仓库的 `main` 默认开启以下规则：

1. 禁止普通成员直接 push；
2. 必须通过 Pull Request 合并；
3. 至少 1 次审批；
4. 要求 PR 分支与 `main` 同步后再 merge；
5. required checks 全部通过后才能 merge；
6. 关键目录变更受 `CODEOWNERS` 保护。

## 4.2 允许 bypass 的条件

只有 `core` 可执行 bypass，且仅限以下情况：

1. 分支保护配置本身失效，导致正常修复无法进入主线；
2. CI 平台故障，但修复内容本身已经通过本地同等验证；
3. 生产事故或发布事故需要快速止血；
4. 明确属于治理修复，且继续等待 PR 会扩大阻塞面。

不允许作为 bypass 理由的情况：

1. “赶进度”；
2. “只是小改动”；
3. “AI 说没问题”；
4. “review 太慢”。

## 4.3 bypass 后的补动作

凡是 bypass 进入 `main` 的改动，必须在同日补齐：

1. 事件说明；
2. 本地验证证据；
3. 为什么不能走正常 PR；
4. 后续恢复动作。

记录位置：

- 治理类问题写回 `fluentwork-meta/docs/40_研发流程与协作/`
- 事故或发布类问题写回对应仓的 release / rollback 记录

---

## 五、required checks 基线

## 5.1 当前原则

FluentWork 采用：

> **CI 必须阻断，第二审查 AI 先报告后阻断。**

也就是说：

1. 编译、测试、格式、结构校验属于硬门禁；
2. OpenCodeReview 当前属于第二审查层，默认不是 required check；
3. 只有在误报率、稳定性和处理成本都可接受后，才考虑把 AI 审查提升为强门禁。

## 5.2 各仓推荐 required checks

### `fluentwork-meta`

1. `meta-docs-check`
2. `agent-config-check`

后续增强：

1. `markdown-lint`
2. `link-check`
3. `actionlint`
4. `docs-filename-check`

### `fluentwork-ios`

1. `ios-ci`
2. `agent-config-check`

后续如 workflow 拆分，再细化为：

1. `ios-build`
2. `ios-unit-test`
3. `ios-smoke`

### `fluentwork-backend`

1. `backend-ci`
2. `agent-config-check`

后续如 workflow 拆分，再细化为：

1. `gofumpt`
2. `goimports`
3. `golangci-lint`
4. `go-test`
5. `docker-build`

### `fluentwork-infra`

1. `infra-ci`
2. `agent-config-check`

后续增强：

1. `workflow-lint`
2. `deploy-config-check`
3. `deploy-dry-run`

## 5.3 非 required 的当前项

当前不建议直接设为 required：

1. `opencode-review`
2. 文档仓上的 AI 自动审查
3. 重度 E2E 或高成本真机验证

理由：

1. 这几类检查当前更适合作为报告信号；
2. 误报和外部依赖波动还未稳定；
3. 项目仍处于治理骨架到业务骨架过渡期，应避免过早把主线锁死。

---

## 六、目录 owner 与审查边界

## 6.1 `fluentwork-meta`

以下内容默认要求 `core` 审批：

1. `docs/10_项目治理/`
2. `agents/shared/`
3. `agents/templates/`
4. `.github/workflows/`

## 6.2 `fluentwork-ios`

以下内容默认要求 `ios` 或 `core` 审批：

1. AudioEngine
2. SpeechSession 状态机
3. 根状态容器与依赖注入骨架
4. 发布相关 workflow

## 6.3 `fluentwork-backend`

以下内容默认要求 `backend` 或 `core` 审批：

1. `cmd/voice-gateway/`
2. 状态机与协议适配层
3. migration
4. deploy / prod 配置相关 workflow

## 6.4 `fluentwork-infra`

以下内容默认要求 `infra` 或 `core` 审批：

1. `.github/workflows/`
2. `deploy/`
3. 环境模板
4. rollback 相关脚本

---

## 七、变更顺序与操作纪律

## 7.1 调整 GitHub 门禁时的固定顺序

任何涉及 review、workflow、branch protection 的调整，统一按下面顺序：

1. **先改 secrets / variables**
2. **再改 workflow**
3. **最后改 branch protection**

这样做的原因是：

1. 先保证新检查有配置基础；
2. 再让 workflow 跑起来；
3. 最后才把它升成硬门禁，避免出现“门禁先开，检查还没稳定”的死锁。

## 7.2 调整 required checks 的审批规则

以下动作必须由 `core` 拍板：

1. 新增 required check；
2. 降级或移除 required check；
3. 把 AI 审查从报告模式升为阻断模式；
4. 修改 `main` 的审批人数；
5. 允许 bypass 某项门禁。

## 7.3 变更记录要求

以下治理动作必须回写文档：

1. team 权限模型变化；
2. `main` 保护规则变化；
3. required checks 变化；
4. OpenCodeReview 策略变化；
5. 任何一次 bypass 主线。

优先回写位置：

1. `docs/10_项目治理/`：规则变化
2. `docs/40_研发流程与协作/`：运行方式变化
3. `docs/60_评审与复盘/`：阶段性复盘

---

## 八、日常运行检查表

## 8.1 新仓初始化时

1. 仓库 owner team 已绑定；
2. `main` 已受保护；
3. `CODEOWNERS` 已存在；
4. `CLAUDE.md` / `AGENTS.md` 已存在；
5. 至少 1 条 CI 成功；
6. PR 模板与 Issue 模板已可用；
7. required checks 已与实际 workflow 名称对齐。

## 8.2 新 workflow 上线时

1. 本地或测试分支已验证；
2. workflow 名称稳定；
3. secrets 已存在；
4. 不会产生超范围写权限；
5. 先观察，再决定是否升为 required。

## 8.3 发布前

1. `main` 无临时放宽规则；
2. required checks 处于绿色；
3. 发布审批人明确；
4. rollback 路径存在；
5. 例外放行策略未被滥用。

---

## 九、当前执行状态判断

以 2026-08 当前仓库状态看：

1. 三个代码仓已具备基础 CI、agent 配置校验和 OpenCodeReview 第二审查；
2. `main` 主线已具备基本可用的保护与检查能力；
3. `fluentwork-meta` 的治理文档与共享 agent 模板已存在；
4. 但 `meta` 仓的 `CODEOWNERS`、Issue / PR 模板与更完整的文档检查项仍需继续补齐；
5. OpenCodeReview 仍应保持 report-first，不宜立即升为 required check。

因此当前结论是：

> **治理面已从“总方案阶段”进入“可运行阶段”，但还没到“全部收紧”阶段。**

---

## 十、最终口径

FluentWork 当前 GitHub 治理的标准口径应统一为：

1. `main` 必须受保护；
2. 编译、测试、结构校验是硬门禁；
3. OpenCodeReview 是第二审查层，先报告后收紧；
4. 只有 `core` 可以调整主线门禁与执行 bypass；
5. 任何治理例外都必须留痕并补回文档。
