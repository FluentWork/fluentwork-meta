# FluentWork Agent 与 Skills 治理策略

**版本**：V1.0  
**日期**：2026-08  
**定位**：判断 FluentWork 多仓之间是否需要独立的共享 skills 仓，并定义当前阶段的治理策略  
**上游依据**：`12_FluentWork-AI协作开源研发与CI-CD方案.md`、`13_FluentWork仓库初始化与CI-CD执行计划.md`  
**状态**：决策已定，可进入后续落地

---

## 一、结论

### 1.1 是否存在多仓共用的 skills

**存在。**

四个仓库后续会共用一层 agent / skills 规则，主要包括：

1. `CLAUDE.md` / `AGENTS.md` 的基础结构；
2. Git / Issue / PR / Review Gate 约束；
3. AI 工具角色分工；
4. Matt Pocock 风格 skills 的使用约定；
5. 通用的安全边界、禁止事项与代码审查要求；
6. 文档更新、测试补齐、最小差异修改等团队共识。

### 1.2 是否需要现在单独创建一个共享 skills 仓

**当前阶段不需要。**

当前 FluentWork 只有 4 个同组织仓库，这些共享规则又强依赖项目治理、技术方案和研发流程文档。此时再拆一个 `fluentwork-skills` 或类似仓库，会带来额外复杂度，但不会显著降低维护成本。

### 1.3 当前推荐策略

> **不新建独立共享 skills 仓。以 `fluentwork-meta` 作为共享真源仓，各项目仓单独创建自己的 `CLAUDE.md` / `AGENTS.md` / repo-specific skills 入口。**

---

## 二、为什么“有共享内容”不等于“要拆共享仓”

这两个判断要分开看。

### 2.1 的确共享的部分

以下内容跨 `meta` / `ios` / `backend` / `infra` 高概率一致：

1. agent 文件命名与入口约定；
2. 使用哪些公共 skills；
3. 代码评审和提交纪律；
4. AI 产出必须补测试、补文档、经过人类门禁；
5. 对 destructive git / secrets / deploy 的通用限制；
6. 对 OpenCodeReview、GitHub Actions、CODEOWNERS 的协作约定。

### 2.2 仍然明显分仓的部分

以下内容必须留在各仓本地：

1. `fluentwork-ios` 的 SwiftUI / TGReduxKit / AudioEngine / SpeechSession 规则；
2. `fluentwork-backend` 的 Go / migration / gateway / 幂等 / 鉴权规则；
3. `fluentwork-infra` 的 deploy / secrets / workflow / rollback 规则；
4. `fluentwork-meta` 的文档治理、编号、规范更新规则。

也就是说，**共享层存在，但仓库特有层同样很重**。当前阶段把共享层单独拆仓，收益还不够覆盖管理成本。

---

## 三、为什么当前不建议拆共享 skills 仓

## 3.1 管理成本会上升

如果现在新建一个共享仓，至少会新增这些问题：

1. 版本如何同步到 4 个仓库；
2. `CLAUDE.md` / `AGENTS.md` 是复制、引用还是桥接；
3. CI 如何校验 4 个仓引用的是同一版本；
4. 当项目治理文档更新时，要改 `meta` 还是改 skills 仓；
5. 谁是共享仓 owner，谁能改所有仓的 agent 行为。

对当前规模来说，这些成本偏早。

## 3.2 共享规则强依赖项目治理文档

当前共享规则不是一个通用开源工具包，而是**FluentWork 项目自己的开发制度**。  
这些制度天然应该和 PRD、技术方案、执行计划放在同一个治理真源里。

所以更合理的真源位置是：

- `fluentwork-meta`

而不是再拆一个与治理文档平级的新仓。

## 3.3 现在还没有跨项目复用压力

如果未来这些 skills 需要被**多个独立项目**复用，那才是拆共享仓的强信号。  
但当前只是 FluentWork 组织下的 4 个仓，仍属于一个项目的多仓结构，还不到那一步。

---

## 四、当前阶段的推荐结构

## 4.1 真源放在 fluentwork-meta

推荐在 `fluentwork-meta` 中建立：

