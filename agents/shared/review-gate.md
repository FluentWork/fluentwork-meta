# Review Gate

## 目标

统一四个仓库的代码审查门禁，确保 AI 参与开发时仍保持可控质量。

## 审查链路

```text
作者自检
  -> gstack /review（合入前结构审查；本地 pre-commit 要求 attestation）
  -> CI checks（build / test / lint）
  -> 人类 owner 审批（高风险目录）
  -> merge
```

## 当前策略（2026-08）

1. **主审查**：使用 **gstack `/review`** 对相对 base 的 diff 做合入前审查；
2. **本地 pre-commit**：三代码仓 `.githooks/pre-commit` 调用 `gstack-review-gate.sh`，要求提交前已跑 gstack `/review` 并以 `GSTACK_REVIEWED=1` 声明；
3. GitHub **不**运行任何 code-review / `opencode-review` workflow；CI 只做 build / test / lint / config；
4. OpenCodeReview（OCR）脚本与 `.opencodereview/` 可保留作可选手工工具，**默认不参与门禁**。

## gstack `/review` 合入与 commit 规则

1. 提交前应跑一轮 gstack `/review`（或等价 Cursor skill `/review`）；
2. 因技能无法在 bash 中自动执行，pre-commit 采用 **attestation 门禁**：
   - 正常：`GSTACK_REVIEWED=1 git commit ...`
   - 紧急旁路：`SKIP_GSTACK_REVIEW=1 git commit ...`（须在 commit/PR 说明原因）；
3. 对 **高风险目录** 与明确缺陷类发现：合入前必须处理或由人类 owner 书面豁免；
4. 机器审查不能替代人类 owner；高风险目录仍须 owner 强审；
5. 以最新一轮 review 结论为准；修完代码后再跑一轮。

判定口径（与既有 severity 习惯对齐）：

- 必须修：缺陷、数据损坏、鉴权/票据失效、明显安全问题等；
- 可跟进：风格、文档漂移、非阻断测试缺口（除非 owner 要求本 PR 处理）。

## 本地落地（三仓）

各代码仓（`fluentwork-ios` / `fluentwork-backend` / `fluentwork-infra`）：

1. `./scripts/setup-git-hooks.sh`（iOS：`./Scripts/setup-git-hooks.sh`）启用 `.githooks`；
2. pre-commit → `gstack-review-gate.sh`（要求 `GSTACK_REVIEWED=1` 或 `SKIP_GSTACK_REVIEW=1`）；
3. OCR 相关脚本不再由 pre-commit 调用。

## 规则

1. 机器审查不能替代人类 owner；
2. 合入前以 gstack `/review` + CI + owner 为准；
3. 高风险目录必须保留人类强审；
4. 低价值噪音应收敛，避免淹没协作记录；
5. CI 不做 interactive skills / code-review 执行主体。

## 高风险目录示例

1. iOS：AudioEngine、SpeechSession、调试桥接；
2. Backend：voice-gateway、session state machine、migration；
3. Infra：prod deploy、secrets、回滚脚本；
4. Meta：治理模板、workflow、规范文件。

## 审查输出要求

1. findings first；
2. 给文件和行号；
3. 区分 bug、risk、missing test、docs drift，并标注 severity（`high` / `medium` / `low`）；
4. 明确哪些必须修、哪些可接受豁免。
