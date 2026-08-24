# FluentWork 仓库初始化与 CI/CD 执行计划

**版本**：V1.0  
**日期**：2026-08  
**定位**：把 `12_FluentWork-AI协作开源研发与CI-CD方案.md` 细化为可执行的初始化计划  
**上游依据**：`10_FluentWork项目启动书.md`、`11_FluentWork团队分工文档.md`、`12_FluentWork-AI协作开源研发与CI-CD方案.md`、`14_FluentWork-Agent与Skills治理策略.md`
**状态**：计划阶段，待指令后执行

---

## 一、目标

本计划只解决两件事：

1. 将 `fluentwork-meta`、`fluentwork-ios`、`fluentwork-backend`、`fluentwork-infra` 4 个仓库初始化到可协作状态；
2. 为 4 个仓库建立第一版 CI/CD 骨架与 GitHub 治理骨架。

本计划**不包含**业务功能开发，不启动 iOS / backend 具体编码，只做“能稳定开始开发”的基础设施与协作面初始化。

---

## 二、现状判断

### 2.1 已完成

1. GitHub 组织 `FluentWork` 已建立；
2. 4 个远端仓库已创建；
3. 当前本地仓库已切到 `fluentwork-meta` 远端；
4. `fluentwork-meta` 文档已完成第一轮分类归档与第二轮编号整理。

### 2.2 当前缺口

1. 组织级 team / branch protection / Projects / secrets 尚未形成明确落地清单；
2. 4 仓库缺统一的 Issue / PR / CODEOWNERS / workflow skeleton；
3. `ios`、`backend`、`infra` 仓还未建立工程骨架；
4. 代码评审、发布门禁、环境审批尚未形成一套串起来的流水线；
5. 4 仓库的 `CLAUDE.md` / `AGENTS.md` / shared skills 真源尚未建立。

---

## 三、执行原则

1. **先治理后编码**：先把仓库规则、模板、门禁搭起来，再进业务实现。
2. **先 skeleton 后强化**：先保证每个仓库都有最小可运行 workflow，再逐步加重测试与发布门禁。
3. **分仓独立初始化**：每个仓库有自己的 CI，但共用一套命名、分支、审查和发布规则。
4. **默认最小权限**：Actions、Secrets、环境审批都按最小权限配置。
5. **报告优先，阻断后置**：第一阶段先让检查“跑起来并可见”，第二阶段再将关键项升为 required checks。
6. **共享真源留在 meta**：当前不新增独立共享 skills 仓，agent / skills 的共享规则统一落在 `fluentwork-meta`。

---

## 四、总体执行顺序

按下面顺序推进，不并行打乱：

1. `fluentwork-meta` 治理仓补齐模板与治理入口；
2. GitHub 组织层设置；
3. 在 `meta` 建 shared agent / skills 真源与模板；
4. `fluentwork-infra` 建复用 workflow 与环境模板；
5. `fluentwork-ios` 建工程骨架与 CI skeleton；
6. `fluentwork-backend` 建工程骨架与 CI skeleton；
7. 回到 `meta` 仓补首批里程碑 Issue 与跨仓任务映射；
8. 最后再把 required checks、环境审批和 release gate 收紧。

---

## 五、分仓执行清单

## 5.1 fluentwork-meta

目标：成为治理、文档、项目管理、规范入口。

首批落地项：

1. `README.md` 继续收口为仓库入口；
2. 建 `.github/ISSUE_TEMPLATE/`；
3. 建 `.github/PULL_REQUEST_TEMPLATE.md`；
4. 建 `CODEOWNERS`；
5. 建 `.github/workflows/`：
   - `markdown-lint.yml`
   - `link-check.yml`
   - `actionlint.yml`
   - `docs-filename-check.yml`
6. 建 `agents/shared/` 与 `agents/templates/`；
7. 起草共享 `CLAUDE.md` / `AGENTS.md` 模板与 Matt Pocock skills 使用策略；
8. 建 `docs/40_研发流程与协作/` 下的流程文档；
9. 建里程碑 Issue 列表与项目看板字段。

