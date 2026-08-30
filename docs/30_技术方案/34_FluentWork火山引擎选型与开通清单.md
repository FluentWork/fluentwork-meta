# FluentWork 火山引擎选型与开通清单

**版本**：V1.0  
**日期**：2026-08-30  
**定位**：按当前 PRD / 技术方案，说明在火山云（火山引擎）上应开通哪些**产品线、应用、模型**，以及各自服务 FluentWork 的哪条链路  
**上游**：`30` 技术方案 §2、`31` 后端、`33` Prompt、`50` 注入 POC、`meta` #12 PREREQ  
**状态**：开通指引（模型具体版本号以控制台「可开通列表」为准，本文给选型档位而非写死过期 model id）

---

## 一、先看结论：你要开两套控制台，不是一套

FluentWork 在火山侧是 **「豆包语音 + 火山方舟」双控制台**：

| 控制台 | 解决什么 | 产出凭证 | 谁用 |
|---|---|---|---|
| **豆包语音**（Speech） | 实时听/说：端到端对话、流式 ASR、流式 TTS | **API Key**（新版控制台；请求头 `X-Api-Key`；不再使用 AppID / Access Token） | `voice-gateway`（`B13`/`B14`）、模块化降级链路 |
| **火山方舟**（Ark） | 文本大模型：评价/炼化/每日一读/命中匹配等 | `API Key`（`ark-…`）+ 各场景 `Endpoint ID`（`ep-…`）；**Key 与 Endpoint 须同项目** | `app-server` / `ai-worker`（`B8`/`B11`/`B12`） |


纪律（技术方案硬口径）：

1. **凭证只落服务端**，iOS 永不持有火山 Key / Token  
2. 客户端只连自研 `voice-gateway`（自有 WSS 协议）  
3. 换模型 / 换形态只改网关与 worker，不改 App

---

## 二、按产品能力映射（开什么 → 用在哪）

### 2.1 总览图

```text
说的房间主链路（B1）
  └─ 豆包语音 · 端到端实时语音大模型          ← 优先
       └─ 失败降级 → 模块化（ASR + 方舟 LLM + TTS）

B7 命中检测旁路（B12）
  ├─ 用户话轮文本：端到端 ASR 回调 或 旁路流式 ASR
  └─ 语义匹配：方舟 · 小模型 Endpoint

回顾 / 炼化（B8）
  └─ 方舟 · 旗舰文本模型 Endpoint（评价+炼化合并一次调用）

每日一读正文（B11）
  └─ 方舟 · 旗舰文本模型 Endpoint（可与评价共用或独立 ep）

每日一读朗读 / 闪测判定（后续波次）
  ├─ 豆包语音 · 流式 TTS
  └─ 豆包语音 · 流式 ASR + 方舟 · 小模型（语义判定 E2）
```

### 2.2 豆包语音：必须开通的服务

在 **豆包语音控制台 → 应用管理** 创建应用（建议命名见第四节），并对该应用开通：

| 开通服务（控制台名称） | FluentWork 用途 | 对应 issue / 波次 | 优先级 |
|---|---|---|---|
| **端到端实时语音大模型**（含全双工/半双工版本，以控制台当前可开通项为准） | 「说的房间」主对话：ASR+对话+TTS 一连接 | `B13`/`B14`/`I12`，Phase 0 | **P0 立刻开** |
| **流式语音识别大模型**（大模型流式 ASR） | ① 模块化降级；② B14/V6 旁路 ASR；③ 闪测跟读取文本 | `B14` 备选、`B13` 降级、后续闪测 | **P0 立刻开**（与端到端同 App） |
| **语音合成大模型**（流式 TTS / 双向流式） | 模块化降级播报；每日一读朗读（语速 0.8/1.0/1.2） | `B13` 降级、`B11`/朗读 | **P0 立刻开** |
| 录音文件识别（可选） | 会话结束后补转写兜底（非主路径） | 后续优化 | P2 |
| 发音评测类能力（若控制台有） | **V1.1 再开**，MVP 不做 | 后置 | 不做 |

