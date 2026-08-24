# gstack 源码能力分析与 FluentWork 落位建议

**版本**：V1.0  
**日期**：2026-08  
**定位**：基于本地 `~/Developments/gstack` 源码，分析哪些能力适合在 FluentWork 当前阶段复用  
**来源范围**：`gstack/README.md`、`.github/workflows/*`、`.github/PULL_REQUEST_TEMPLATE.md`、`ios-qa/SKILL.md`、`review/SKILL.md` 等  
**状态**：分析阶段，待指令后执行

---

## 一、结论先行

`gstack` 对 FluentWork **有明显可复用价值**，但不建议“整仓引入”或“全套照搬”。

更合适的策略是分三层吸收：

1. **直接借鉴流程与模板**：PR 模板、CI 分层、质量门禁、发布链路；
2. **作为团队本地工作流工具使用**：`/review`、`/qa`、`/ship`、`/setup-deploy`、`/ios-qa`；
3. **谨慎评估后再落到仓库制度**：例如 team mode、自动更新、复杂证据账本、完整 gstack skill 体系。

对 FluentWork 当前阶段，最值得优先吸收的是：

1. review / ship / qa 的工作流设计；
2. GitHub Actions 的最小权限与 required checks 思路；
3. iOS 真机 / 真设备 QA 的方法论；
4. PR 模板里的“证据优先”理念；
5. secret scan / workflow lint / dependency review 这类质量门禁。

---

## 二、为什么值得分析

FluentWork 当前处于：

1. 多仓初始化期；
2. AI 参与开发比例高；
3. iOS + backend + infra 并行；
4. 后续需要更强的 review、QA、release discipline。

而 `gstack` 的核心长处正好集中在：

1. **把 AI 协作流程编排成固定链路**；
2. **把 review / QA / ship 串成一条标准流程**；
3. **给 GitHub PR 和 CI/CD 加上实际可运行的门禁**；
4. **对 iOS 真机 QA 有专门能力**。

因此它不是“直接拿来当依赖包”，而是一个非常好的**方法库 + 模板库 + 本地工作流工具库**。

---

## 三、分析结果总览

## 3.1 适合直接复用

1. GitHub PR 模板结构；
2. CI 工作流设计思路；
3. `review` / `qa` / `ship` 的流程切分；
4. `ios-qa` 对真机验证的思路；
5. `setup-deploy` 对部署接入的时序；
6. workflow lint、secret scan、dependency review 的门禁思路。

## 3.2 适合方法借鉴，不建议直接搬代码

1. 全套 skill 安装与 team mode；
2. 连续 checkpoint / telemetry / local ledger；
3. 复杂的 multi-skill orchestration；
4. 大量针对 gstack 自身仓库结构的脚本。

## 3.3 当前不建议落地

1. 把 gstack 整仓 vendoring 进 FluentWork；
2. 在仓库里强制所有成员使用 gstack；
3. 在尚未稳定的阶段引入完整 `/ship`、`/land-and-deploy` 自动化闭环；
4. 依赖其复杂本地状态目录作为团队协作真源。

---

## 四、逐项能力分析

## 4.1 Review 工作流

### 观察

从 `README.md`、`review/SKILL.md` 可以确认，gstack 把 `/review` 作为“预合入审查”核心环节，强调：

1. 在 landing 前做 diff review；
2. 抓结构性问题，而不是只看表面风格；
3. 把 review 放进固定流水线，而不是零散请求。

### 对 FluentWork 的价值

非常高。

适用位置：

1. `fluentwork-backend` 的 PR 审查流程；
2. `fluentwork-ios` 的普通功能 PR 自检；
3. `fluentwork-meta` 的 workflow / 模板变更评审。

### 推荐落位

- **制度层**：在 `docs/40_研发流程与协作/` 定义 review 流程；
- **工具层**：把 gstack `/review` 作为 owner 的本地辅助审查工具；
- **平台层**：与 OpenCodeReview 并行，形成“本地 review + GitHub 第二审查”双层结构。

### 不建议

- 直接把 gstack `/review` 当作 GitHub 必过 check。