验收标准：

- PR 能带 issue、文档依据、验收项；
- 文档仓能自动检查 Markdown、链接、文件命名；
- 关键治理目录有 owner 审批要求。

## 5.2 fluentwork-infra

目标：承接复用 workflow、环境模板、部署骨架。

首批落地项：

1. 目录骨架：
   - `.github/workflows/`
   - `env/`
   - `deploy/`
   - `monitoring/`
   - `scripts/`
2. 复用 workflow：
   - `reusable-markdown-check.yml`
   - `reusable-go-ci.yml`
   - `reusable-ios-ci.yml`
   - `reusable-deploy-gate.yml`
3. 环境模板：
   - `dev.env.example`
   - `staging.env.example`
   - `prod.env.example`
4. 部署约定文档：
   - 环境变量命名规范
   - secrets 注入规范
   - 回滚说明模板
5. `CLAUDE.md` / `AGENTS.md` 本仓入口文件

验收标准：

- 其他仓可通过 `workflow_call` 复用基础检查；
- 环境变量模板与部署脚本边界清晰；
- prod 发布具备人工审批入口。

## 5.3 fluentwork-ios

目标：建立 SwiftUI 主仓与最小 iOS CI 面。

首批落地项：

1. Xcode / SwiftPM 工程骨架；
2. TGReduxKit / Factory 基础接入位；
3. 根路由与 AppState 骨架；
4. `.github/workflows/ios-ci.yml`：
   - build
   - unit test
   - smoke test
5. `CLAUDE.md` / `AGENTS.md` 本仓入口文件；
6. 基础目录 owner 规则：
   - 音频引擎
   - SpeechSession 状态机
   - Debug / 测试桥接

验收标准：

- 仓库可在 CI 中完成最小编译与单测；
- 关键目录被 CODEOWNERS 保护；
- PR 模板能指向上游文档与 issue。

## 5.4 fluentwork-backend

目标：建立 Go 主仓与最小后端 CI 面。

首批落地项：

1. Go module 初始化；
2. `cmd/`、`internal/`、`migrations/`、`configs/` 骨架；
3. health / config / logging 基线；
4. `.github/workflows/backend-ci.yml`：
   - `go fmt`
   - `go vet`
   - unit test
   - migration check
   - Docker build
5. `CLAUDE.md` / `AGENTS.md` 本仓入口文件；
6. 关键目录 owner 规则：
   - voice gateway
   - session state machine
   - deploy / prod config

验收标准：

- 后端仓具备最小 build + test + image 检查；
- 数据库迁移与关键服务目录有 owner 门禁；
- staging / prod 发布路径预留但不立即接通。

---

## 六、组织层初始化清单

执行对象：GitHub 组织 `FluentWork`

### 6.1 Teams

至少建立：

1. `core`
2. `ios`
3. `backend`
4. `infra`

### 6.2 Projects

建议字段：

1. `Repo`
2. `Phase`
3. `Priority`
4. `Status`
5. `Owner`
6. `Milestone`

### 6.3 Branch Protection

各仓统一：

1. 保护 `main`
2. 禁止直接 push
3. 要求 PR merge
4. 要求至少 1 次审批
5. 要求 required checks 通过
6. 要求分支与主干同步后再 merge

### 6.4 Secrets / Variables

组织级优先放：

1. `OPENCODEREVIEW_*` 或通用 LLM 审查密钥
2. `APPLE_*` 签名相关密钥
3. `DEPLOY_*` 环境密钥
4. 只读公共变量，如 `CI_DEFAULT_BRANCH`

---

## 七、CI/CD 初始化分阶段计划

## Phase A：先跑通

目标：每个仓至少有 1 条成功 workflow。

包含：

1. 依赖安装
2. 基础 lint / format / syntax check
3. 最小测试或构建
4. agent 入口文件存在性检查

此阶段不做：

1. 自动部署生产
2. 重度 E2E
3. 强阻断式 AI 审查

