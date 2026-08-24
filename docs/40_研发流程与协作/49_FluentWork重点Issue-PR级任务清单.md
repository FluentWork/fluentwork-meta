# FluentWork 重点 Issue PR 级任务清单

**版本**：V1.0
**日期**：2026-08
**定位**：将第一波四个重点 issue 继续下钻到 PR 级任务，形成可直接执行的最小提交单元建议
**上游依据**：`48_FluentWork第一波重点Issue二级拆分草案.md`、`46_FluentWork-backend第一波Issue草案.md`、`47_FluentWork-ios第一波Issue草案.md`
**变更说明**：把 B3、B5、I3、I5 继续拆为 PR 级任务清单，便于实际编码和逐 PR 推进

---

## 一、使用说明

本文件的目标不是再改 Epic 或一级 issue，而是：

1. 为一级 issue 提供建议的 PR 切分顺序
2. 控制每个 PR 的改动范围
3. 让评审、测试和回滚边界更清晰

使用规则：

1. 一个 PR 应尽量只覆盖一个 PR 级任务
2. PR 标题可以直接复用这里的标题建议
3. PR 正文应回链到一级 issue，而不是直接回链到 `meta` Epic
4. 若某个 PR 级任务仍然过大，再继续往下拆，但不要跨越一级 issue 边界

---

## 二、Backend：B3 的 PR 级任务

原一级 issue：

`B3: scaffold voice-gateway WSS protocol`

建议拆为 4 个 PR。

### B3-PR1：搭建 gateway 握手与 ticket 校验骨架

建议标题：

`backend: scaffold gateway handshake and ticket validation`

范围：

1. 搭建 WSS server 入口
2. 最小连接生命周期
3. ticket 校验骨架
4. 非法 / 过期 ticket 错误返回

不包含：

1. 完整控制帧 schema
2. 音频帧处理
3. reconnect

验收重点：

1. 客户端可以用合法 ticket 建连
2. 非法 ticket 可被拒绝
3. 错误返回可被 iOS 区分处理

### B3-PR2：落控制帧模型与最小消息收发

建议标题：

`backend: add gateway control frame models`

范围：

1. `session.start`
2. `interrupt`
3. `session.end`
4. 最小状态 / 反馈控制帧

不包含：

1. 音频帧二进制处理
2. 完整 reconnect 策略

验收重点：

1. schema 字段与契约文档一致
2. 能完成最小控制消息收发
3. 编解码失败路径清晰

### B3-PR3：接入音频帧格式与转发边界

建议标题：

`backend: add gateway audio frame transport boundary`

范围：

1. 二进制音频帧格式
2. 序列号字段
3. 最小转发边界

不包含：

1. 完整音频业务语义
2. review 持久化

验收重点：

1. 网关能识别并转发最小音频帧
2. 客户端后续可接入序列号丢帧规则

### B3-PR4：补心跳与协议契约测试

建议标题：

`backend: add gateway heartbeat and protocol tests`

范围：

1. ping 机制
2. 最小 reconnect 占位
3. 握手 / schema 契约测试

验收重点：

1. 心跳超时能识别
2. 协议关键路径有测试
3. 不靠人工对比 schema

---

## 三、Backend：B5 的 PR 级任务

原一级 issue：

`B5: scaffold review worker pipeline`

建议拆为 4 个 PR。

### B5-PR1：投递 `session.finished` 事件

建议标题：

`backend: emit session finished events`

范围：

1. `session.end` 后投递事件
2. 带最小上下文
3. 防重复脏投递

验收重点：

1. 正常结束能投递
2. 重复结束不会无限重复制造事件

### B5-PR2：搭建 worker 消费骨架

建议标题：

`backend: scaffold review worker consumer`

范围：

1. worker 启动入口
2. 拉取任务
3. 解析最小消息

验收重点：

1. worker 能消费到任务
2. 异常任务不会拖死主循环

### B5-PR3：补任务状态机与幂等键

建议标题：

`backend: add review task idempotency and states`

范围：

1. `session_id + task_type` 幂等键
2. pending / processing / ready / failed
3. 重复消费保护

验收重点：

1. 状态流转清晰
2. 重复消费不产生重复结果

### B5-PR4：补失败重试与最小结果写回

建议标题：

`backend: add review retry policy and result persistence`

范围：

1. 一次失败重试
2. failed 状态落库
3. ready / failed 最小结果写回

验收重点：

