# 82 — FluentWork 环境启动总览

**定位**：一份「我现在要跑起来」的对照表。回答的是**怎么起、起成什么样、怎么验**，
不重复 `30_技术方案/` 里的架构决策，也不替代各仓的 README。

**日期**：2026-09-25
**核对方式**：每条都指到脚本或配置文件的具体行；启动结果取自当日实测日志。

---

## 0. 一页速查

| 我想做的事 | 命令 | 前置 |
|---|---|---|
| 起后端（最快，不碰 Docker） | `./scripts/dev-up.sh` | Go 1.26+ |
| 起后端 + 本地 MySQL | `./scripts/dev-up.sh --local-mysql` | brew MySQL 已跑 |
| 起后端 + Docker MySQL/Redis | `./scripts/dev-stack.zsh` | Docker daemon |
| 只起后端不起语音网关 | `./scripts/dev-up.sh --no-gateway` | — |
| 给**真机**联调 | `./scripts/dev-up.sh --host <Mac 的 LAN IP>` | 手机与 Mac 同一 Wi-Fi |
| 换成本地假语音（不连火山） | `VOICE_GATEWAY_PROVIDER=dev-echo ./scripts/dev-up.sh` | — |
| 停 | 在起服务的终端按 `Ctrl-C`；再 `./scripts/dev-down.sh` 收 Compose | — |
| 提交前跑门禁 | `./scripts/dev-check.sh` | gofumpt / goimports / golangci-lint |
| iOS 模拟器冒烟 | `./Scripts/smoke-iphone17pro.sh` | Xcode |
| iOS 单元测试 | `swift test` | Swift 6 工具链 |

---

## 1. 仓与端口约定

| 仓 | 角色 | 本地端口 |
|---|---|---|
| `fluentwork-meta` | 产品/架构决策的唯一上游 | — |
| `fluentwork-backend` | `app-server` / `voice-gateway` / `worker` | 8080 / 8081 |
| `fluentwork-ios` | SwiftUI 客户端 | — |
| `fluentwork-infra` | 部署与共享 schema 真源 | MySQL 3306 / Redis 6379 |

**iOS 侧只认两个地址**：HTTP `http://<host>:8080/api/v1`、WSS `ws://<host>:8081/v1/voice`。

---

## 2. Backend：两个**独立**的轴

这是本节唯一需要记住的一句话：**「存储」和「语音 provider」是两条不相干的轴，不要混着说。**

| 轴 | 取值 | 由谁决定 |
|---|---|---|
| **存储** | 进程内存 / 本地 MySQL / Docker MySQL | 有没有 `MYSQL_DSN` |
| **语音 provider** | `mock` / `dev-echo` / `volc-duplex` | `VOICE_GATEWAY_PROVIDER` |

### 2.1 最容易踩的一条：`dev-up.sh` 会加载 `.env.volc.local`

`scripts/dev-up.sh:116-123` 的加载顺序是：

1. `.env`（存在则用），否则 `configs/app-server.env.example`
2. **然后无条件再加载 `.env.volc.local`**（存在则用）

而 `.env.volc.local` 里设了 `VOICE_GATEWAY_PROVIDER` 与火山/方舟的密钥。
所以：

> **默认的 `./scripts/dev-up.sh` 并不是「纯离线」——它是「内存存储 + 真实火山语音 provider」。**

当日实测日志可证：`{"msg":"B8 stuck rescue wired","provider":"volc-duplex"}`。
这意味着默认模式下**语音链路要连外网、要有有效密钥**。

另外，`internal/voicegateway/config.go:124` 里 `LoadConfig()` **自己也会加载**
`.env.volc.local`，不依赖启动脚本。容器部署不需要这两个 dotenv 文件
（`config.go:119`）。

**优先级**：脚本显式 export 的变量 > `.env.volc.local` > `.env` / 示例文件
（`dev-up.sh:107-111` 对已存在的同名变量直接跳过）。所以临时改 provider 只要：