## Phase B：变成门禁

目标：把最关键的检查升成 required checks。

包含：

1. meta：Markdown / link / actionlint / 文件命名
2. ios：build / unit test / smoke
3. backend：fmt / vet / unit test / migration / Docker build
4. infra：workflow lint / env 模板校验 / deploy dry-run
5. 4 仓：`CLAUDE.md` / `AGENTS.md` 模板一致性检查

## Phase C：接发布链路

目标：形成从 PR 到 staging / release 的标准路径。

包含：

1. `develop` 日常集成
2. `release/*` 预发布验证
3. `main` 受保护发布
4. staging / prod 环境审批
5. 发布后 health check / canary / rollback checklist

---

## 八、首批要创建的仓库文件

## 8.1 meta

```text
.github/
  ISSUE_TEMPLATE/
  workflows/
agents/
  shared/
  templates/
PULL_REQUEST_TEMPLATE.md
CODEOWNERS
```

## 8.2 ios

```text
.github/
  workflows/
CODEOWNERS
CLAUDE.md
AGENTS.md
Package.swift / Xcode project
Tests/
```

## 8.3 backend

```text
.github/
  workflows/
CODEOWNERS
CLAUDE.md
AGENTS.md
go.mod
cmd/
internal/
migrations/
```

## 8.4 infra

```text
.github/
  workflows/
CLAUDE.md
AGENTS.md
env/
deploy/
monitoring/
scripts/
```

---

## 九、推荐的首批 required checks

### meta

1. `markdown-lint`
2. `link-check`
3. `actionlint`
4. `agent-config-check`

### ios

1. `ios-build`
2. `ios-unit-test`
3. `agent-config-check`

### backend

1. `go-vet`
2. `go-test`
3. `docker-build`
4. `agent-config-check`

### infra

1. `workflow-lint`
2. `deploy-config-check`
3. `agent-config-check`

---

## 十、执行里程碑

### M0：治理面就绪

- meta 模板、CODEOWNERS、文档检查 workflow 完成
- shared agent / skills 真源与模板完成
- 组织层 teams / projects / protection 规则完成

### M1：代码仓骨架就绪

- ios / backend / infra 目录骨架完成
- 每仓至少 1 条 CI 成功

### M2：门禁生效

- required checks 打开
- 关键目录 owner 审批生效
- PR 模板与 issue 模板开始强制使用

### M3：发布链路可演练

- staging 发布演练通过
- release / rollback checklist 完成

---

## 十一、执行时的暂停点

以下节点建议在真正执行时暂停给出结果确认：

1. GitHub 组织层 team / 权限模型配置前；
2. 各仓 branch protection 开启前；
3. OpenCodeReview / 第二审查 AI 从报告模式切到阻断模式前；
4. staging / prod secrets 接入前；
5. iOS 签名与 TestFlight 自动化接入前。

---

## 十二、开始执行时的推荐顺序

当收到“开始执行”指令后，建议按以下批次落地：

1. 在 `fluentwork-meta` 落模板、CODEOWNERS、基础 workflows；
2. 在 `fluentwork-meta` 落 shared agent / skills 真源与模板；
3. 配置 GitHub 组织层 team / protection / Projects；
4. 初始化 `fluentwork-infra`；
5. 初始化 `fluentwork-ios`；
6. 初始化 `fluentwork-backend`；
7. 回到 `meta` 补 milestone issues 与 cross-repo 跟踪。

---

## 十三、产出物清单

本计划执行完成后，应至少看到：

1. 4 个仓库都有基础目录与 README；
2. 4 个仓库都有 `.github/workflows/`；
3. 4 个仓库都有 PR 模板和 CODEOWNERS；
4. 4 个仓库都有 `CLAUDE.md` 与 `AGENTS.md`；
5. `main` 分支保护与 required checks 生效；
6. `meta` 仓已有首批里程碑 issue；
7. `infra` 仓已有可复用 CI 模板；
8. 第二审查 AI 已接到 PR 流程中，但先以报告模式运行。
