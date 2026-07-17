---
version: 0715
module: agent-runtime
feature_type: functional
feature_id: FEAT-2026-009
status: active
---
# Agent-runtime 组件调用端侧工具响应特性文档

## 1. 特性定位

FEAT-2026-009 定义 `agent-runtime` 当前版    本支持“带有端侧工具的智能体请求”的服务端响应事实：当 Agent 执行过程中需要调用客户端侧工具、页面能力、终端插件或人工确认时，runtime 必须把该需求作为 A2A Task 的可观察中断状态返回给客户端，并在客户端主动提交结果后恢复原 Task。

本特性解决的问题是：业务应用通过 `agent-client` 发起一次智能体服务调用后，服务端 Agent 可能在执行中需要客户端本地能力参与。

例如读取当前页面上下文、触发终端插件、要求用户确认某个高风险动作。runtime 不能直接访问客户端资源，也不能把等待客户端工具的状态伪装成已完成。runtime 必须通过 A2A 响应、SSE 事件、Task 查询投影或受治理 Gateway / Event Bus 投影告诉客户端“当前 Task 需要端侧工具结果”，由客户端执行后再把结果提交回 runtime。

在总体架构中，本特性位于 `agent-runtime` 的 A2A 服务入口、Agent 执行过程和 `agent-client` 本地能力执行之间。

runtime 是服务端 Task owner，负责把端侧工具需求投影为 Task 中断、接收客户端结果、校验结果并恢复 Task；

`agent-core` 负责在 Agent 执行中识别或产生端侧工具调用移交意图，具体由 FEAT-2026-010 承接；

`agent-client` 负责本地能力声明、执行和结果提交，具体由 FEAT-2026-007 承接；Gateway / Event Bus 只负责受治理转发和投影交付，不拥有 Task 状态。

本特性面向以下角色：

- 业务应用开发者：理解一次智能体调用可能返回“需要客户端工具结果”的中断状态，并在客户端完成工具执行后继续任务。
- Runtime 模块开发者：实现端侧工具请求的 A2A Task 响应、结果接收、校验和 Task 恢复语义。
- agent-core 集成方：把 Agent 执行中的端侧工具调用移交给 runtime，而不是直接执行客户端工具。
- Gateway / IngressGateway 开发者：透传端侧工具请求和客户端结果，不改变 Task owner。
- agent-client 集成方：消费端侧工具请求，执行本地能力并主动提交结果。
- 测试与验收团队：验证 Client、Gateway、runtime 基于 A2A 协议形成端侧工具调用闭环。

本特性只定义 runtime 侧“端侧工具请求响应与结果恢复”的外部行为。客户端本地工具注册和执行由 FEAT-2026-007 定义；agent-core 任务粒度动态工具可见性和调用移交由 FEAT-2026-010 定义；Gateway 路由和总线投影由 FEAT-011、FEAT-012、FEAT-013 和 FEAT-017 定义；runtime 标准 A2A 服务入口由 FEAT-001 定义。

## 2. 当前版本能力要求

| 能力             | 要求级别 | 事实要求                                                                                                       |
| ---------------- | -------- | -------------------------------------------------------------------------------------------------------------- |
| 端侧工具请求响应 | MUST     | runtime 必须能在 Agent 执行需要客户端侧工具时，向 client 返回 A2A Task 可观察的等待状态或等价响应。            |
| A2A 中断语义     | MUST     | 等待客户端工具结果时，Task 不得被标记为 completed；必须表现为等待输入、等待能力结果或等价可恢复状态。          |
| Client 可见投影  | MUST     | 端侧工具请求必须能通过阻塞响应、SSE、Task 查询或 Gateway / Event Bus 投影被 client 观察到。                    |
| 结果提交入口     | MUST     | client 执行本地工具后，必须能通过受治理 C/S 通道（A2A）把结果、拒绝或错误提交回 runtime。                      |
| Task 恢复        | MUST     | runtime 接收并校验客户端结果后，必须恢复原 Task，并由 Agent 执行逻辑决定继续、完成、失败或再次等待。           |
| 客户端资源边界   | MUST     | runtime 不直接访问客户端 DOM、插件、文件、本地端口或业务 UI。                                                  |
| Core 边界        | MUST     | runtime 接收 core 产生的端侧工具调用移交意图，但不在本特性中定义 core 的动态工具目录、策略裁剪或模型工具注入。 |