```bash
VOICE_GATEWAY_PROVIDER=dev-echo ./scripts/dev-up.sh
```

`dev-echo` 是给开发者跑通全链路用的本地替身（`provider_factory.go:11-20`），
它按脚本返回固定的 `ServerASRText`，不需要任何密钥。

### 2.2 模式 A：内存存储（默认）

```bash
./scripts/dev-up.sh
```

- `MYSQL_DSN` 被清空（`dev-up.sh:207-209`）→ 账号/会话/语料/内容/成本/练习/素材/话题
  八个 store 全部走进程内存。启动日志里会有 9 条
  `MYSQL_DSN is empty; using in-memory ... store` 的 WARN。
- **数据不持久**，重启即清空。
- app-server 上 `0.0.0.0:8080`，voice-gateway 上 `0.0.0.0:8081`。
- 启动流程自带：等 `/healthz` → 冒烟 `POST /api/v1/auth/guest` → 播种开发语料
  （`AUTO_CORPUS_SEED` 默认 1，播给 `corpus-seed-dev-device`）。
- `--no-gateway` 只起 app-server。

> 真机不需要手动播种语料：开发环境下 app-server 会在设备的**第一个会话**自动
> 播种同一份起步语料（`corpus.StarterProvisioner`），日志为
> `starter corpus provisioned (development only)`。

### 2.3 模式 B：本地 MySQL（brew）

```bash
./scripts/dev-up.sh --local-mysql
```

- 要求 `mysql` CLI 可用且本机 MySQL 在跑（`dev-up.sh:185-192`），否则直接退出并提示
  `brew services start mysql`。
- 会自动建库建用户：`fluentwork` 库 + `fw`/`fw`（`dev-up.sh:193-195`），然后按文件名顺序
  执行 `migrations/*.sql`。
- 设 `MYSQL_DSN=fw:fw@tcp(127.0.0.1:3306)/fluentwork?...`，并打开
  `APP_RUN_REVIEW_WORKER=1`（共享 MySQL 时 app-server 的进程内 review worker 默认是关的，
  `dev-up.sh:211-216`）。

> ⚠️ **重启必须加 `--skip-migrations`。**
> `0008` 起的迁移是 `ALTER`，不幂等；重放会 `Duplicate column name ...` 并把整轮启动带走
> （`dev-up.sh:31-34, 171-181`）。所以第二次起是：
> ```bash
> ./scripts/dev-up.sh --local-mysql --skip-migrations
> ```
> 注意 `--skip-migrations` 在不带 `--mysql` / `--local-mysql` 时是 no-op，脚本会主动提醒
> （`dev-up.sh:82-85`）。

首次初始化也可以走低层脚本：`./scripts/local-db-init.sh`（建库建用户 + 跑迁移）。
注意它建的库排序规则是 `utf8mb4_unicode_ci`，而 `dev-up.sh --local-mysql` 用的是
`utf8mb4_0900_ai_ci` —— 两者不一致，只影响排序语义，但换路径时要知道。

### 2.4 模式 C：Docker 全栈（MySQL + Redis）

```bash
./scripts/dev-stack.zsh
```

- 用 `deploy/docker-compose.yml` 拉起 `mysql:8.0`（3306）与 `redis:7`（6379），
  库 `fluentwork`、用户 `fw/fw`、root `fwroot`。
- 迁移在容器内执行（`docker compose exec -T mysql mysql -ufw -pfw fluentwork`）。
- 然后把控制权交给 `dev-local-start.sh --no-services`。
- 收尾：`./scripts/dev-down.sh`。
  **注意它只停 Compose 服务**，脚本自己写明 app-server / voice-gateway 不归它管，
  要用 `Ctrl-C` 停。

也可以只要 MySQL：`./scripts/dev-up.sh --mysql`。

### 2.5 低层脚本（一般不要直接用）

