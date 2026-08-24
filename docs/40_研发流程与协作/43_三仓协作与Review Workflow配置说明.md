# 三仓协作与 Review Workflow 配置说明

**版本**：V1.0  
**日期**：2026-08  
**定位**：说明 `fluentwork-ios`、`fluentwork-backend`、`fluentwork-infra` 当前如何协作，GitHub workflow 如何配置，以及未来如何切换 review provider  
**上游依据**：`12_FluentWork-AI协作开源研发与CI-CD方案.md`、`13_FluentWork仓库初始化与CI-CD执行计划.md`、`40_OpenCodeReview代码审查接入方案.md`

---

## 一、当前三仓分工

## 1.1 fluentwork-ios

定位：

- SwiftUI 客户端主仓
- 状态管理、页面、服务接入、测试

当前 workflow 重点：

1. `agent-config-check`
2. `ios-ci`
3. `opencode-review`

## 1.2 fluentwork-backend

定位：

- Go 后端主仓
- API、gateway、worker、migration、契约

当前 workflow 重点：

1. `agent-config-check`
2. `backend-ci`
3. `opencode-review`

## 1.3 fluentwork-infra

定位：

- CI/CD、deploy、环境模板、workflow 与观测配置仓

当前 workflow 重点：

1. `agent-config-check`
2. `infra-ci`
3. `opencode-review`

---

## 二、当前协作链路

当前建议的链路是：

```text
本地实现
  -> 本地自检
  -> gstack /review
  -> push 分支
  -> GitHub PR
  -> CI checks
  -> OpenCodeReview
  -> 人类 owner 审批
```

也就是说：

1. `gstack` 主要用于本地开发阶段；
2. OpenCodeReview 主要用于 GitHub PR 阶段；
3. 二者不是二选一，而是前后衔接。

---

## 三、当前 workflow 文件位置

## 3.1 iOS