## 3. 外部接口与入口要求

| 入口                     | 类型                         | 事实要求                                                                                                 |
| ------------------------ | ---------------------------- | -------------------------------------------------------------------------------------------------------- |
| A2A 调用响应             | runtime -> Gateway / client  | 当执行需要端侧工具结果时，响应必须表达 Task 正在等待客户端侧工具输入，不得伪装成完成结果。               |
| A2A SSE / Task 投影      | runtime -> Gateway / client  | 流式调用或异步任务必须能把端侧工具请求投影给 client，包含工具请求说明、所需输入和结果提交关联信息。      |
| Task 查询                | client -> Gateway -> runtime | client 查询 Task 时，必须能看到当前 Task 是否正在等待端侧工具结果。当前runtime只支持根据TaskID进行查询。 |
| 客户端结果提交           | client -> Gateway -> runtime | client 完成本地工具执行后，必须能提交成功结果、用户拒绝或执行错误，runtime 负责Task恢复与 执行。         |
| Gateway / Event Bus 转发 | 受治理交付路径               | Gateway / Event Bus 只负责请求、投影和结果的交付，不解释客户端工具业务含义，不改变 Task owner。          |

## 4. 场景与用户旅程

| 场景                          | 前置条件                                                                 | 用户/系统动作                                                     | 期望行为                                                                                                                                                                                                                                                                                                     |
| ----------------------------- | ------------------------------------------------------------------------ | ----------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 端侧工具请求随调用返回        | 业务应用通过 agent-client 发起智能体调用，Agent 执行中需要客户端本地能力 | runtime 从 Agent 执行过程中收到端侧工具调用移交意图               | runtime 不完成 Task，而是通过 A2A 响应或 SSE 投影返回“等待端侧工具结果”；<br />Gateway 透传该状态；client 展示或调度对应本地能力。                                                                                                                                                                         |
| 客户端执行本地工具并恢复 Task | client 已收到端侧工具请求，且本地能力可用                                | client 执行页面能力、终端插件或人工确认，并把结果提交给 Gateway   | Gateway 将结果转发到 runtime；runtime 校验该结果属于当前 Task 后恢复 Agent 执行；Task 可继续运行、完成、失败或再次请求端侧工具。                                                                                                                                                                             |
| 用户拒绝或本地工具失败        | 本地工具需要用户确认，或客户端执行失败                                   | client 提交用户拒绝、权限不足、能力不可用或执行失败（工具侧触发） | 地方runtime 将该结果作为客户端侧响应处理，不直接伪造成功；<br />Agent 按照真实的后端模型处理返回结果；                                                                                                                                                                                                       |
| 流式调用中的端侧工具中断      | client 使用流式观察，Agent 执行中需要端侧工具                            | runtime 通过 SSE / Task 投影发出等待端侧工具结果状态              | Gateway 只桥接或投影状态，不承载客户端工具执行；<br />client 按请求执行本地能力并提交结果后，runtime 继续原 Task 的流式执行或返回终态。**特别说明：**<br />工具调用失败后，因三方工具调用产生的问题，由Client将错误信息+taskID返回给runtime进行处理。完全遵守Agent业务逻辑，如更换工具或者结束流程等。 |

### 4.1 端到端闭环说明

1. Client 通过 `agent-client` 发起 A2A 兼容调用，请求经 Gateway 进入 runtime。
2. runtime 调用 Agent 执行；当 Agent 需要端侧工具时，core 或框架适配层只产生调用移交意图，不执行客户端工具。
3. runtime 将该意图转化为 Task 可观察的等待状态，通过 A2A 响应、SSE、Task 查询投影或总线投影返回给 Gateway / client。
4. client 根据该请求执行本地能力，或由用户确认、拒绝。
5. client 通过受治理 C/S 通道提交结果；Gateway 只转发结果。
6. runtime 校验结果与当前 Task 的关联关系，恢复 Agent 执行，并继续产生 Task 状态、SSE 流或终态响应。

## 5. 行为语义与边界

### 5.1 核心行为语义