原因：

- 它更适合人驱动的本地工作流，不是轻量、标准化的 CI bot。

---

## 4.2 QA / 浏览器验证能力

### 观察

`README.md` 中 `/qa`、`/qa-only`、`/browse` 是成熟主线能力，强调：

1. 打开真实浏览器；
2. 进行活体验证；
3. 修 bug 后补回归；
4. QA 结果与计划、review、ship 串起来。

### 对 FluentWork 的价值

中高。

当前直接价值主要在：

1. 后台 / 管理台 / 文档站若上线 Web 面时可直接使用；
2. 后续若 `backend` / `infra` 有控制台或 dashboard，可做 staging QA；
3. 对 GitHub Pages 文档站也可做轻量验证。

### 推荐落位

- 当前先不写入仓库制度；
- 作为负责人本地 QA 工具储备；
- 等到有真实 Web 面再正式纳入测试流程。

---

## 4.3 iOS 真机 QA 能力

### 观察

`ios-qa/SKILL.md` 与 `README.md` 表明 gstack 对 iOS 有专门支持：

1. 真机连接；
2. 通过嵌入式 `StateServer` 驱动 app；
3. 读取 Swift 源码辅助理解界面；
4. 可做 USB / 远程 tailnet 模式。

### 对 FluentWork 的价值

非常高。

这是 gstack 对当前项目最特别、最贴近的能力之一，因为 FluentWork 本身就是 iOS-first。

### 推荐落位

1. **短期**：作为 `fluentwork-ios` 的增强 QA 研究方向；
2. **中期**：在 `docs/50_测试与验收/` 新增“iOS 真机 QA 方案”文档；
3. **工程位**：当 iOS 工程骨架完成且进入可交互阶段后，再评估引入 DebugBridge / StateServer 类方案。

### 注意点

不建议在仓库初始化阶段立刻接入。

原因：

1. 需要 app 内嵌桥接；
2. 对工程结构有侵入；
3. 应在 UI 流程具备可测试性之后再做。

---

## 4.4 Ship / Land-and-deploy / Setup-deploy

### 观察

`README.md` 和 `setup-deploy/SKILL.md` 说明 gstack 很重视：

1. 先 review，再 ship；
2. deploy 之前要有明确环境与平台判断；
3. 发布后还有 canary / verification。

### 对 FluentWork 的价值

高。

因为 FluentWork 正在搭多仓与 CI/CD，最缺的就是“发布路径的固定顺序”。

### 推荐落位

1. **meta 仓**：写成发布 runbook 和 checklist；
2. **infra 仓**：建立 staging / prod deploy skeleton；
3. **本地工具**：负责人可用 gstack `/setup-deploy`、`/ship` 作为执行辅助。

### 不建议

- 现在就把 gstack `/land-and-deploy` 全量自动化到生产环境。

原因：

- 当前项目还没到可以一键合并后一键部署的阶段。

---

## 4.5 PR 模板与证据优先

### 观察

`gstack/.github/PULL_REQUEST_TEMPLATE.md` 的核心不是“描述改了什么”，而是：

1. 为什么改；
2. 现场证据是什么；
3. 哪些范围改了；
4. 哪些没测；
5. 证明有人实际验证过。

### 对 FluentWork 的价值

非常高。

当前 FluentWork 本身就强调 AI 产出必须有人类门禁，因此“证据优先”比普通 PR 模板更适合这个项目。

### 推荐落位

- 直接吸收其结构，改写为 FluentWork 自己的 PR 模板；
- 放到各代码仓和 `meta` 仓中。

建议保留的字段：

1. `Why`
2. `上游 issue / 文档`
3. `Live evidence / 测试结果`
4. `风险点`
5. `未覆盖项`

不建议照搬的字段：

- `GSTACK PR` 活体验证截图这类强品牌化要求。

---

## 4.6 GitHub Actions 设计

### 观察

从 gstack 的 `.github/workflows/` 能看到几类很适合借鉴的原则：

1. **最小权限**：`permissions` 尽量收紧；
2. **required checks 早建立**；
3. **workflow lint 单独存在**；
4. **quality gate 单独存在**；
5. **free tests 与高成本 eval 分层**。

