# Review Gate

## 目标

统一四个仓库的代码审查门禁，确保 AI 参与开发时仍保持可控质量。

## 审查链路

```text
作者自检
  -> CI checks
  -> OpenCodeReview / 第二审查 AI
  -> 人类 owner 审批（高风险目录）
  -> merge
```

## OpenCodeReview 合入规则（强制）

以 OpenCodeReview（或同等第二审查 AI）的 **severity** 为准：

1. **存在任意 `high` finding → 禁止合并**，必须先修改并推送，直到该轮 review 无 `high`；
2. **没有 `high` → 可以合并**（`medium` / `low` / nit 不阻塞合入，可记 follow-up）；
3. 豁免仅允许人类 owner 书面说明原因（PR 评论），且不得用于安全类 `high`；
4. 合入前以最新一次成功完成的 review run 为准；修完代码后须等新一轮 review 结束再判。

判定口径：

- `high`：缺陷、数据损坏、鉴权/票据失效、明显安全问题等必须修；
- `medium` / `low`：可合并后跟进，除非人类 owner 明确要求本 PR 处理。

## 规则

1. 机器审查不能替代人类 owner；
2. 第二审查 AI 对 `high` 具有合入否决效力（见上节）；非 `high` 仍可作为报告层；
3. 关键目录必须保留人类强审；
4. 对低价值噪音评论应收敛到 summary，不让 inline comments 淹没 PR。

## 高风险目录示例

1. iOS：AudioEngine、SpeechSession、调试桥接；
2. Backend：voice-gateway、session state machine、migration；
3. Infra：prod deploy、secrets、回滚脚本；
4. Meta：治理模板、workflow、规范文件。

## 审查输出要求

1. findings first；
2. 给文件和行号；
3. 区分 bug、risk、missing test、docs drift，并标注 severity（`high` / `medium` / `low`）；
4. 明确哪些必须修（至少全部 `high`）、哪些可接受豁免。