`dev-local-start.sh`（brew services 版全流程）、`local-services-start.sh`、
`local-db-init.sh`、`local-services-stop.sh`。`docs/01_本地启动.md` 的分层建议是：
**新人优先用模式 A/B/C，不要从这一层拼装启动流程。**

### 2.6 质量门禁与活体 smoke

```bash
./scripts/dev-check.sh          # gofumpt / goimports / golangci-lint / go test / go build / 缺陷纪律
./scripts/smoke-review-ready.sh # guest -> session -> session.end -> worker -> review ready
```

`dev-check.sh` 会先检查三个工具在不在（`gofumpt` / `goimports` / `golangci-lint`），
缺任何一个直接失败。注意 `104_架构分析/05` 记过一条：**`golangci-lint` 本机曾未安装，
`depguard` 那条纪律因此在本机从未执行过。**

---

## 3. iOS 侧

### 3.1 模拟器

`.xcodeproj` 已在仓里，直接构建即可。单元测试：

```bash
swift test
```

> 若改了 `project.yml`，需要重新生成工程：`USER=$(id -un) xcodegen generate`。
> **`USER` 这个前缀不能省** —— XcodeGen 读 `USER` 环境变量，而在本机工具环境里它是
> 未设置的，会报 `Couldn't find current username` 并且**静默不生成**工程。

模拟器冒烟（起模拟器 + 装 + 启动 + 跑导航测试）：

```bash
./Scripts/smoke-iphone17pro.sh
```

可覆盖：`SIMULATOR_NAME`（默认 `iPhone 17 Pro`）、`BUNDLE_ID`（`com.fluentwork.host`）、
`SCHEME`（`FluentWorkHost`）、`SKIP_HOST_BUILD=1`（只跑测试不构建）、`DERIVED_DATA`、`LOG_DIR`。

### 3.2 真机：`LOCAL_HOST` 管到哪儿、不管到哪儿

在 Xcode scheme 的 Run → Arguments → Environment Variables 里加 `LOCAL_HOST = <Mac 的 LAN IP>`。
`AppEnvironment.current`（`AppEnvironment.swift:87-94`）读到它就走 `.local(host:)`。

**但要说清楚它的作用边界**，否则会照着一个错的模型排障：

| 项 | 由 `LOCAL_HOST` 决定吗 |
|---|---|
| HTTP `apiBaseURL` | ✅ 是，`http://<host>:8080/api/v1` |
| WSS 地址（**实际用的那个**） | ❌ **不是** |
| `AppEnvironment.wssBaseURL` | ✅ 是，但**这个字段在生产路径上没人读** |

最后一行是关键：`DefaultSpeechSessionClient.startSession` 用的是
**后端 `POST /sessions` 返回的 `wss_url`**（`DefaultSpeechSessionClient.swift:78`
`URL(string: created.wssURL)`），而 `wssBaseURL` 全仓只在自己的定义处和一个测试断言里
出现（`FoundationComponentsTests.swift:368`）。

> **所以真机能不能连上，取决于后端启动时的 `--host`，不取决于 iOS 侧的 `LOCAL_HOST`。**
> `LOCAL_HOST` 只负责把 HTTP 打到 Mac 上；WSS 地址是后端告诉客户端的。

顺带一条容易吃亏的：`.local` 的默认值**不是 localhost，而是一个写死的 IP 字面量**
（`AppEnvironment.swift:39-40`），注释里写的「Fall back to localhost (works for simulator)」
与实际不符。这个值**会随 Mac 的 LAN IP 变化而失效**，且它同时决定 HTTP 与
（无人读的）`wssBaseURL`：

- 2026-09-25 工作区里它是 `192.168.2.181`
- 同一个文件的 HEAD 版本是 `192.168.2.156`

