# 36 FluentWork 可观测性与事件 Schema 设计草案

本文件已迁移到 `fluentwork-infra`，`meta` 不再作为该类 schema 的 source of truth。

当前主文档路径：

- `fluentwork-infra/docs/observability/00_FluentWork可观测性与事件Schema设计.md`
- `fluentwork-infra/docs/observability/01_共享Schema实现设计.md`

迁移原因：

1. `infra` 仓已存在，可承接跨仓 contract / schema 治理
2. 这类定义后续不仅覆盖语音链路，也会继续扩展到购买、订阅、review、worker、网关等事件
3. `meta` 仍保留产品与技术设计文档，但不再持有 schema 主定义

当前约定：

1. 文档说明在 `infra/docs/observability/`
2. 业务/分析 schema 在 `infra/schemas/events/`
3. transport schema 在 `infra/schemas/transport/`
4. 当前已落地 canonical 文件包括：
   - `infra/schemas/transport/wss-control-frames-v1.json`
   - `infra/schemas/events/speech-observability-events-v1.json`