1. 重试次数受控
2. `GET /sessions/:id/review` 后续可直接消费这些状态

---

## 四、iOS：I3 的 PR 级任务

原一级 issue：

`I3: add socket transport and frame decoding`

建议拆为 4 个 PR。

### I3-PR1：封装 WSS 建连与生命周期

建议标题：

`ios: scaffold socket transport connection lifecycle`

范围：

1. WebSocket 建连入口
2. connect / disconnect 最小状态
3. 错误上抛

不包含：

1. 控制帧模型
2. 音频帧
3. 丢帧规则

验收重点：

1. 可连接 backend WSS 地址
2. 失败可上抛到上层

### I3-PR2：补控制帧编解码

建议标题：

`ios: add socket control frame codecs`

范围：

1. Codable 控制帧模型
2. 关键控制帧编解码
3. 编解码错误处理

验收重点：

1. 与 backend schema 一致
2. 控制消息收发可跑通

### I3-PR3：补音频帧封装与传输边界

建议标题：

`ios: add socket audio frame transport boundary`

范围：

1. 最小音频帧模型
2. 上下行二进制边界
3. 为 AudioEngine 对接留接口

验收重点：

1. 能发送 / 接收最小音频帧
2. 不把 AudioEngine 细节耦进 transport

### I3-PR4：补序列号丢帧规则与心跳上抛

建议标题：

`ios: add frame discard rules and heartbeat state propagation`

范围：

1. 序列号丢帧纯函数
2. 边界单测
3. ping / reconnect 结果上抛到 Store

验收重点：

1. 丢帧规则有测试
2. reconnect 信号不被困在 transport 内部

---

## 五、iOS：I5 的 PR 级任务

原一级 issue：

`I5: scaffold speaking room screen`

建议拆为 4 个 PR。

### I5-PR1：搭房间页布局骨架

建议标题：

`ios: scaffold speaking room layout`

范围：

1. 页面最小布局
2. 消息区 / 状态区 / 输入区占位

不包含：

1. 复杂交互
2. 最终视觉打磨

验收重点：

1. 页面骨架稳定
2. View 不直接依赖底层 service

### I5-PR2：接房间状态显示与说话键 action

建议标题：

`ios: wire speaking room state presentation`

范围：

1. 说话键只发 action
2. connecting / waiting / recording / failed 状态可见

验收重点：

1. View 只读 state
2. 状态显示能跟随 Store 变化

### I5-PR3：补转录浮层与消息占位

建议标题：

`ios: add speaking room transcript overlay and message placeholders`

范围：

1. 用户转录浮层
2. AI 消息区占位
3. 最小文本显示

验收重点：

1. 交互流可见
2. 不追求第一波视觉精修

### I5-PR4：补 loading / error / retry 最小错误态

建议标题：

`ios: add speaking room loading and retry states`

范围：

1. loading 态
2. failed 态
3. retry action 路径

验收重点：

1. 主链路失败时有明确 UI
2. retry 行为能上抛到 Store

---

## 六、推荐执行顺序

如果你准备开始真正开工，建议按下面顺序：

1. `B3-PR1`
2. `I3-PR1`
3. `B3-PR2`
4. `I3-PR2`
5. `B3-PR3`
6. `I3-PR3`
7. `I5-PR1`
8. `I5-PR2`
9. `B5-PR1`
10. `B5-PR2`
11. `B5-PR3`
12. `B5-PR4`
13. `I5-PR3`
14. `I5-PR4`
15. `I3-PR4`
16. `B3-PR4`

这条顺序的核心是：

1. 先把 backend / iOS 的协议边界两边都站起来
2. 再让 UI 消费最小状态
3. 最后补 review 异步闭环与错误态

---

## 七、这里还要不要再用 Matt 风格 tools

结论：

1. 现在已经**可以不用**
2. 但如果你想继续把某个 PR 再细成“实现步骤清单”，它仍然有价值

最适合继续喂给它的是：

1. `B3-PR2`
2. `B3-PR4`
3. `I3-PR4`
4. `I5-PR4`

因为这几个 PR 相对更容易混入测试、状态传播和错误路径，继续细化通常更有帮助。

---

## 八、最终口径

第一波真正进入编码时，建议按：

1. `meta` Epic
2. repo 一级 issue
3. PR 级任务

这三层推进。

只有当某个 PR 级任务仍然过大时，才继续向下拆分到“提交步骤”级别。