- [agent-config-check.yml](file:///Users/bigapple/Developments/fluentwork-ios/.github/workflows/agent-config-check.yml)
- [ios-ci.yml](file:///Users/bigapple/Developments/fluentwork-ios/.github/workflows/ios-ci.yml)
- [opencode-review.yml](file:///Users/bigapple/Developments/fluentwork-ios/.github/workflows/opencode-review.yml)

## 3.2 Backend

- [agent-config-check.yml](file:///Users/bigapple/Developments/fluentwork-backend/.github/workflows/agent-config-check.yml)
- [backend-ci.yml](file:///Users/bigapple/Developments/fluentwork-backend/.github/workflows/backend-ci.yml)
- [opencode-review.yml](file:///Users/bigapple/Developments/fluentwork-backend/.github/workflows/opencode-review.yml)

## 3.3 Infra

- [agent-config-check.yml](file:///Users/bigapple/Developments/fluentwork-infra/.github/workflows/agent-config-check.yml)
- [infra-ci.yml](file:///Users/bigapple/Developments/fluentwork-infra/.github/workflows/infra-ci.yml)
- [opencode-review.yml](file:///Users/bigapple/Developments/fluentwork-infra/.github/workflows/opencode-review.yml)

---

## 四、当前 main 分支门禁

三个代码仓当前都启用了：

1. `Require a pull request before merging`
2. `Require approvals: 1`
3. `Dismiss stale pull request approvals when new commits are pushed`
4. `Require branches to be up to date before merging`
5. `Require conversation resolution before merging`
6. `Require linear history`

各仓 required checks 如下：

1. `fluentwork-ios`
   - `agent-config-check`
   - `repo-structure-check`
   - `swift-package-test`
2. `fluentwork-backend`
   - `agent-config-check`
   - `repo-structure-check`
   - `go-build-and-test`
3. `fluentwork-infra`
   - `agent-config-check`
   - `repo-structure-check`
   - `workflow-lint`

说明：

- `opencode-review` 当前**不是 required check**；
- 它作为第二审查层提供评论信号，但暂不阻断 merge；
- 这样做是为了先观察模型稳定性、误报率和评论密度。

---

## 五、当前 OpenCodeReview 配置方式

三个仓的 `opencode-review.yml` 都采用同一模式：

1. `pull_request` 触发；
2. 先检查 secrets 是否存在；
3. 若 secrets 缺失则跳过；
4. 若 secrets 存在则调用 `alibaba/open-code-review@main`。

当前关键参数是：

1. `llm_url`
2. `llm_auth_token`
3. `llm_model`
4. `llm_use_anthropic`
5. `language`
6. `sticky_summary`
7. `incremental`
8. `route_severity_below`

---

## 六、当前 provider 配置

当前组织级 secrets 使用的是：

1. `OPENCODEREVIEW_LLM_URL`
2. `OPENCODEREVIEW_LLM_TOKEN`
3. `OPENCODEREVIEW_LLM_MODEL`

当前实际 provider 口径：

- 协议：OpenAI-compatible
- Provider：DeepSeek API
- URL：`https://api.deepseek.com/chat/completions`
- Model：`deepseek-v4-pro`
- `llm_use_anthropic`：`false`

这意味着：

> 当前 workflow 是按 **OpenAI-compatible provider** 写的，不是 Anthropic protocol。

---

## 七、以后想改哪里

## 7.1 想改模型

只改 GitHub secrets：

1. `OPENCODEREVIEW_LLM_MODEL`

例如：

- 从 `deepseek-v4-pro` 改到别的 DeepSeek 模型；
- 或改到任意兼容的 OpenAI-style model name。

一般**不需要改 workflow 文件**。

## 7.2 想改 provider，但仍是 OpenAI-compatible

例如以后从 DeepSeek 换到：

- OpenAI
- OpenRouter
- Moonshot
- 其他 OpenAI-compatible 服务

通常只需要改：

1. `OPENCODEREVIEW_LLM_URL`
2. `OPENCODEREVIEW_LLM_TOKEN`
3. `OPENCODEREVIEW_LLM_MODEL`

同时保持：

- `llm_use_anthropic: 'false'`

这类切换通常也**不需要改 workflow 结构**。

## 7.3 想切到 Anthropic-compatible provider

如果以后从 OpenAI-compatible 改成 Anthropic-style provider，例如切到 “A 社” 协议模式，则除了改 secrets 外，还要改 workflow。

需要改的是三个仓的：

- `opencode-review.yml`

重点改：

1. 把 `llm_use_anthropic` 从 `'false'` 改成 `'true'`
2. 把 `OPENCODEREVIEW_LLM_URL` 改为 Anthropic-compatible endpoint
3. 更新 token 与 model

也就是说：

> **OpenAI-compatible 之间切换，主要改 secrets；OpenAI-compatible 和 Anthropic-compatible 之间切换，既要改 secrets，也要改 workflow 参数。**

---

## 八、实际修改入口

以后如果要改 review 行为，建议优先按下面顺序找入口：

1. 改 provider / model / token
   - GitHub Organization secrets
   - `OPENCODEREVIEW_LLM_URL`
   - `OPENCODEREVIEW_LLM_TOKEN`
   - `OPENCODEREVIEW_LLM_MODEL`
2. 改 review 策略
   - 三个代码仓各自的 `.github/workflows/opencode-review.yml`
3. 改“是否阻断合并”
   - GitHub 仓库 `main` 分支 protection rule
   - 是否把 `opencode-review` 加入 required checks
4. 改基础构建/测试门禁
   - `fluentwork-ios/.github/workflows/ios-ci.yml`
   - `fluentwork-backend/.github/workflows/backend-ci.yml`
   - `fluentwork-infra/.github/workflows/infra-ci.yml`

建议修改原则：

- **优先改 secrets，后改 workflow，最后再改 branch protection**；
- 这样可以把“provider 切换”和“门禁收紧”拆成两个低耦合动作，回滚也更容易。

---

## 九、如果以后想调整 review 行为

最常改的是这几个点：

## 9.1 调整评论密度

改：

- `route_severity_below`

当前值：

- `medium`

含义：

- `medium` 及以下更倾向进 summary；
- 更高严重度更可能写 inline comment。

如果觉得评论太多，可以把阈值收紧。  
如果觉得评论太少，可以放宽。

## 9.2 调整 summary 行为

改：

- `sticky_summary`
- `incremental`

含义：

1. `sticky_summary: 'true'`
   - PR 中维护一条持续更新的 summary
2. `incremental: 'true'`
   - 对增量改动做更贴近 PR 演进的审查

## 9.3 调整触发时机

改：

- `on.pull_request.types`

当前是：

1. `opened`
2. `synchronize`
3. `reopened`
4. `ready_for_review`

如果以后想加：

- 评论触发重跑
- 手动命令触发

就需要扩展 workflow 事件。

---

## 十、哪些东西不建议随便改

1. 直接把 OpenCodeReview 升成 required check；
2. 一开始就把低严重度全改成 inline；
3. 在没有验证 provider 输出稳定性前频繁切模型；
4. 让本地开发也强依赖 OpenCodeReview，而不是用 `gstack /review` 做快反馈。

---

## 九、推荐维护方式

后续维护建议：

1. **本地开发策略** 主要写在各仓 `CLAUDE.md` / `AGENTS.md`
2. **PR 第二审查策略** 主要维护在各仓 `opencode-review.yml`
3. **跨仓协作原则** 统一回写到 `fluentwork-meta/docs/40_研发流程与协作/`
4. **provider 切换** 优先走“先改 secrets，后做测试 PR 验证”的方式

---

## 十、最终口径

当前 FluentWork 三仓的推荐协作口径是：

> **本地开发用 gstack 做快反馈，GitHub PR 用 OpenCodeReview 做第二审查，provider 优先通过组织级 secrets 管理；只有在协议类型发生变化时，才同时修改 workflow 文件。**
