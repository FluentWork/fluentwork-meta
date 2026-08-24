# FluentWork 第一波重点 Issue 二级拆分草案

**版本**：V1.0
**日期**：2026-08
**定位**：把第一波中最容易过大的 backend / iOS issues 继续下钻为二级子任务，供 GitHub checklist、repo 内子 issue 或 Matt 风格 tools 继续细化使用
**上游依据**：`46_FluentWork-backend第一波Issue草案.md`、`47_FluentWork-ios第一波Issue草案.md`、`31_FluentWork后端技术方案文档.md`、`32_FluentWork-iOS App端技术设计文档.md`
**变更说明**：将 B3、B5、I3、I5 四个重点 issue 继续拆解为可执行子任务

---

## 一、使用说明

这份文档不新增 `meta` Epic，也不替代 repo issue。

它的用途是：

1. 作为 GitHub issue 正文里的 checklist
2. 作为 repo 内继续建子 issue 的底稿
3. 作为 Matt 风格 `to tickets` / task breakdown 的输入材料

统一原则：

1. 一级 issue 仍然是 `backend` / `ios` 仓的真源
2. 二级拆分服务于执行，不改变一级 issue 边界
3. 若二级项已经足够小，可以直接作为 PR 实施步骤，不必再继续拆

---

## 二、Backend：B3 二级拆分

原 issue：

`B3: scaffold voice-gateway WSS protocol`

推荐二级任务：

### B3.1 握手与 ticket 校验骨架

目标：

1. 接入 WSS server
2. 校验 ticket
3. 在握手失败时返回清晰错误

验收点：

1. 非法 ticket 被拒绝
2. 过期 ticket 被拒绝
3. 合法 ticket 可完成最小建连

### B3.2 控制帧 schema 定义

目标：

1. 定义 `session.start`
2. 定义 `interrupt`
3. 定义 `session.end`
4. 定义最小反馈 / 状态帧

验收点：

1. 字段名与契约文档一致
2. 可序列化 / 反序列化
3. 非法字段有错误处理路径

### B3.3 音频帧 schema 与收发边界

目标：

1. 定义二进制音频帧格式
2. 明确序列号字段
3. 明确客户端 / 网关各自职责

验收点：

1. 客户端能发送最小音频帧
2. 网关能识别并转发
3. 后续可承接序列号丢帧规则

### B3.4 ping / reconnect 最小支持

目标：

1. 增加 ping 机制
2. 预留 reconnect 恢复入口
3. 不在第一波实现完整恢复状态机

验收点：

1. 心跳超时可识别
2. 最小 reconnect 流程有接口或占位
3. 不阻塞主链路建连

### B3.5 协议契约测试

目标：

1. 为关键控制帧建立 schema 校验
2. 为握手和非法 ticket 建最小测试

验收点：

1. 握手成功 / 失败路径有测试
2. 关键控制帧字段有测试
3. PR 评审时不需要靠人工比对 schema

建议 checklist：

```md
- [ ] B3.1 握手与 ticket 校验骨架
- [ ] B3.2 控制帧 schema 定义
- [ ] B3.3 音频帧 schema 与收发边界
- [ ] B3.4 ping / reconnect 最小支持
- [ ] B3.5 协议契约测试
```

---

## 三、Backend：B5 二级拆分

原 issue：

`B5: scaffold review worker pipeline`

推荐二级任务：

### B5.1 `session.finished` 事件投递

目标：

1. 在 `session.end` 后投递最小事件
2. 带上 `session_id` 与必要上下文

验收点：

1. 会话结束会触发事件
2. 重复结束事件不会重复投递脏数据

### B5.2 Worker 消费骨架

目标：

1. 建立 worker 消费入口
2. 能拉取并解析最小任务

验收点：

1. worker 能启动
2. 能消费到任务
3. 异常任务不会拖死消费循环

### B5.3 幂等键与任务状态

目标：

1. 以 `session_id + task_type` 作为幂等键
2. 明确 pending / processing / ready / failed 状态

验收点：

1. 重复消费不产生重复结果
2. 状态流转清晰

### B5.4 一次失败重试策略

目标：

1. 失败后重试一次
2. 二次失败进入 failed

验收点：

1. 重试次数受控
2. 不出现无限重试
3. 失败信息可被 review 查询接口消费

### B5.5 最小 review 结果写回

目标：

1. worker 能写回最小 review 数据或状态
2. 为 `GET /sessions/:id/review` 提供输入

