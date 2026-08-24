# Matt Pocock Skills

## 目标

定义 FluentWork 如何引入和使用 Matt Pocock 风格 skills，而不是把外部 skills 直接当成项目制度。

## 使用原则

1. 外部 skills 是辅助能力，不是项目真源；
2. 是否采用某个 skill，由 `fluentwork-meta` 的共享规则决定；
3. 各仓只接入自己真正需要的部分；
4. 任何外部 skill 都不能覆盖 FluentWork 自己的治理边界。

## 推荐用法

1. 在 `meta` 中记录允许使用的 skill 列表；
2. 在各仓 `CLAUDE.md` / `AGENTS.md` 中只写“可用哪些、什么时候用”；
3. repo-specific 高风险区域仍以本仓规则为准；
4. skills 的升级与替换要能单独审查。

## 不建议的做法

1. 四个仓都复制一整套外部 skills 真源；
2. 让外部 skill 文本直接成为仓库唯一规则；
3. 在 CI 中动态安装、加载和运行整套外部 skills；
4. 在没有审查的情况下替换共享 templates。

## 当前建议

当前阶段先做：

1. 在 `meta` 维护外部 skills 使用边界；
2. 在各仓入口文件中声明可使用 Matt Pocock 风格 skills；
3. 后续若出现 repo-specific 适配需求，再按仓单独补充。