### 对 FluentWork 的价值

非常高。

尤其适合当前初始化阶段的仓库：

1. `actionlint.yml`
2. `quality-gate.yml`
3. `dependency-review.yml`
4. `osv-scanner.yml`

### 推荐落位

#### meta

- `actionlint`
- `markdown/link checks`

#### backend

- `go ci`
- `quality gate`
- `dependency review`

#### infra

- `workflow lint`
- `deploy config check`
- `secret scan`

#### ios

- `build/test`
- 后续再接更重的模拟器与真机验证

---

## 4.7 Secret scan / quality gate

### 观察

`quality-gate.yml` 体现的是一套很实用的组合：

1. diff 级凭证扫描；
2. 依赖审计；
3. shell 边界检查。

### 对 FluentWork 的价值

高。

尤其适合：

1. `infra` 仓；
2. `backend` 仓；
3. `meta` 仓中的 workflow / shell 脚本。

### 推荐落位

- 作为第一批 required checks 候选；
- 优先放在 `infra` 与 `backend`；
- `ios` 可先放轻量版本。

---

## 五、按 FluentWork 仓库的落位建议

## 5.1 fluentwork-meta

优先吸收：

1. PR 模板结构；
2. actionlint / workflow lint；
3. 文档发布与流程检查思路；
4. review 流程文档化方法。

不建议：

1. 把整套 gstack skill 放进仓库；
2. 使用复杂本地状态账本作为团队依赖。

## 5.2 fluentwork-ios

优先吸收：

1. `/review` 作为本地辅助审查；
2. iOS 真机 QA 方法论；
3. 后续研究 `ios-qa` 型调试桥接。

建议时序：

1. 先有可运行工程；
2. 再有基本 UI 流；
3. 再考虑真机 QA 桥接。

## 5.3 fluentwork-backend

优先吸收：

1. review / ship 的流程 discipline；
2. quality gate；
3. dependency / security checks；
4. PR 模板中的 live evidence 结构。

这是最适合最早吸收 gstack 方法论的代码仓。

## 5.4 fluentwork-infra

优先吸收：

1. setup-deploy 的时序思路；
2. workflow lint；
3. deploy gate；
4. secrets / shell 边界检查。

---

## 六、推荐的落地顺序

### 第一批：立刻吸收

1. PR 模板结构
2. workflow lint
3. quality gate
4. review / ship 的流程概念

### 第二批：代码仓稳定后吸收

1. `/review` 作为本地工作流
2. `/setup-deploy` 作为 infra 设计辅助
3. `/qa` 作为 Web 面验证工具

### 第三批：iOS 工程成型后再吸收

1. `ios-qa`
2. 真机调试桥接
3. 更强的 release / canary 流程

---

## 七、明确不建议的做法

1. 把 gstack 当成 FluentWork 的运行时依赖；
2. 当前就要求所有仓、所有成员都接 team mode；
3. 把其复杂本地状态目录和学习记录机制当作协作真源；
4. 在项目骨架未稳时直接接入侵入式 iOS QA 桥接；
5. 一次性把 `/review`、`/qa`、`/ship`、`/land-and-deploy` 全部制度化。

---

## 八、最终建议

对 FluentWork 当前阶段，推荐采用下面这个判断：

> **把 gstack 当作“高价值外部方法库与本地工作流工具”，而不是要被完整集成进仓库的框架。**

最值得现在就落地的内容：

1. 借鉴 PR 模板；
2. 借鉴 CI / quality gate 设计；
3. 把 `/review`、`/setup-deploy` 视为负责人本地工作流工具；
4. 将 `ios-qa` 列为 `fluentwork-ios` 中期增强项；
5. 在 `meta` 仓把这些落为自己的治理文档和执行规范。

---

## 九、推荐的后续文档落位

基于本分析，后续建议补 2 份文档：

1. `docs/50_测试与验收/51_iOS真机QA与调试桥接方案.md`
2. `docs/40_研发流程与协作/41_PR模板与Review Gate规范.md`