验收点：

1. review ready 后可查询
2. review failed 后也可查询状态

建议 checklist：

```md
- [ ] B5.1 `session.finished` 事件投递
- [ ] B5.2 Worker 消费骨架
- [ ] B5.3 幂等键与任务状态
- [ ] B5.4 一次失败重试策略
- [ ] B5.5 最小 review 结果写回
```

---

## 四、iOS：I3 二级拆分

原 issue：

`I3: add socket transport and frame decoding`

推荐二级任务：

### I3.1 WSS 建连封装

目标：

1. 封装 WebSocket 连接入口
2. 处理 connect / disconnect 最小状态

验收点：

1. 可对接 backend WSS 地址
2. 连接失败可上抛

### I3.2 控制帧编解码

目标：

1. 建立 Codable 控制帧模型
2. 支持关键控制帧收发

验收点：

1. schema 与 backend 一致
2. 编解码失败路径可见

### I3.3 音频帧封装

目标：

1. 定义最小音频帧模型
2. 为 AudioEngine 对接留边界

验收点：

1. 可发送最小音频帧
2. 可接收最小音频帧

### I3.4 序列号丢帧纯函数

目标：

1. 按技术文档实现丢帧规则
2. 独立为纯函数

验收点：

1. 有单测
2. 覆盖相等、过期、空队列等边界

### I3.5 ping / reconnect 上抛到 Store

目标：

1. 心跳失败能触发上层状态
2. reconnect 结果能传到 Store / middleware

验收点：

1. failure 可上抛
2. reconnect 不把 transport 逻辑耦死在 View 层

建议 checklist：

```md
- [ ] I3.1 WSS 建连封装
- [ ] I3.2 控制帧编解码
- [ ] I3.3 音频帧封装
- [ ] I3.4 序列号丢帧纯函数
- [ ] I3.5 ping / reconnect 上抛到 Store
```

---

## 五、iOS：I5 二级拆分

原 issue：

`I5: scaffold speaking room screen`

推荐二级任务：

### I5.1 页面骨架与状态区布局

目标：

1. 建立房间页最小布局
2. 预留消息区、输入区、状态区

验收点：

1. 页面能承载主链路
2. 不直接访问底层 service

### I5.2 说话键与房间状态切换

目标：

1. 让说话键只发 action
2. 让页面能反映 connecting / waiting / recording / failed 等状态

验收点：

1. View 只读 state
2. 状态切换不直接耦合底层实现

### I5.3 转录浮层与消息占位

目标：

1. 提供用户转录浮层占位
2. 提供 AI 消息区占位

验收点：

1. 至少能展示最小文本状态
2. 不追求第一波视觉精修

### I5.4 loading / error / retry 最小错误态

目标：

1. 让连接失败、会话创建失败、降级等状态有最小 UI 承载

验收点：

1. 错误态可见
2. retry 行为有明确 action

### I5.5 说的房间最小交互走查

目标：

1. 手动走查主状态流转
2. 确认不出现 View 直接依赖 transport / audio

验收点：

1. 主状态至少可走到连接、等待、录音、失败
2. 结构符合技术文档禁区

建议 checklist：

```md
- [ ] I5.1 页面骨架与状态区布局
- [ ] I5.2 说话键与房间状态切换
- [ ] I5.3 转录浮层与消息占位
- [ ] I5.4 loading / error / retry 最小错误态
- [ ] I5.5 说的房间最小交互走查
```

---

## 六、这里还要不要再用 Matt 风格 tools 继续拆

当前结论：

1. **不是必须**
2. **但很适合用于这四个 issue**

最适合继续喂给它的输入就是：

1. `B3`
2. `B5`
3. `I3`
4. `I5`

推荐用法：

1. 先保留一级 issue 不变
2. 把本文件里的二级 checklist 贴进 issue
3. 再用 Matt 风格 tools 只对其中某一项继续拆 PR 级 tasks

不推荐：

1. 重新改写一级 issue 标题
2. 重新改写 `meta` Epic 边界
3. 在没有同步 `meta` 的情况下扩张范围

---

## 七、最终口径

第一波继续往下拆时，最合理的方式是：

1. `meta` 保持 Epic 真源不变
2. repo issue 保持一级边界不变
3. 对 B3、B5、I3、I5 使用二级拆分
4. 只有当二级任务仍然过大时，才继续用 Matt 风格 tools 下钻到 PR 级任务