**新版口径（官方已修正）：鉴权只靠 API Key（`X-Api-Key`）**，覆盖端到端 / 流式 ASR / TTS；**不再依赖 AppID / Access Token**。  
控制台里的「实例名」是开通/计费记录，**不是**工程必填凭证；端到端实时语音建连文档（新版）只需 `X-Api-Key` + `X-Api-Request-Id`。  
少数子接口（如部分 TTS / 流式 ASR）若文档仍要求 `X-Api-Resource-Id`，使用官方**产品级 SKU**（如 `seed-tts-2.0`、`volc.seedasr.sauc.duration`），**不要**把控制台实例名长串当 ResourceId。

端到端选型注意（对接 `B14`）：

1. 优先选文档标明可返回 **用户话轮 ASR 文本事件**（如 ASRResponse / Chat 文本回调）的版本  
2. 确认是否支持 **会话中途更新 system / 追加上下文**（注入通道）——这是 B7 的 Go/No-Go  
3. 控制台若区分 **O 版本（低时延助手）** 与 **SC 版本（强人设）**：说的房间更偏向「可配 system / 人设 + 低时延」；POC 阶段两个都要能拉到文档，再实测锁定一个  
4. 产品页参考：<https://www.volcengine.com/product/realtime-voice-model>；API 文档入口：豆包语音 → 端到端实时语音大模型

### 2.3 火山方舟：必须创建的推理接入点

在 **火山方舟 → 开通管理** 开通模型后，在 **在线推理 → 创建推理接入点**，为每个业务场景建独立 Endpoint（便于配额、灰度、成本拆账）。

| Endpoint 建议名 | 选型档位（控制台选「当前旗舰 / 轻量」） | FluentWork 用途 | 对应 issue |
|---|---|---|---|
| `fw-review-refine` | **旗舰档**（如 Doubao Seed / Pro 系长上下文；会话转录可达数千 token） | 评价 + 炼化合并一次调用，强制 JSON Schema | `B8` |
| `fw-daily-read` | **旗舰档**（可与上者同模型不同 ep，便于单独限流） | 每日一读正文生成 | `B11` |
| `fw-topic-card` | **旗舰档**（可暂复用 `fw-review-refine`，W7 再拆） | 话题卡生成（第二波后） | 后续 |
| `fw-hit-match` | **小模型 / Lite / Mini 档**（低延迟、低价） | B7 话术块语义匹配，超时 800ms | `B12` |
| `fw-drill-judge` | **小模型档**（可与 hit-match 同模型不同 ep） | 闪测语义判定 E2 | 后续闪测 |
| `fw-text-degrade` | **小模型或中档** | 三级降级末级：纯文本对话 | `B13` 降级 / 已有 messages stub 替换 |

调用约定（方舟）：

1. Base URL 形如：`https://ark.cn-beijing.volces.com/api/v3`（以控制台为准）  
2. 请求里的 `model` 填 **Endpoint ID（`ep-…`）**，不是营销名  
3. 评价 / 炼化必须开 **结构化输出 / JSON Schema**（或等价 constrained decoding）；不开则不要接 `B8`  
4. 每次调用落 `ai_cost_logs`（技术方案纪律）

**文本模型档位怎么选（避免被型号改名坑到）：**

| 档位 | 选用原则 | FluentWork 场景 |
|---|---|---|
| 旗舰 | 强指令遵循、长上下文、结构化 JSON 稳 | 评价 / 炼化 / 每日一读 / 话题卡 |
| 小模型 | 首 token 快、单价低、短输入短输出 | B7 命中、闪测判定、文本降级 |

控制台开通时：在「语言模型」里勾选对应系列并点开通；**不要只开语音、忘开方舟**——第二波 `B8` 会直接卡死。

---

## 三、按 FluentWork 链路的「最小开通包」

### 3.1 本周就能支撑 Phase 0（B14 + 后续 B13）

必须齐：

1. 豆包语音应用 ×1（开发）  
2. 开通：端到端实时语音 + 流式 ASR + 流式 TTS  
3. 方舟 API Key ×1  
4. 方舟 Endpoint：至少 `fw-hit-match`（小模型）——B14 注入 POC 若要测「命中后注入话术」的行为，可先人工写死注入文本，不强制先开；**但 B12 前必须开**  
5. 商务三项（`meta` #12）：免费额度算账、数据不用于训练条款、并发配额  

可选但强烈建议同步开：

