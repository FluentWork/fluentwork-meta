# Review Gate

## 目标

统一四个仓库的代码审查门禁，确保 AI 参与开发时仍保持可控质量。

## 审查链路

```text
作者自检
  -> 本地 OpenCodeReview（git pre-commit）
  -> CI checks（build / test / lint）
  -> 人类 owner 审批（高风险目录）
  -> merge
```

可选：本地再用 `gstack /review` 做更深的结构审查（不替代 OCR 门禁）。

## OpenCodeReview 合入规则（强制）

以本地 OpenCodeReview（`ocr review` + `ocr-fail-on-high.sh`）的 **severity** 为准：

1. **存在任意 `high` / `critical` finding → 禁止提交**，必须先修改，直到该轮 review 无 `high`/`critical`；
2. **没有 `high`/`critical` → 可以提交**（`medium` / `low` / nit 不阻塞，可记 follow-up）；
3. 豁免仅允许人类 owner 书面说明原因（commit/PR 说明），且不得用于安全类 `high`；紧急旁路使用 `SKIP_OCR=1` 时必须同样留痕；
4. 以最新一次成功完成的本地 review run 为准；修完代码后须再跑一轮再判。

判定口径：

- `high` / `critical`：缺陷、数据损坏、鉴权/票据失效、明显安全问题等必须修；
- `medium` / `low`：可提交后跟进，除非人类 owner 明确要求本变更处理。

## 本地落地（三仓统一）

各代码仓（`fluentwork-ios` / `fluentwork-backend` / `fluentwork-infra`）：

1. 一次性启用 hook：`./scripts/setup-git-hooks.sh`（iOS 为 `./Scripts/setup-git-hooks.sh`）
2. pre-commit 调用 `ocr-local-review.sh` → `ocr-fail-on-high.sh`
3. GitHub **不再**运行 `opencode-review` workflow

## 规则

1. 机器审查不能替代人类 owner；
2. 本地 OCR 对 `high`/`critical` 具有提交否决效力（见上节）；非 `high` 仍可作为报告层；
3. 关键目录必须保留人类强审；
4. 对低价值噪音评论应收敛到 summary，不让 inline comments 淹没协作记录。

## 高风险目录示例

1. iOS：AudioEngine、SpeechSession、调试桥接；
2. Backend：voice-gateway、session state machine、migration；
3. Infra：prod deploy、secrets、回滚脚本；
4. Meta：治理模板、workflow、规范文件。

## 审查输出要求

1. findings first；
2. 给文件和行号；
3. 区分 bug、risk、missing test、docs drift，并标注 severity（`high` / `medium` / `low`）；
4. 明确哪些必须修（至少全部 `high`/`critical`）、哪些可接受豁免。
