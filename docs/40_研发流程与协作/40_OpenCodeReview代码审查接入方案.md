# OpenCodeReview 代码审查接入方案

**版本**：V1.0  
**日期**：2026-08  
**定位**：定义 FluentWork 如何把 OpenCodeReview 接入 GitHub PR 审查链路  
**上游依据**：`12_FluentWork-AI协作开源研发与CI-CD方案.md`、`13_FluentWork仓库初始化与CI-CD执行计划.md`  
**状态**：方案阶段，待指令后执行

---

## 一、目标

在 FluentWork 的代码审查链路中加入一层**自动化第二审查**，重点补齐：

1. 人审来不及覆盖的回归风险；
2. AI 生成代码中的幻觉、旧 API、逻辑跳跃、漏测问题；
3. 跨文件一致性与关键目录的额外审查压力。

本方案默认把你说的 `opencodereview` 解释为 **Alibaba OpenCodeReview / open-code-review**，即可在 GitHub PR 中自动发 summary 与 inline comments 的审查工具。

---

## 二、在 FluentWork 中的角色定位

OpenCodeReview 不是主评审，也不是唯一 gate。它在 FluentWork 中的定位是：

> **PR 第二审查层**

推荐的完整链路：

1. 作者自检：本地测试 + PR 模板自查；
2. CI 检查：lint / build / unit test / migration / smoke；
3. **OpenCodeReview**：自动给出 PR 级 findings；
4. 人类 owner 审批：对关键目录做最终判断。

也就是说，它应该和传统 CI、人类评审并列，而不是替代其中任何一层。

---

## 三、为什么适合当前 FluentWork

当前 FluentWork 有几个明显特点：

1. 多仓并行：`meta`、`ios`、`backend`、`infra`；
2. AI 参与实现比例高；
3. 小团队，需要把 review 产能尽量自动化；
4. 后续会有 Go、Swift、YAML、Markdown、Shell 多种材料。

OpenCodeReview 在这个阶段的价值主要有三类：

1. **补 AI 代码特有缺陷**：例如过时 API、上下文错位、结构过度设计；
2. **把 review 结果写回 GitHub PR**：能直接融入现有 GitHub 流程；
3. **适合作为报告层先落地**：先跑起来，再决定哪些级别升级为 gate。

---

## 四、推荐的接入策略

## 4.1 总体策略

采用“**先报告、后收紧**”：

### Phase A：报告模式

- PR 自动触发；
- 产出 sticky summary；
- 允许发 inline comments；
- 不阻断 merge。

### Phase B：半门禁模式

- `backend`、`infra` 关键仓开始将严重问题纳入 required checks；
- 低严重度问题只进 summary，不刷满 inline comments。

### Phase C：关键路径门禁

- 只对关键目录和关键严重度启用阻断；
- `ios` 音频 / 状态机、`backend` 网关 / 会话状态机、`infra` prod deploy 等目录纳入高门槛。

---

## 五、按仓库的落位建议

## 5.1 fluentwork-meta

目标：检查模板、workflow、文档治理脚本。

建议：

1. 先启用 report-only；
2. 关注：
   - workflow 配置错误
   - shell / yaml 风险
   - 文档与流程脚本不一致

不建议：

- 对纯 Markdown 修改频繁刷 inline comments。

## 5.2 fluentwork-backend

目标：作为最早进入半门禁的代码仓。

建议：

1. 首批启用自动 PR review；
2. 重点关注：
   - 接口契约漂移
   - 并发 / 状态机边界
   - 数据删除 / 幂等 / 鉴权路径
   - migration 风险

理由：

- Go 后端更适合先做结构化、规则化 code review；
- 与部署和数据安全关系更紧，第二审查收益最高。

## 5.3 fluentwork-ios

目标：先报告，后对关键目录升门禁。

建议：

1. 初期只对普通 UI / store / service 目录做自动审查；
2. 核心音频 / SpeechSession 状态机维持“AI 报告 + 人类强审”的模式；
3. 等工程骨架稳定后，再增加规则文件。

理由：

- SwiftUI 与状态机代码里，误报成本可能高于前期收益；
- 先让它对非生死线代码发挥作用更稳妥。

## 5.4 fluentwork-infra

目标：较早进入半门禁。

建议：

1. 关注 deploy workflow、环境变量引用、回滚说明、危险权限；
2. 对 prod deploy 工作流变更可配置为必须过 OpenCodeReview summary 检查后才能 merge。

---

## 六、推荐的 GitHub 工作流形态

推荐每个代码仓都放一条独立 workflow，例如：

`/.github/workflows/opencode-review.yml`

建议触发事件：

```yaml
on:
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]
  issue_comment:
    types: [created]
```

推荐策略：

1. PR 打开 / 更新时自动跑一次；
2. 支持手动评论重新触发；
3. 草稿 PR 可以选择跳过；
4. fork PR 默认只做无密钥的轻量模式，或直接跳过。

---

## 七、推荐的接入模式

## 7.1 模式 A：直接用 OpenCodeReview Action

适合：