6. 方舟 `fw-review-refine`（旗舰）——方便 `B15`/`B8` 立刻接真模型，不必等语音 POC 结束  

### 3.2 第二波功能面（B8 / B11 / B12）

在 3.1 基础上补齐：

| 能力 | 开通项 |
|---|---|
| 回顾 + 炼化 | `fw-review-refine` |
| 每日一读 | `fw-daily-read`（或复用旗舰 ep） |
| B7 匹配 | `fw-hit-match` |
| 文本降级真模型 | `fw-text-degrade` |

### 3.3 明确先不要开 / 后置

| 能力 | 原因 |
|---|---|
| 发音评测 | V1.1 |
| 多口音 TTS / 声音复刻 | V1.5 听力训练 |
| 视觉多模态 | 本产品无图文主路径 |
| 海外节点 / 出境链路 | 合规定案国内 |

---

## 四、应用与项目管理建议

### 4.1 豆包语音怎么建

新版按 **API Key + 开通服务** 管理即可（不再建 AppID/Token；**工程接线以 Key 为准，不强制落 ResourceId / 实例名**）。

| Key 用途 | 环境 | 控制台 Key 名（2026-08-30） | 说明 |
|---|---|---|---|
| 语音 Dev / default | 本地 / POC / 内测 | `api-key-20260830213945` | `B14`、gateway 联调；写入 `VOLC_SPEECH_API_KEY(_DEV)` |
| 语音 Prod | 生产 | `api-key-20260830213853` | 与 Dev 隔离；写入 `VOLC_SPEECH_API_KEY_PROD` |

开通记录（便于对账，非 env 必填）：端到端实例、流式 ASR 实例已在控制台创建。每把 Key 旁记录已开通服务列表即可。

### 4.2 方舟侧怎么建

**当前 POC / 本地开发口径（已落地）：**

1. 项目：`default`  
2. API Key：Dev 用 Key（与 default 下 `fw-*` Endpoint 同项目）  
3. Endpoint：见第十节「Dev 清单」  

**生产口径（已落地，六路 smoke PASS）：**

1. 项目：`FluentWork-Prod`  
2. API Key：Prod 专用 Key  
3. Endpoint：见第十节「Prod 清单」（含 `fw-drill-judge` = `…pmkg6`）  
4. 生产环境变量只注入 Prod Key + Prod `ep-`，**禁止** default Key 打生产流量  
5. 账单与配额按 Prod 项目单独核算  

纪律：

1. **Key 与 Endpoint 必须同项目**（跨项目会 403/404，已实测）  
2. **禁止**所有场景共用一个 ep  
3. Endpoint 备注写清对应 FluentWork issue（如 `B8`）

### 4.3 密钥落盘（工程）

建议环境变量（名称可微调，但语义固定）：

```text
# 豆包语音（gateway）— 新版仅 API Key（必填）；不强制 ResourceId / 实例名
VOLC_SPEECH_API_KEY=          # 本地默认 Dev
VOLC_SPEECH_API_KEY_DEV=
VOLC_SPEECH_API_KEY_PROD=
# 可选：仅当 TTS/ASR 子接口文档仍要求产品级 X-Api-Resource-Id 时再填
# VOLC_SPEECH_RESOURCE_TTS=seed-tts-2.0

# B14 POC 直连（可与 Dev 语音 Key 共用）
VOLC_POC_API_KEY=

# 火山方舟（worker / app-server）— Key 与 ep 同项目
ARK_API_KEY_DEV=ark-...
ARK_API_KEY_PROD=ark-...
ARK_API_KEY=                 # 本地可指向 DEV
ARK_EP_REVIEW_REFINE=ep-...
ARK_EP_DAILY_READ=ep-...
ARK_EP_HIT_MATCH=ep-...
ARK_EP_TEXT_DEGRADE=ep-...
```

全部只进服务端密钥管理 / `.env`（gitignore），CI 用 OIDC 或仓库 Secrets。

---

## 五、开通操作 checklist（执行顺序）

### Step A — 账号与商务（卡 B13/B14 live）