也就是说**换网络环境时它是第一个要改的地方**，而且改它是改代码、不是改配置。
更好的做法是把 host 也走 `LOCAL_HOST`，让默认值真的落到 `127.0.0.1`。
`TEST_LOCAL_HOST` 只影响 `testLocal`，不影响 `.local`。

### 3.3 麦克风替身

真机验证不必真说话：

| 变量 | 作用 |
|---|---|
| `FW_MOCK_MIC=1` | 打开替身；点一次「开始说话」= 说一句脚本化的话 |
| `FW_MOCK_MIC_UTTERANCE_MS` | 这一句持续多久（默认 1500ms） |
| `FW_MOCK_MIC_AUTO_MS` | 可选，无人值守，每 N 毫秒自动说一句 |

采集交给脚本，**播放仍走真引擎**（`AppDependencies.swift:629`、`MockAudioEngine.swift:43-45`）。

### 3.4 落地门禁（iOS 与 backend 不同）

`AGENTS.md` 的 Local Review Gate 要求三件一起提交：

1. `swift test`
2. `FluentWorkHost` 的 Debug 构建
3. `docs/` 下一号 `NN_` 实现说明（写清原则、落点文件、根因、为什么不并进既有路径、影响面）

---

## 4. 真机联调的正确姿势

```bash
# 1. 查 Mac 的 LAN IP（en0 可能没有 IP，别只看 en0）
ipconfig getifaddr en0; ipconfig getifaddr en1

# 2. 用这个 IP 起后端 —— 它会成为回给 iOS 的 wss_url
./scripts/dev-up.sh --host 192.168.2.181

# 3. iOS scheme 里加 LOCAL_HOST=192.168.2.181
```

脚本会把两行都打出来，照抄即可：

```
📡 WSS URL for iOS: ws://192.168.2.181:8081/v1/voice
   Set LOCAL_HOST=192.168.2.181 in Xcode scheme for physical device testing.
```

### 4.1 一个真实故障的完整判据链（2026-09-25）

**症状**：真机起会话后 267ms 就 `connecting → failed`。

```
[Tracker] timing_phase_transition ["total_ms":"267.345","from":"connecting","to":"failed"]
[Tracker] timing_session_end ["total_ms":"267.439"]
```

**判据链**（值得记住的是「用哪条日志否定哪条假设」）：

1. app-server 日志里有该设备的 `POST /api/v1/auth/guest` 200 与
   `POST /api/v1/sessions` 200（`device_id` 是真 UUID）→ **HTTP 通路正常**，
   客户端确实拿到了 `session_id` 与 `ticket`。
2. app-server **没有** `/internal/v1/tickets/consume` 的记录。
   voice-gateway 在 WSS 握手时会调它（`ticket_client.go:81`），而 app-server
   记录每一条 HTTP 请求 → **网关从未收到连接**。
3. 网关自己**没有请求日志中间件**，所以「网关日志空白」本身不是证据；
   证据是第 2 条的缺席。
4. 会话接口返回的 `wss_url` 当时是 `ws://127.0.0.1:8081/v1/voice`
   （因为 `dev-up.sh` 的 `HOST` 默认 `127.0.0.1`）→ 在手机上 `127.0.0.1` 指向手机自己
   → 立即 connection refused，267ms 的量级吻合。

**根因**：后端回给客户端的 WSS 地址是 `127.0.0.1`，而客户端在另一台设备上。

**修法**：用 `--host <LAN IP>` 重启后端（本文 §4）。

**顺带否定的两个假设**（避免下次再查一遍）：
- 不是 ATS/明文网络问题 —— 同一个 host 的明文 HTTP 已经打通，WebSocket 走同一套策略。
- 不是鉴权/ticket 问题 —— guest 与 sessions 都是 200，ticket 根本没被消费。

---

## 5. 验证清单