```text
agents/
├── shared/
│   ├── ai-collaboration.md
│   ├── git-and-pr-rules.md
│   ├── review-gate.md
│   ├── skills-policy.md
│   └── matt-pocock-skills.md
└── templates/
    ├── CLAUDE.md.template
    └── AGENTS.md.template
```

这里放的是：

1. 共享规则真源；
2. 各仓 `CLAUDE.md` / `AGENTS.md` 模板；
3. Matt Pocock skills 的引入原则与使用边界。

## 4.2 各仓单独持有入口文件

每个仓库都应各自存在：

1. `CLAUDE.md`
2. `AGENTS.md`

这些文件不应只靠一个中央文件替代。

原因：

1. agent 进入仓库时，首先读取的是当前仓根目录入口；
2. 每个仓都需要本仓特有规则；
3. 各仓独立 clone / 独立开发时不能依赖另一个仓才能工作。

## 4.3 各仓文件应当“薄”

各仓自己的 `CLAUDE.md` / `AGENTS.md` 应尽量只写三类内容：

1. 本仓定位；
2. 本仓特有禁区与高风险区域；
3. 指向 `fluentwork-meta` 中共享规则真源的路径说明。

换句话说：

- **共享规则集中**
- **仓库入口分散**

---

## 五、Matt Pocock skills 应该怎么放

## 5.1 当前建议

Matt Pocock 风格的 skills 如果要引入，建议分两层：

### 共享层

在 `fluentwork-meta/agents/shared/` 中定义：

1. 哪些 skills 允许全仓通用；
2. 这些 skills 的使用边界；
3. 它们与 FluentWork 自己的治理规则如何衔接；
4. 哪些场景禁止直接依赖外部默认行为。

### 仓库层

在各仓只保留：

1. 本仓实际要用到的 bridge / 入口说明；
2. 本仓额外约束；
3. 本仓特有 skills。

## 5.2 不建议的做法

1. 四个仓都各自复制一份完整 Matt Pocock skills 真源；
2. 共享规则散落在四个仓中手工维护；
3. 让某个代码仓充当其他三个仓的 skills 真源。

---

## 六、CI/CD 中要不要有 skills 加载系统

**当前阶段不需要，也不建议。**

CI/CD 更适合验证**结果与配置**，而不是运行一套交互式 skills 系统。

### 6.1 CI 该做什么

CI 中建议做：

1. 检查 `CLAUDE.md` / `AGENTS.md` 是否存在；
2. 检查是否基于最新模板；
3. 检查共享规则引用是否正确；
4. 检查高风险目录的 owner / workflow / review gate 是否生效。

### 6.2 CI 不该做什么

CI 中不建议做：

1. 动态加载整套 skills runtime；
2. 依赖交互式 agent 执行流程；
3. 让 CI 成为 skills 的主运行环境。

原因：

1. 不稳定；
2. 难复现；
3. 成本高；
4. 不适合作为 required checks。

结论：

> **skills 用于开发时，CI 用于校验配置与产出。**

---

## 七、何时才值得拆一个共享 skills 仓

只有在下面条件同时出现至少 2 到 3 个时，才建议重新评估：

1. 除 FluentWork 外，还有别的独立项目要复用同一套 skills；
2. 共享 skills 的更新频率明显高于项目治理文档；
3. 需要独立版本发布、变更日志和升级策略；
4. 共享 skills 维护者与项目仓维护者开始分离；
5. 各仓对模板同步已经成为明显负担。

在那之前，保持 `meta` 为真源更合适。

---

## 八、当前决策

当前正式决策如下：

1. **不创建独立的共享 skills 仓；**
2. **共享真源放在 `fluentwork-meta`；**
3. **四个仓库各自创建自己的 `CLAUDE.md` / `AGENTS.md`；**
4. **共享模板与共享 rules 从 `meta` 生成或同步到各仓；**
5. **CI 只做 agent 配置一致性检查，不加载完整 skills 系统。**

---

## 九、下一步建议

基于本决策，后续应落地以下内容：

1. 在 `fluentwork-meta` 新建 `agents/shared/` 与 `agents/templates/`；
2. 起草四个仓库的 `CLAUDE.md` 模板；
3. 起草四个仓库的 `AGENTS.md` 模板；
4. 增加一条 CI 检查，校验各仓是否存在这些入口文件；
5. 单独整理 Matt Pocock skills 的接入清单与边界说明。