1. 注册 / 登录 [火山引擎](https://www.volcengine.com/)，完成**企业实名**（个人可 POC，正式建议企业）  
2. 谈定并书面确认：  
   - API 数据**不用于训练**  
   - 实时语音**并发路数**与突发扩容 SLA  
   - 免费额度 vs 内测用量（人均分钟 × 20 人 × 4 周）  
3. 结果回写 `meta` #12 PREREQ  

### Step B — 豆包语音

1. 打开豆包语音控制台 → 开通端到端 / 流式 ASR / TTS  
2. 在 **API Key 管理** 创建语音 API Key（新版鉴权：仅 `X-Api-Key`）  
3. Dev / Prod 各一把 Key 写入服务端 Secrets（**无需**把控制台实例名写进工程配置）  
4. 跑 `./scripts/smoke-volc-speech-creds.sh`；B14 再补端到端 WSS 真会话  

### Step C — 火山方舟（Dev = default）

1. 控制台进入 **火山方舟**  
2. **开通管理**：开通旗舰（Seed pro）+ 轻量（Seed turbo）  
3. 在项目 **`default`** 创建六路 `fw-*` 推理接入点  
4. 在 **`default`** 创建 Dev API Key  
5. `./scripts/smoke-volc-ark.sh` 六路全绿  

### Step C′ — 火山方舟（Prod，上线前必做）

1. 项目 **`FluentWork-Prod`** 下重建同名六路 Endpoint + 生产 API Key  
2. 生产密钥进 Secrets，不进 git  
3. 用 Prod Key 单独跑一遍 smoke（可复用脚本，换 env）  

### Step D — 回写工程

1. 更新 `fluentwork-backend` 本地 `.env.example` 注释（不提交真密钥）  
2. `B14` live adapter 就绪后重跑 `./scripts/poc-injection-window.sh`，档位结论回写 `50` 号文档  
3. `B8` 接 `ARK_EP_REVIEW_REFINE`，跑 `./scripts/eval-prompt-regression.sh` + 真模型抽样  

---

## 六、能力 × 模型 × 应用 对照表（给采购 / 开通的人）

| FluentWork 能力 | 火山产品线 | 控制台「应用 / 接入点」 | 模型档位 | 何时必须就绪 |
|---|---|---|---|---|
| 说的房间实时对话 | 豆包语音 · 端到端 | `FluentWork-*` App | 端到端实时语音（低时延 / 可配人设） | Phase 0 `B13`/`B14` |
| 说的房间降级 | 豆包 ASR + 方舟 LLM + 豆包 TTS | 同上 App + `fw-text-degrade` | ASR 大模型 + 小/中档文本 + TTS | 与主链路同时具备 |
| B7 话轮文本 | 端到端 ASR 回调或旁路流式 ASR | 同上 App | 流式 ASR | `B14`/`B12` |
| B7 语义匹配 | 火山方舟 | `fw-hit-match` | 小模型 | `B12` 前 |
| 评价 + 炼化 | 火山方舟 | `fw-review-refine` | 旗舰 | `B8` 提测前 |
| 每日一读文案 | 火山方舟 | `fw-daily-read` | 旗舰 | `B11` |
| 每日一读播报 | 豆包语音 · TTS | 语音 App | 流式 TTS（确认变速） | 朗读页前 |
| 闪测判定 | 豆包 ASR + 方舟小模型 | 语音 App + `fw-drill-judge` | 流式 ASR + 小模型 | 闪测波次 |

---

## 七、常见误区

1. **只开通「豆包 App 网页版」≠ 有 API**：必须走火山引擎控制台开通语音 / 方舟。  
2. **用方舟 API Key 去连语音 WSS，或用语音 Token 调方舟**：两套凭证，不要混。  
3. **方舟只开通模型、不建 Endpoint**：生产调用会 404。  
4. **所有场景共用一个旗舰 Endpoint**：B7 会被旗舰排队拖死，打穿 800ms 旁路预算。  
5. **客户端直连火山**：违反架构与安全口径；POC 也只能服务端 / 网关侧直连。  
6. **把 mock T9 报告当成选型结论**：mock 只验证脚手架；选型与 B7 档位必须以 live 为准。  

---

## 八、参考链接（官方入口）

1. 火山引擎：<https://www.volcengine.com/>  
2. 端到端实时语音产品：<https://www.volcengine.com/product/realtime-voice-model>  
3. 豆包语音文档中心（端到端 / ASR / TTS）：<https://www.volcengine.com/docs/6561>  
4. 端到端 API 接入文档：<https://www.volcengine.com/docs/6561/1257584>（若改版以控制台文档树「端到端实时语音大模型」为准）  
5. 火山方舟（文本大模型）：控制台搜索「方舟」/「Ark」  
6. 本仓库：`50_FluentWork端到端注入能力验证文档.md`、`backend/docs/04_B14_注入POC执行清单.md`

---

## 九、一句话给执行同学

> **语音开「豆包语音」一个 Dev 应用（端到端 + 流式 ASR + TTS）；文本开「火山方舟」两档模型（旗舰给评价炼化，小模型给 B7）；凭证只进服务端。先把这套开齐，Phase 0 的 B14/B13 和后面的 B8/B12 才有地方接。**

---

## 十、开通核对（2026-08-30 实测）

### 10.1 Dev / POC（项目 `default`）

| 项 | ID / 说明 | 连通 |
|---|---|---|
| API Key | default 项目 Key（本地 `ARK_API_KEY` / `ARK_API_KEY_DEV`） | 有效 |
| `fw-review-refine` | `ep-20260830204651-pffhf`（pro） | PASS |
| `fw-daily-read` | `ep-20260830204818-8kdfr`（pro） | PASS |
| `fw-topic-card` | `ep-20260830204912-wtjw9`（pro） | PASS |
| `fw-hit-match` | `ep-20260830205333-prddb`（turbo） | PASS |
| `fw-drill-judge` | `ep-20260830205423-xg4pd`（turbo） | PASS |
| `fw-text-degrade` | `ep-20260830205520-d9d8n`（turbo） | PASS |
| 豆包语音 Dev Key | `api-key-20260830213945`（`X-Api-Key`） | 已建；Key 已校正 |
| 豆包语音 Prod Key | `api-key-20260830213853` | 已建；Key 本身有效；TTS SKU 未授权（403，不影响端到端主路径） |
| 端到端 / 流式 ASR | 控制台已开通（实例名仅对账，**不进工程必填**） | 开通齐；WSS 待 B14 |

### 10.2 Prod（项目 `FluentWork-Prod`）

Endpoint 已创建；用 Prod API Key 实测：

| Endpoint | ID | 模型 | 连通 |
|---|---|---|---|
| `fw-review-refine` | `ep-20260830211617-26d79` | pro | PASS |
| `fw-daily-read` | `ep-20260830211650-vkdj2` | pro | PASS |
| `fw-topic-card` | `ep-20260830211715-q79x9` | pro | PASS |
| `fw-hit-match` | `ep-20260830211747-vwtrb` | turbo | PASS |
| `fw-drill-judge` | `ep-20260830211815-pmkg6` | turbo | PASS |
| `fw-text-degrade` | `ep-20260830211850-pf2ts` | turbo | PASS |

纪律：生产只注入 Prod Key + 上表 Prod `ep-`；**禁止**用 default Key 打 Prod Endpoint。

### 10.3 第二波开写前仍缺

| 项 | 说明 | 阻断谁 |
|---|---|---|
| 端到端语音 **WSS 真会话** | Key + 开通已齐；TTS HTTP ≠ realtime 注入能力 | `B13`/`B14` |
| `meta` #12 商务三项 | 额度 / 不训练 / 并发 | live 冻结 B12 前 |

> 方舟文本侧：Dev（default）与 Prod（FluentWork-Prod）**六路 Endpoint 均已 smoke PASS**（2026-08-30）。  
> 豆包语音侧：新版 **只接 API Key**；控制台实例名不进工程必填配置。

### 10.4 连通性测试

```bash
cd fluentwork-backend
cp configs/volc.env.example .env.volc.local
# Dev：ARK_PROJECT=default + ARK_API_KEY_DEV + default 的 ep-
# Prod：ARK_PROJECT=FluentWork-Prod + ARK_API_KEY_PROD + 10.2 的 ep-
set -a && source .env.volc.local && set +a
./scripts/smoke-volc-ark.sh
./scripts/smoke-volc-speech-creds.sh
```

安全：API Key 勿贴进聊天 / 勿提交 git；若已泄露请轮换。