```bash
# 健康
curl -sS http://127.0.0.1:8080/healthz
curl -sS http://127.0.0.1:8080/readyz
curl -sS http://127.0.0.1:8081/healthz

# 拿 token
TOKEN=$(curl -sS -H 'Content-Type: application/json' \
  -d '{"device_id":"device-1"}' \
  http://127.0.0.1:8080/api/v1/auth/guest \
  | python3 -c 'import sys,json;print(json.load(sys.stdin)["access_token"])')

# 关键一步：确认后端回给客户端的是哪个地址
curl -sS -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"scene_type":"standup"}' \
  http://127.0.0.1:8080/api/v1/sessions \
  | python3 -c 'import sys,json;print(json.load(sys.stdin)["wss_url"])'
```

**最后一条是真机联调前唯一必须看的一行。** 它决定客户端会去连哪里。

契约发现：`GET /`、`GET /openapi.yaml`、仓内同源 `api/openapi-v1.yaml`。

---

## 6. 常见失败模式

| 症状 | 判据 | 处置 |
|---|---|---|
| 真机 `connecting → failed`（毫秒级） | `wss_url` 是 `127.0.0.1` | `--host <LAN IP>` 重启后端 |
| 真机 HTTP 也不通 | app-server 日志没有 guest 请求 | 检查 `LOCAL_HOST` 与是否同一 Wi-Fi |
| 模拟器/真机都连不上，但后端日志全空 | `AppEnvironment.local` 里写死的 IP 与当前 LAN 不一致 | 改 `.local` 的 IP，或设 `LOCAL_HOST` 绕开它（见 §3.2） |
| `--local-mysql` 报 `Duplicate column name` | 迁移重放 | 加 `--skip-migrations` |
| 启动即失败，报 provider 需要密钥 | `volc-duplex` 缺 `VOLC_SPEECH_API_KEY` | 补密钥，或 `VOICE_GATEWAY_PROVIDER=dev-echo` |
| `dev-check.sh` 报 `missing golangci-lint` | 工具未装 | `go install github.com/golangci/golangci-lint/v2/cmd/golangci-lint@latest` |
| 会话卡在 `.connecting` 直到 10s 看门狗 | 音频会话 category 被别的组件改成非 `.playAndRecord` | 见 `fluentwork-ios/docs/` 对应实现说明（`AVAudioSession` 被抢） |
| 改了 `project.yml` 后工程没变 | XcodeGen 报 `Couldn't find current username` | `USER=$(id -un) xcodegen generate` |

---

## 7. `fluentwork-infra` 的现状：还没有「部署环境」可配

`infra` 目前只有骨架 —— `deploy/`、`docker/`、`environments/`、`monitoring/`、`scripts/`
五个目录里**只有 `.gitkeep`**。实际在用的是：

- 共享 schema 真源：`schemas/transport/wss-control-frames-v2.json`（控制帧；v1 已于 2026-09-25 退役）
- 二进制音频帧布局真源：`schemas/transport/wss-binary-audio-frames-v1.json`（sha256 冻结）
- 可观测性事件 schema：`schemas/events/speech-observability-events-v1.json`
- 可观测性设计文档：`docs/observability/`

iOS 与 backend 各自的 `schemas/` 是它的**镜像**，用各仓的 `sync-shared-schemas.sh` 同步。
**改 schema 要改 infra 那份**，镜像不是真源。

所以：**现在没有 staging / production 的可执行启动路径**，`docs/` 里所有「环境」讨论
指的都是本地开发环境。这一条以后变了要回来改。

---

## 8. 维护约定

改了下面任何一处，回来同步本文：

| 改了 | 本文对应节 |
|---|---|
| `scripts/dev-up.sh` 的参数或加载顺序 | §0、§2 |
| `.env.volc.local` 里的 provider / 密钥项 | §2.1 |
| `deploy/docker-compose.yml` 的服务或凭据 | §2.4 |
| `AppEnvironment.swift` 的 host 逻辑 | §3.2 |
| `FW_MOCK_MIC` 的开关 | §3.3 |
| `infra` 的 `environments/` 不再为空 | §7 |
