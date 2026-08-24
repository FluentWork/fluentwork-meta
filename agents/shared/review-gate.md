# Review Gate

## 目标

统一四个仓库的代码审查门禁，确保 AI 参与开发时仍保持可控质量。

## 审查链路

```text
作者自检
  -> CI checks
  -> OpenCodeReview / 第二审查 AI
  -> 人类 owner 审批
  -> merge
```

## 规则

1. 机器审查不能替代人类 owner；
2. 第二审查 AI 先作为报告层，再逐步升级为门禁；
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
3. 区分 bug、risk、missing test、docs drift；
4. 明确哪些必须修、哪些可接受豁免。
