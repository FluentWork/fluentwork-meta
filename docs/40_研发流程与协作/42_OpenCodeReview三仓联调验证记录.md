# OpenCodeReview 三仓联调验证记录

**版本**：V1.0  
**日期**：2026-08  
**定位**：记录 FluentWork 三个代码仓首次 OpenCodeReview 联调结果，确认 GitHub PR 审查链路是否已接通  
**上游依据**：`40_OpenCodeReview代码审查接入方案.md`、`43_三仓协作与Review Workflow配置说明.md`

---

## 一、验证目标

本次验证只确认三件事：

1. `opencode-review.yml` 是否在 PR 打开后自动触发；
2. GitHub 组织级 secrets 是否已正确注入到三个代码仓；
3. OpenCodeReview 是否能把结果写回 PR 评论区。

---

## 二、验证环境

当前 provider 配置为：

1. 协议：OpenAI-compatible
2. Provider：DeepSeek API
3. `OPENCODEREVIEW_LLM_URL`：`https://api.deepseek.com/chat/completions`
4. `OPENCODEREVIEW_LLM_MODEL`：`deepseek-v4-pro`
5. `llm_use_anthropic`：`false`

说明：

- 这组 secrets 配在 GitHub 组织 `FluentWork`；
- 只授权给 `fluentwork-ios`、`fluentwork-backend`、`fluentwork-infra`；
- `fluentwork-meta` 当前未接 OpenCodeReview workflow。

---

## 三、测试 PR

本次使用统一测试分支：

- `test/ocr-verification-20260824-193000`

对应 PR：

1. iOS：`https://github.com/FluentWork/fluentwork-ios/pull/1`
2. Backend：`https://github.com/FluentWork/fluentwork-backend/pull/1`
3. Infra：`https://github.com/FluentWork/fluentwork-infra/pull/1`

这些 PR 都是**联调用测试 PR，不应合并**。

---

## 四、测试方法

为了提高验证信号，每个仓都放入一个“应被 reviewer 指出的问题”：

1. iOS：Swift `force unwrap`
2. Backend：Go nil 指针解引用
3. Infra：Shell 中危险的 `rm -rf` 写法

同时保持现有 CI 可运行，避免把结果混淆成普通编译失败。

---

## 五、验证结果

## 5.1 fluentwork-ios

结果：

- `opencode-review` workflow 成功触发并完成；
- PR 评论区写入了 1 条 summary 评论；
- 识别到 `value!` 的运行时崩溃风险；
- 按当前策略，问题被路由到 summary，没有生成 inline comment。

结论：

> **iOS 仓 OpenCodeReview 已接通，可正常输出审查结果。**

## 5.2 fluentwork-backend

结果：

- `opencode-review` workflow 成功触发并完成；
- PR 评论区写入了 1 条 summary 评论；
- 识别到导出函数中的 nil 指针解引用风险；
- 按当前策略，问题被路由到 summary，没有生成 inline comment。
- 同次联调中，`backend-ci` 的 `go-build-and-test` 额外暴露出一处基础设施问题：
  workflow 安装的是 `golangci-lint` v1，但仓库中的 `.golangci.yml` 已按 v2 配置编写；
- 该问题会导致 CI 在 lint 阶段失败，但**不影响 OpenCodeReview 正常评论**；
- 随后又补了 `revive` 所要求的 package / exported symbol 注释，最终让 `go-build-and-test` required check 转绿；
- 在后续重跑中，OpenCodeReview 继续正常更新 summary，并额外指出了 `golangci-lint@latest` 的可复现性问题。

结论：

> **Backend 仓 OpenCodeReview 已接通，可正常输出审查结果。**

## 5.3 fluentwork-infra

结果：

- `opencode-review` workflow 成功触发并完成；
- PR 评论区写入了 summary；
- 同时在 `scripts/ocr-verification.sh` 上成功生成 1 条 inline review comment；
- 正确指出了未加引号、未校验空值、危险 `rm -rf` 的问题。

结论：

> **Infra 仓 OpenCodeReview 已接通，并且已验证 inline comment 能正常落到代码行。**

---

## 六、整体结论

本次联调结论如下：

1. 三个代码仓的 OpenCodeReview workflow 已全部接通；
2. 组织级 secrets 注入正常；
3. DeepSeek API 的 OpenAI-compatible 配置可用；
4. GitHub PR 中的 summary 评论已验证成功；
5. inline comment 已在 `infra` 仓验证成功。

因此可以正式确认：

> **FluentWork 当前三个代码仓已经具备可用的 OpenCodeReview 第二审查能力。**

补充说明：

- 本次联调同时验证出 backend CI 还存在一处 `golangci-lint` 版本兼容问题；
- 该问题随后已在测试分支上修复并验证通过；
- 这说明当前 backend 仓的 required checks 与 OpenCodeReview workflow 可以并行正常工作；
- `sticky_summary` 会覆盖旧的 summary 内容，因此若要保留联调证据，建议在验证文档中同时记录“首次命中结果”和“最终 PR 状态”；
- 该问题属于普通 CI 配置问题，不影响对 OpenCodeReview 本身是否可用的判断；
- 因此“审查链路已接通”和“个别仓的基础 CI 仍需继续打磨”这两个结论需要分开看。

---

## 七、当前建议

基于本次结果，建议继续保持：

1. 本地开发时主要使用 `gstack /review` 做快速自检；
2. GitHub PR 中使用 OpenCodeReview 做第二审查；
3. 先保持“report-first”，不把 OpenCodeReview 直接升为 required check；
4. 优先在 `backend` 与 `infra` 中观察一段时间，再决定是否收紧策略。

---

## 八、后续动作

后续应考虑：

1. 关闭或保留这三个测试 PR 作为联调记录；
2. 视误报情况调整 `route_severity_below`；
3. 视 provider 成本与效果，决定是否切换到其他 OpenAI-compatible 或 Anthropic-compatible 模型；
4. 若后续要接入 `fluentwork-meta`，单独评估是否值得对文档仓启用 OpenCodeReview。