#### 5.1.0 Task 中断语义

- 端侧工具请求是 Task 执行过程中的可恢复等待状态，不是完成，也不是普通失败。
- client 和 Gateway 只能观察和提交结果，不能伪造 runtime 内部状态。
- 同一 Task 可以在多轮执行中多次等待端侧工具结果，每次都必须能与对应客户端提交结果关联。

#### 5.1.1 Client / Gateway / runtime 分工

- Client 负责展示端侧工具请求、执行本地能力、提交结果或拒绝。
- Gateway 负责认证、路由、协议桥接和投影转发，不拥有 Task 状态。
- runtime 负责 Task owner 语义、等待状态投影、结果校验、恢复执行和终态决定。
- core 负责在 Agent 执行中产出端侧工具调用移交意图，不负责 A2A Task 状态和客户端执行。

#### 5.1.2 结果处理语义

- 客户端结果是外部输入，runtime 必须校验后才能恢复 Task。
- 用户拒绝、权限不足、能力不可用和执行失败都必须有清晰的客户端可见状态。
- runtime 恢复 Task 后，Agent 可以继续执行、再次等待端侧工具、完成或失败。

#### 5.1.3 错误、状态与可观测结果

| 场景                 | 事实要求                                                                                               |
| -------------------- | ------------------------------------------------------------------------------------------------------ |
| 端侧工具请求生成失败 | runtime 返回明确错误或使 Task 进入 failed。                                                            |
| 客户端拒绝           | runtime 接收拒绝结果，并由 Agent 执行逻辑决定降级、失败或返回说明。                                    |
| 客户端工具不可用     | runtime 接收不可用结果，不把该状态伪造成成功。                                                         |
| Task 已终态          | 后续客户端工具结果不得重新推进该 Task。（A2A默认能力）                                                 |
| client 连接中断      | Task 状态仍由 runtime 保持；<br />client 可通过 Task 查询或重新观察获取当前状态。<br />（A2A默认能力） |

### 5.2 显式边界与不承诺项

| 边界                       | 当前版本不承诺                                                   |
| -------------------------- | ---------------------------------------------------------------- |
| 客户端真实执行             | runtime 不执行客户端本地工具，不访问 DOM、插件、文件或本地端口。 |
| core 动态工具目录          | 任务粒度工具目录、模型可见工具和调用移交由 FEAT-2026-010 定义。  |
| agent-client 本地能力      | 本地能力声明、执行和结果提交 facade 由 FEAT-2026-007 定义。      |
| Gateway / Event Bus 所有权 | Gateway / Event Bus 不拥有 pending tool call 或 Task 权威状态。  |

## 6. 对下游设计与实现的约束

- L2 设计必须把本特性作为 runtime 响应端侧工具请求和恢复 Task 的事实来源，保持 A2A Task、SSE 和 Task 查询语义一致。
- FEAT-009 与 FEAT-010 的边界必须保持清晰：core 产生端侧工具调用移交意图，runtime 把该意图转化为 A2A 可观察等待状态并处理客户端结果。
- Gateway / IngressGateway / Event Bus 只能交付端侧工具请求和客户端结果，不得拥有 Task 状态或解释工具业务结果。
- 测试必须覆盖 Client 发起调用、runtime 返回端侧工具等待、Gateway 透传、client 提交结果、runtime 恢复 Task、用户拒绝、工具不可用和连接中断后的查询恢复。
- 开发指南不得把端侧工具响应描述为 runtime 直连客户端资源。
- 任何新增客户端直连能力，都必须先更新本特性或新增 version-scope 特性。
- 本特性术语必须保持稳定：端侧工具请求、A2A Task、INPUT_REQUIRED、客户端结果、Task 恢复、Gateway 投影、runtime Task owner。

## 7. 关联文档

- `agent-sdk/Docs/Agent-runtime组件调用端侧工具响应特性设计.md`
- `Docs/FEAT_Design/FEAT-2026-007-agent-client-local-tool-registration-remote-driven-invocation.md`
- `Docs/FEAT_Design/FEAT-2026-010-agent-core-dynamic-client-side-tool-registration-invocation.md`
- `Docs/FEAT_Design/FEAT-001-standardized-agent-service-entrypoint.md`