- 需要 PR inline comments + sticky summary；
- 希望把 OCR 作为专门 code review 工具独立运行。

特点：

1. 对接简单；
2. 输出更贴近“代码审查机器人”；
3. 适合作为 FluentWork 的标准第二审查层。

推荐度：**最高**

## 7.2 模式 B：用 OpenCode GitHub Action 做 `/oc review`

适合：

- 想把 review、triage、fix issue、按评论触发实现都放进一套工具；
- 希望后续让 bot 接 issue 或 PR comment 自动工作。

特点：

1. 更通用，不只做 review；
2. 更像“评论驱动的 Agent”；
3. 比单纯 code review 更重。

推荐度：**中**

建议：

- 当前先用模式 A 做标准 PR 审查；
- 后续若要做 `/oc` 评论触发修复，再单独引入模式 B。

---

## 八、推荐的初始参数

以下是 FluentWork 第一阶段的建议值：

1. 语言：`中文`
2. 输出方式：
   - sticky summary = 开
   - incremental = 开
   - 低严重度路由到 summary
3. 严重度策略：
   - `critical` / `high` 允许 inline
   - `medium` / `low` 优先进 summary
4. 上传 artifacts：开
5. 初期不阻断 merge，只留结果供人审参考

---

## 九、推荐的 rollout 节奏

## 9.1 第一步：先在 backend 落地

原因：

1. 后端更适合规则化审查；
2. 最早能看到有价值的风险提示；
3. 误报对开发节奏影响相对更可控。

## 9.2 第二步：落地 infra

原因：

1. workflow / deploy / secrets 风险高；
2. 自动化审查非常值得。

## 9.3 第三步：落地 ios

原因：

1. SwiftUI / 状态机需要先看误报率；
2. 建议先从非核心目录开始。

## 9.4 第四步：落地 meta

原因：

1. 价值主要在 workflow、脚本和模板变更；
2. 纯文档改动不应让 review 机器人过度刷屏。

---

## 十、与现有 Review Gate 的关系

FluentWork 目标链路建议如下：

```text
本地自检
  -> GitHub CI
  -> OpenCodeReview
  -> 人类 Owner 审批
  -> Merge
```

关键点：

1. OpenCodeReview 不是 merge owner；
2. 人类 owner 仍保留最终决定权；
3. 关键目录必须保留 CODEOWNERS 强审；
4. OpenCodeReview 的结论应该进入 PR 模板中的“风险点 / 第二审查结果”字段。

---

## 十一、推荐的 PR 模板联动字段

建议在 PR 模板中增加：

1. `第二审查 AI 是否已运行`
2. `第二审查 AI 主要 findings`
3. `是否已处理 high/critical findings`
4. `是否申请人工豁免`

这样可以把自动审查结果显式带入人审流程。

---

## 十二、建议的组织级 secrets

若采用 OpenCodeReview Action，建议优先用组织级 secrets：

1. `OPENCODEREVIEW_LLM_URL`
2. `OPENCODEREVIEW_LLM_TOKEN`
3. `OPENCODEREVIEW_LLM_MODEL`
4. `OPENCODEREVIEW_USE_ANTHROPIC`

如果走 OpenCode GitHub Action，则按 provider 放：

1. `ANTHROPIC_API_KEY`
2. 或 `OPENAI_API_KEY`
3. 或其他 provider 对应密钥

原则：

- 优先组织级统一管理；
- repo 级只保留特殊覆盖项；
- fork PR 不暴露敏感密钥。

---

## 十三、推荐的执行顺序

当收到执行指令后，建议按以下顺序：

1. 在 `fluentwork-backend` 建 `opencode-review.yml`；
2. 先用 report-only 参数跑通 3 到 5 个 PR；
3. 记录误报、重复评论、评论噪音；
4. 再决定是否把 high/critical findings 升为门禁；
5. 复制方案到 `infra`；
6. 再按目录分层方式接入 `ios`。

---

## 十四、明确不建议的做法

1. 一上来就在 4 个仓全部设为 required check；
2. 让机器人直接替代人类审批；
3. 在纯文档仓对所有低严重度问题刷 inline comments；
4. 不做目录分层，就对 iOS 核心状态机代码直接强阻断；
5. 让 review 工具和实现工具共用同一套“自动修复即提交”权限。

---

## 十五、最终建议

对 FluentWork 当前阶段，最稳妥的做法是：

1. **采用 Alibaba OpenCodeReview 作为标准第二审查层**；
2. **先在 `backend` 仓以 report-only 模式落地**；
3. **把 `infra` 作为第二个接入仓**；
4. **iOS 仓先只覆盖普通目录，核心语音链路仍由人类强审兜底**；
5. **保留人类 owner 作为最终 gate**。

如果后续你还想把 issue comment / PR comment 驱动的 Agent 能力也接进来，再单独加一层 OpenCode GitHub Action 的 `/oc` 触发流程，而不是一开始把两件事揉在一起。

---

## 十六、参考来源

1. OpenCode GitHub 官方文档：<https://opencode.ai/docs/github/>
2. Alibaba Open Code Review Action 配置：<https://github.com/alibaba/open-code-review>
