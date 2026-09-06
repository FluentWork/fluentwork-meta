# seed-tts-2.0 iOS SDK 引入决策:辩证分析

> 分支：`analysis/seed-tts-2.0-sdk-vs-relay-tts`
> 日期：2026-09-06
> 前置：
> - [`70_` 系列约束](#证据索引)——尤其 71_「SDK 与自研链路对比分析」、72_「PRD 主闭环视角接入决策评估」
> - `56_B17_TTS_Provider_Issue_Draft_2026-09-06.md`——B17 实施包
> - `62_I15_TTS播放集成_Issue_Draft_2026-09-06.md`——iOS 端 TTS 播放集成
> - `74_V2_0_凭证与P0修复验证日志_2026-09-06.md` §2.5——TTS 凭证现状
> 范围：**只评估**「在 I15 实施时,把客户端 TTS 解码 / 播放 / jitter buffer / 中断模块,改为复用 seed-tts-2.0 iOS 客户端 SDK」这一个子决策
> 不重复：全 SDK 替换(Dialog 全双工 SDK)、端侧 ASR SDK、音频技术 SDK——这些已由 71_/72_ 拍板

---

## TL;DR

**结论:不建议在 I15 中引入 seed-tts-2.0 iOS SDK。保留手写 Opus 解码 + AVAudioEngine 方案。**

理由不是"SDK 不好",而是用 I15 这次发射窗口换 SDK 的**净收益为负**:
1. 当前瓶颈不在 iOS 解码层,而在 voicegateway ↔ 火山 WSS 后半段(`provider_volc_duplex.go` 的 keepalive / turn 状态机才是历史故障多发点)
2. seed-tts-2.0 SDK 是「客户端 SDK 形态」,即使只取 TTS 子集,也会引入"客户端需要某种身份凭证/会话绑定方式"的接缝,要么走完整 direct mode(违反零 key 原则),要么走 proxy mode(SDK 文档未承诺支持,需自行包装)
3. iOS 端 SDK 会锁定到火山 TTS 协议层(`seed-tts-2.0`);当前 I15 收 Opus 字节流,与未来可换的 provider(自建流式 TTS / ElevenLabs V1.5)解耦
4. SDK 接入的工程量(I15.1 步骤 1 已估 0.4 dev-day 手写解码)与直接用手写解码几乎相同;但接入后多出"火山 SDK 升级 → 回归测试 → 包体增长 → 合规复审"的持续成本

**但本结论是条件性的**。若以下任一条件在 I15 启动前变为真,**应单独立项重新评估**:

- A. 火山官方明确支持「客户端 SDK + 自定义 WSS endpoint」模式(SDK 只负责解码,服务端由我方代理),且给出 Go/Objective-C 完整样例
- B. iOS 实测发现 Opus FEC 不够用,出现 P0 级别的卡顿 / 杂音 / 中断延迟(目前 I15 测试矩阵 T-I15-5 已用 Opus FEC 内置覆盖)
- C. iOS 端出现「音色切换」或「语速 / 情感控制」需求,且我方 gateway 无法在 WSS 帧里高效透传(SDK 内置 API 更顺手)
- D. B17 实施时发现服务端流式 TTS 延迟本身就有 200ms+ 抖动,iOS 端 jitter buffer 不够,SDK 自带的 buffer 调度算法成为性价比最优解

---

## 一、问题界定:与既有 71_/72_ 的边界

### 1.1 71_/72_ 已经回答了什么

`71_` 2026-09-03 结论:
> 当前已验收的主链路,**不切换、不追加官方 SDK**(指 Dialog 全双工 / 流式 ASR / 音频技术 SDK)

`72_` 2026-09-03 结论:
> 现阶段不值得为了"降低排查成本"把自研 relay 换成官方 Dialog / 流式 ASR 客户端 SDK

两份文档评估的对象是**全栈 SDK**:录音 + VAD + 录音识别 + TTS + 播放器全部托管,或其中至少录音 + 转写两段托管。结论的核心论证是:

| 论点 | 出处 |
|---|---|
| 主闭环需要服务端权威的 text + audio + confidence + turn metadata | `72_` §1、§3 |
| 客户端直连会破坏零 key 原则 | `71_` §4、72_ §2 |
| 历史故障分布在自己 turn/phase 状态机 + gateway↔vendor 网络后半段 + review 数据管线,SDK 只覆盖一小块 | `72_` §3 |
| B13 端侧 ASR 7 连败教训:hot path 上不应放平台/厂商不确定性 | `72_` §3 |

### 1.2 本次决策要回答的,71_/72_ 没覆盖

**本次问题的精确范围**:服务端主链路**保持现状**(voicegateway 中转、火山 duplex 持有 key、客户端零凭证),只在 iOS 客户端**复用 seed-tts-2.0 SDK 的 TTS 子集**,即:

- ✅ 客户端 audio session / AVAudioEngine 配置:仍是 iOS 手写
- ✅ Opus 解码 → PCM buffer:考虑是否替换为 SDK 内置解码
- ✅ AVAudioPlayerNode 调度 + 中断:考虑是否替换为 SDK 的播放器封装
- ✅ jitter buffer / 重采样:考虑是否替换为 SDK 自带
- ❌ 不改 voicegateway
- ❌ 不改 WSS 协议层(不引入客户端 SDK 的"直连"模式)
- ❌ 不引入新的凭证流转(凭证仍在后端)

### 1.3 这是 hybrid 路径,不是简单的"接不接 SDK"

71_/72_ 评估的是 A(不接)/B(全接)/C(只换 ASR)三选一。本文档评估的是第四种:

> **D. Hybrid 路径:服务端 relay 不变,客户端用 SDK 的 TTS 子集做解码 + 播放 + 中断**

这条路径在 71_/72_ 时**没有官方文档支持**——火山 iOS SDK 是否支持"自定义 WSS endpoint"模式(SDK 只负责客户端音频侧,不发起 WSS)是未知数。这也是本辩证文档要重点评估的不确定性。

---

## 二、客观事实:seed-tts-2.0 iOS SDK 的真实形态

### 2.1 SDK 客观信息(基于 2026-09-06 检索)

| 维度 | 事实 |
|---|---|
| 官方文档 | 火山引擎 → 豆包语音 → 「双向流式 TTS - iOS SDK 接口文档」(`docs.volcengine.com/docs/6561/...`) |
| 集成方式 | CocoaPods → `volcengine-specs`(`github.com/volcengine/volcengine-specs.git`),依赖 `TTNetworkManager` 等字节跳动网络栈 |
| 平行产品 | Android 端「双向流式 TTS - Android SDK」同期上线(依赖 BoringSSL、OkHttp、bytedance boringssl.so) |
| 鉴权 | 与服务端 WSS 鉴权同源:`X-Api-Key`(新版控制台口径) |
| SDK 名称里的「双向」含义 | **上行流式 text + 下行流式 audio**——SDK 既发也收;不是"全双工对话",而是"流式 API 上下行同时跑" |
| 「全双工对话 SDK」区别 | 那是 Dialog 端到端 SDK;含 VAD + turn 判定 + LLM hook。TTS SDK 不含这些 |
| 推测的 SDK 客户端组件 | ① WSS 连接管理 + BoringSSL ② 文本上行分片 + 流控 ③ Opus/PCM 解码器 ④ jitter buffer ⑤ 音频播放器封装 ⑥ 中断 / 恢复 API |
| 推测的可配置项 | voice_id / speed / volume / pitch / audio format(具体 API 形态需查 doc) |

> **注**:由于官方 docs.volcengine.com 页面需 JS 才能渲染,本节「推测」部分基于 Android SDK 平行实现 + CocoaPods 命名约定 + 历史 SDK 模式推断,**未拿到 iOS 头文件逐行确认**。B17 启动前应**实测确认 SDK 是否支持「自定义 WSS endpoint / 自定义会话管理」模式**(这是 hybrid 路径成立的前置条件,见 §三.2)。

### 2.2 seed-tts-2.0 与「豆包 duplex 全双工 TTS」的关系

| 形态 | 关系 |
|---|---|
| **seed-tts-2.0 作为 resource_id** | 火山 HTTP API 的产品级 SKU,服务端 HTTP/2 调用时填 `X-Api-Resource-Id: seed-tts-2.0`——这是 voicegateway 服务端要做的 |
| **seed-tts-2.0 作为 iOS 客户端 SDK 调用入口** | 客户端 SDK 把同样的 resource_id 封装成 `startSynthesis(text:voice:)` API,WSS 连接由 SDK 内部建立 |
| **火山 duplex 内置 TTS** | 走 `wss://openspeech.bytedance.com/api/v3/duplex/realtime/dialogue`,TTS 是"端到端对话"模型自带的输出,**音色固定**(72_ §3 已记录 B14 duplex 默认 voice `zh_female_vv_jupiter_bigtts`) |
| **D-2 拍板的 4 个音色** | `zh_male_tech_01` / `en_female_professional` / `en_male_narrator` / `en_female_clear`,这些是独立 TTS 的 voice_id,**不是 duplex 自带** |

**关键点**:D-2 拍板的 4 个音色属于**独立流式 TTS**(走 seed-tts-2.0 模型),不是 duplex 内置——所以 I15 的 TTS 播放链路从架构上**必须**走 B17 的独立 TTS Provider(不是 B14 的 duplex)。

这反过来证明了:**seed-tts-2.0 iOS SDK 在 hybrid 路径下,作用面就是「独立流式 TTS 输出」这一段,不是 duplex 全双工**。这与 I15 Issue Draft 完全对齐(I15 本来就是接 B17 的输出)。

---

## 三、支持引入的论点(FOR)

### 3.1 论点 1:SDK 自带 jitter buffer 算法通常优于手写

**论据**:
- 火山 SDK 团队针对自家 WSS 协议做了 jitter buffer 优化(抖动控制、缓冲水位自适应、首字加速)
- 手写 buffer 在弱网下需要大量调参经验,SDK 内置多年沉淀
- 减少 iOS 团队的「WSS 帧时序理解」负担

**反向风险**:我方实测目前没有发现「Opus FEC 不够用」的 P0 症状。I15 Issue Draft T-I15-5 已经标注「Opus FEC 内置 → 容忍丢 1 帧」,即假设"网络抖动"已经被 Opus FEC 解决;SDK jitter buffer 在 FEC 已覆盖的前提下,边际收益小。

### 3.2 论点 2:SDK 自带「中断 + 恢复」API 减少 iOS 状态机代码量

**论据**:
- I15 步骤 1 的 `TTSPlayer.interrupt()` 是手写 `playerNode.stop()` + 队列清空;SDK 可能提供 `cancel(sessionID:)` / `pause()` / `resume()` 之类的封装
- 减少 iOS team 在状态机调测上的时间

**反向风险**:
- iOS 状态机的中断语义与 voicegateway 的 turn 状态机强耦合(I15 步骤 3:用户打断 ≤ 100ms 静音,要求 `SpeechSessionMachine` 与 `ttsPlayer.interrupt()` 联动);SDK 封装的"中断"是只清客户端 queue 还是同时通知服务端?这与 B17 服务端协议设计有关,**SDK 中断语义不一定与现有服务端帧契约对齐**
- 当前 0.2 dev-day 的「打断兼容 + 状态机联动」成本已经包含手写中断的调测;SDK 接入后这部分成本不会消失,只会转化为「理解 SDK 中断 API + 适配服务端帧」的另一种成本

### 3.3 论点 3:SDK 自带多音色切换 / 情感控制 API

**论据**:
- D-2 拍板 4 音色,I15 Issue Draft 步骤 4 已预留 Setting → Voice Selection UI
- 切换音色 SDK 通常提供 `setVoice(id:)` 一行 API,手写需要重新建 session + 重发 WSS 配置

**反向风险**:
- 当前 voicegateway 设计:音色切换在 **服务端**完成(B17 §1.4 VoiceConfig 透传),客户端无需感知音色 ID 改变;切音色时客户端只看到流式 Opus 字节流变化
- 这是当前架构的**优势**,不是劣势:把音色配置收敛在 voicegateway,客户端保持纯数据流消费者,降低耦合
- 引入 SDK 反而要在客户端维护「音色 ID ↔ SDK 内部 voice 标识」的映射表,与服务端 VoiceConfig 重复

### 3.4 论点 4:减少我方对 Opus 解码细节的维护

**论据**:
- iOS Opus 解码可用 `libopus` 或 Apple 自带 `AVAudioConverter`;前者需引入 C 依赖,后者功能有限
- 火山 SDK 内置 Opus 解码器,**省去 iOS 引入 libopus 的包体成本与跨平台编译成本**

**反向风险**:
- `AVAudioConverter` 在 iOS 17+ 已可处理 Opus(Apple 自带的 Opus 支持起步晚但 17 已成熟);I15 步骤 1 选 `AVAudioConverter` 不引入第三方 C 依赖
- 若实测发现 `AVAudioConverter` 对火山输出 Opus 帧解码有兼容问题,这个论点才成立。**目前是理论风险,不是已发生的问题**

---

## 四、反对引入的论点(AGAINST)

### 4.1 论点 1:客户端 SDK 形态与零 key 架构根本性冲突

**核心论证**(继承自 71_/72_,但有 TTS 子集特定的展开):

seed-tts-2.0 SDK 是**客户端 SDK 形态**——它的设计假设是 SDK 在 iOS 进程里发起 WSS 连接、持有 API Key、收到服务端流式 audio 直接播放。

要在 hybrid 路径下使用,必须满足以下任一前提:

| 前提 | 是否成立 | 说明 |
|---|---|---|
| 客户端 SDK 支持「自定义 WSS endpoint」 | ⚠️ 未确认 | 官方 doc 渲染失败,**这是 hybrid 路径成立的前置条件**;若 SDK 不支持,hybrid 路径 = direct 路径 = 违反零 key 原则 |
| 客户端 SDK 支持「我只用解码器,不用你的 WSS client」 | ⚠️ 未确认 | 大概率 NO——SDK 是整合组件,不会单独拆出 decoder |
| 客户端 SDK 支持「我转发 WSS 流量,你只做客户端缓冲调度」 | ⚠️ 未确认 | 实现上需要我方在 voicegateway 维持 SDK 兼容的协议版本,且火山升级 SDK 时我方必须同步 |

**判断**:在没有官方文档确认上述前提前,hybrid 路径**实际上等同于 direct 路径**——一旦接入 SDK,iOS 端就会被推到「客户端直连火山」形态,零 key 架构不保。

**历史教训**(72_ §3 已记录):
> B13 曾尝试"客户端 Apple Speech 为主 ASR",真机联调 **7 连败**(`requiresOnDeviceRecognition` 不可用、授权时序、错误码不一致、401、audio session、端侧 STT 800ms、双 ASR 架构),最终 B14 改回 **server-side ASR relay 一次通过**。
>
> 官方客户端 SDK 同样是"把平台/厂商不确定性放在 hot path"。它可能比 Apple Speech 稳,但方向和 B13 的教训相反。

### 4.2 论点 2:历史故障多发层不在 iOS 解码层

`72_` §3 已统计近两周真实故障分布:

| 层 | 占比(粗估) | SDK 覆盖 |
|---|---|---|
| iOS 状态机 / UI | 高 | ❌ |
| gateway↔vendor 网络后半段(keepalive / broken pipe / turn 超时) | 高 | ❌(SDK 接入反而让网络层变成黑盒) |
| iOS 音频/VAD(句子截断 / audio session deactivation) | 中 | 部分(SDK 自带 VAD / session 管理) |
| review / 语料数据管线 | 中 | ❌ |
| 凭证 / 401 恢复 | 低 | 部分(SDK 自带 token 刷新) |
| **iOS 解码 / jitter buffer / 播放器调度** | **极低(目前无 P0 报告)** | ✅ |

**结论**:当前 P0/P1 几乎都不在 iOS 解码层。投入 I15 这次发射窗口换 SDK,**打不到主要故障源**。

### 4.3 论点 3:SDK 锁定火山协议层,削弱可替换性

| 维度 | 当前 I15 方案 | 引入 SDK 后 |
|---|---|---|
| iOS 收到的字节流 | Opus 帧(协议无关) | 火山 SDK 内部封装的对象 / 回调 |
| 换 TTS provider 成本 | 改 voicegateway 流式输出端,iOS 0 改动 | 改 iOS 端 SDK 调用,iOS + voicegateway 双改 |
| 升级火山 TTS 模型 | 改 voicegateway `voicepoc/volc_duplex.go` 等少数文件 | iOS 升级 SDK 版本 + voicegateway 升级 + 双向兼容测试 |
| 走 ElevenLabs V1.5 多音色 | voicegateway 加一个新 Provider,iOS 0 改动 | iOS 需要替换 SDK 调用,Voice Selection 同步改 |
| 包体影响 | 0(只用手写解码 + 系统 AVAudioEngine) | +3-8 MB(火山 iOS SDK 含 BoringSSL + Opus 解码器 + 业务代码) |

**判断**:当前 I15 方案把"协议无关性"做在 iOS 端,这是有价值的解耦,SDK 接入会把这层解耦拆掉。

### 4.4 论点 4:持续成本 vs 一次性成本不对等

**一次性成本(I15 阶段)**:
- 手写解码:0.4 dev-day(I15 步骤 1 估时)
- SDK 接入:至少 0.4-0.6 dev-day(下载 SDK + 跑通样例 + 适配 WSS 帧格式 + 中断语义对齐)
- 差距:**< 0.2 dev-day**

**持续成本(I15 之后)**:
- 手写方案:维护 ~200 行 Swift(Opus 解码 + AVAudioPlayerNode 调度),无第三方依赖,升 iOS SDK 时偶发小适配
- SDK 方案:
  - 火山 SDK 每次版本升级 → 回归测试 + 兼容性评估
  - 包体持续增长(火山 SDK 可能拉入完整 TTNetworkManager)
  - 火山若改 WSS 协议(已发生过,B14 D2 实测 `model=1.2.6.0` 与早期版本协议字段不同)→ 客户端同步升级,可能影响 I15 测试矩阵 T-I15-1/2/5
  - 合规复审:App Store 提交时含商业 SDK 需过更严审(目前 iOS team 走零商业 SDK 路径)

**判断**:持续成本与一次性成本**严重不对等**。一次性成本几乎相同,持续成本 SDK 方案高一个数量级。

### 4.5 论点 5:与 I15 DoD「Voice Selection UI」轻微冲突

I15 DoD 第 4 项「音色选择 UI + UserDefaults 持久化」,按 I15 Issue Draft 步骤 4 描述:
> 本 Issue 仅落 UI + UserDefaults;实际传入 backend 由 B17 后的下一个 sprint 实施

即 I15 阶段的音色选择 UI **不涉及任何 iOS 端 SDK 调用**——只把选择写入 UserDefaults,等 B17 启动日由 voicegateway 读取。如果引入 SDK,iOS 端就要立刻承担"把 voice_id 喂给 SDK"的实现,扩展了 I15 的 scope。

---

## 五、关键变量与判断依据

本决策的结论依赖于以下 6 个变量。任一变量改变,结论可能反转。

| 变量 | 当前判断 | 推翻结论的阈值 |
|---|---|---|
| V1. SDK 是否支持 hybrid 模式(自定义 WSS) | ⚠️ 未确认(应实测) | 若确认支持 → 重评 |
| V2. iOS 解码层是否有 P0 故障 | 无 | 出现 1 次 P0(Opus 卡顿 / 杂音 / 中断超时)→ 优先评估 SDK jitter buffer |
| V3. voicegateway ↔ 火山延迟抖动 | 待 B17 实测 | 若 P90 抖动 >300ms → SDK jitter buffer 价值上升 |
| V4. iOS 端音色 / 语速控制需求 | 弱(D-2 拍板固定 4 音色,语速服务端控制) | 若出现多音色预览 / 实时调速 UI → SDK 内置 API 价值上升 |
| V5. 火山 SDK 包体影响 | 未知 | 若 SDK >5MB → 与 ILS 启动门槛冲突(待 ILS 团队确认) |
| V6. 火山 WSS 协议稳定性 | 当前稳定(B14 D2 PASS) | 火山若频繁改协议 → SDK 升级成本上升,继续不接 |

---

## 六、四条候选路径 + 决策矩阵

### 6.1 候选路径

| 路径 | 描述 | I15 工作量 | 持续成本 | 风险 |
|---|---|---|---|---|
| **A. 现状(手写 I15)** | iOS 手写 Opus 解码 + AVAudioEngine + playerNode | 0.4 dev-day | 低 | 低 |
| **B. 接入 seed-tts-2.0 SDK(hybrid)** | 客户端 SDK 复用 + voicegateway 中转(假设 SDK 支持) | 0.6 dev-day | 中-高 | 中-高(SDK 是否真支持 hybrid 未确认) |
| **C. 接入 SDK(direct 模式)** | iOS 直连火山,voicegateway 不参与 TTS | 0.4 dev-day + 大改 voicegateway | 高 | 极高(违反零 key) |
| **D. 等 B17 实测后再说** | I15 按 A 推进,B17 上线后观察 voicegateway 延迟 + iOS 端故障,再做数据驱动的二次评估 | 0.4 dev-day(I15) | 低 | 低(数据不够 → 暂时保持现状) |

### 6.2 加权决策矩阵

| 维度 | 权重 | A(手写) | B(SDK hybrid) | C(SDK direct) | D(B17 后评) |
|---|---|---|---|---|---|
| 架构一致性(零 key) | 25 | 10 | 5 | 0 | 10 |
| P0 故障覆盖率 | 20 | 8 | 5 | 2 | 7 |
| iOS 团队维护成本 | 15 | 7 | 5 | 3 | 7 |
| 持续成本(TCO) | 15 | 9 | 4 | 2 | 8 |
| 可替换性(多 provider) | 10 | 10 | 4 | 2 | 9 |
| I15 时间窗契合度 | 10 | 9 | 7 | 5 | 9 |
| 风险可逆性 | 5 | 10 | 6 | 2 | 9 |
| **加权总分(满分 100)** | **100** | **87** | **50** | **18** | **86** |

**结论**:A(现状手写)与 D(等 B17 后评)几乎并列;B(SDK hybrid)显著落后;C(SDK direct)被否决。

**A vs D 的取舍**:
- A:已拍板,直接推进 I15
- D:多走一次 B17 实测,但 B17 启动日(2026-09-10)与 I15 启动日(W3 Day 3)相差 ≤ 3 天,等待收益边际
- **建议采用 A**,把 D 作为"如果 B17 实测发现 I15 不够用 → 立即评估 SDK"的回退方案

---

## 七、结论与触发条件

### 7.1 主结论

**I15 阶段不引入 seed-tts-2.0 iOS SDK**。按 A 路径推进,沿用 I15 Issue Draft 已规划的手写 Opus 解码 + AVAudioEngine + playerNode 调度 + 中断语义。

### 7.2 触发条件(任一命中即重评)

| 触发条件 | 重评方向 | 优先级 |
|---|---|---|
| TC-1:火山官方确认 seed-tts-2.0 iOS SDK 支持 hybrid 模式(自定义 WSS endpoint / 客户端只做解码) | 重新计算路径 B 的得分 | P1 |
| TC-2:iOS 端出现 Opus 解码 P0(卡顿 / 杂音 / 中断超时 >500ms) | 优先评估 SDK 的 jitter buffer 是否能覆盖(可只接入解码器,不全接 SDK) | P0 |
| TC-3:B17 实测 voicegateway 流式 TTS 延迟抖动 P90 >300ms | 评估 SDK jitter buffer 算法,作为 voicegateway 抖动补偿的替代方案 | P1 |
| TC-4:火山 WSS 协议在 I15 上线后频繁变动 | SDK 接入成本上升,**反而强化不接 SDK 的结论** | N/A |
| TC-5:V1.5 引入 ElevenLabs 多音色 | 引入"客户端 SDK 中间抽象层"(自研,不依赖火山 SDK),封装多 provider | P2 |
| TC-6:iOS 端出现"实时调速 / 实时切音色"交互(用户拖 slider) | SDK 内置 API 价值上升,但首要看服务端 voicegateway 能否支持 | P2 |

### 7.3 与既有决策的协调

- 不修改 `71_` `72_` 的结论(全 SDK 替换 → 否;端侧 ASR → 否)
- 不修改 I15 Issue Draft 的实施步骤(本结论即"按 I15 Issue Draft 原计划推进")
- 在 `62_` 中已加入「架构选型澄清」段,记录"为什么不接 SDK"的 rationale——本文档是该段的上游详细论证
- 同步 `53_` §Layer 1 / `74_` §C-2 表格,不调整 B17 的实施计划(B17 是服务端实施,与本决策正交)

---

## 八、证据索引

### 8.1 项目内文档

- `fluentwork-meta/docs/70_外部研究/71_豆包语音SDK与FluentWork自研链路对比分析.md`——SDK 能力盘点
- `fluentwork-meta/docs/70_外部研究/72_豆包语音SDK接入决策评估_PRD主闭环视角.md`——全 SDK 决策评估
- `fluentwork-meta/docs/30_技术方案/34_FluentWork火山引擎选型与开通清单.md`——火山选型与产品级 SKU 命名规范
- `fluentwork-meta/docs/30_技术方案/37_FluentWork-B14_Client_ASR_Relay_Architecture.md`——B14 架构决策(iOS 不持 key)
- `fluentwork-meta/docs/40_研发流程与协作/47_D1_D5_开放技术决策备忘录_2026-09-03.md` §D-2——4 音色拍板
- `fluentwork-meta/docs/40_研发流程与协作/53_FluentWork_V2_0启动前待办清单_2026-09-03.md` §四 C-2——TTS 凭证状态
- `fluentwork-meta/docs/40_研发流程与协作/56_B17_TTS_Provider_Issue_Draft_2026-09-06.md`——B17 实施包
- `fluentwork-meta/docs/40_研发流程与协作/62_I15_TTS播放集成_Issue_Draft_2026-09-06.md`——I15 实施包 + 架构澄清段
- `fluentwork-meta/docs/40_研发流程与协作/74_V2_0_凭证与P0修复验证日志_2026-09-06.md` §2.5——TTS 凭证复跑
- `fluentwork-backend/internal/voicegateway/provider_volc_duplex.go`——keepalive / turn 状态机
- `fluentwork-backend/internal/voicegateway/config.go`——gateway 配置(Volc Speech API Key 等)
- `fluentwork-ios/docs/13_ClientASR集成与使用指南.md`——B13→B14 演进史
- `fluentwork-ios/docs/20_I20_voice_turn_boundary_pitfalls.md`——turn 边界历史故障

### 8.2 外部资料

- 火山引擎 → 豆包语音 → 双向流式 TTS - iOS SDK 接口文档:`https://docs.volcengine.com/docs/6561/1597646`(页面需 JS 渲染,本文 §2.1「推测」部分基于平行 Android SDK + CocoaPods 命名约定推断,**B17 启动前应实测确认**)
- 火山引擎 → 豆包语音 → 双向流式 TTS - Android SDK 接口文档:`https://docs.volcengine.com/docs/6561/...`(平行产品,作 SDK 形态参考)
- 火山引擎 → CocoaPods specs:`https://github.com/volcengine/volcengine-specs.git`(TTNetworkManager 依赖)
- 火山引擎 → GitHub Org:`https://github.com/volcengine`
- 第三方 Go 协议参考(非官方):`https://github.com/GizClaw/doubao-speech-go/blob/main/docs/realtime_duplex.md`(用于理解 WSS 协议层)

### 8.3 本文档待补(随决策演化)

- [ ] 实测确认 seed-tts-2.0 iOS SDK 是否支持 hybrid(自定义 WSS endpoint)模式——B17 启动前必须完成
- [ ] 实测 voicegateway 流式 TTS 延迟抖动 P90——B17 上线后 3 日内出数据
- [ ] iOS Opus 解码实测结果(I15 T-I15-1/2/5 自动化)——I15 上线后回归测试日志

---

## 九、修订记录

| 版本 | 日期 | 修改 | 作者 |
|---|---|---|---|
| V1.0 | 2026-09-06 | 初版;基于 §二 客观事实 + §三/四 论点 + §六 决策矩阵 | 72_ 框架继承 |
