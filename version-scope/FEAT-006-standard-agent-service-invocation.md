---
version: 0715
module: agent-client
feature_type: functional
feature_id: FEAT-2026-006
status: active
---
# Agent-client 组件标准化智能体服务调用特性文档

## 1. 特性定位

FEAT-2026-006 定义 `agent-client` 当前版本作为业务应用侧标准 Agent 服务调用 SDK 的入口事实：业务应用必须通过统一 client facade 提交智能体调用、观察执行过程、查询任务、取消任务和提交补充输入，而不是直接感知后端 `agent-runtime`、`agent-service`、Gateway、Event Bus 或 A2A JSON-RPC 细节。

本特性解决的问题是：不同业务应用需要用同一套客户端调用模型接入智能体平台。应用侧只应看到统一 client facade 下的调用、观察、查询、取消和补充输入能力；平台侧可以通过 Gateway / IngressGateway、agent-bus 或 runtime 标准 A2A 入口完成治理、路由、事件转发和执行。客户端只维护调用过程所需的最小本地上下文和只读状态展示，不拥有服务端 Task 权威状态。

对总体设计而言，本特性是 C/S 流量进入智能体框架的客户端 SDK 入口约束。`agent-client` 位于 application 与 Gateway / IngressGateway / agent-bus 入口之间，负责把业务输入、凭证上下文、幂等键、trace 和观察意图标准化；服务端 Task、SSE、取消控制、补充输入和终态由 `agent-runtime` 拥有，并可由 Gateway / Event Bus 投影给客户端。客户端 facade 可以是 REST 风格或 SDK 方法，但不能形成独立于 A2A Task 的第二套状态机。

本特性面向以下角色：

- 业务应用开发者：通过 `agent-client` 调用智能体服务并展示任务状态。
- 企业终端集成方：把业务页面和凭证上下文接入标准调用链路。
- agent-client SDK 开发者：实现调用 facade、SSE 消费、轮询、取消、补充输入提交和只读状态展示。
- Gateway / IngressGateway 开发者：把 client facade 请求映射到标准平台入口。
- agent-runtime 开发者：承接标准 A2A Task、SSE、取消和订阅语义。
- 测试与验收团队：按客户端外部行为设计黑盒和集成验证。

本特性只定义 `agent-client` 面向业务应用的标准调用入口和只读状态展示。服务端 Agent Card、A2A JSON-RPC、Task 权威状态、agent-bus forwarding、端侧工具结果恢复和 Gateway 路由转发由对应 runtime、gateway、bus 和 client capability 特性承接。

## 2. 当前版本能力要求

| 能力                           | 要求级别 | 事实要求                                                                                                                                                                      |
| ------------------------------ | -------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 标准调用 facade                | MUST     | `agent-client` 必须向业务应用提供统一调用入口，至少覆盖提交调用、流式观察、状态查询、取消和补充输入提交。                                                                   |
| 业务上下文与凭证传递           | MUST     | SDK 必须允许请求携带 agentId、correlation、trace、幂等键和凭证上下文，并传递到 Gateway / IngressGateway 或受治理平台入口。                                                   |
| 平台入口对接                   | MUST     | SDK 请求必须进入受治理平台入口；业务应用不得直接配置 runtime 内部 API、物理 endpoint、broker 或 Event Bus topic。                                                             |
| 流式观测                       | MUST     | SDK 必须支持消费 Gateway / runtime A2A SSE 或等价服务流，并把服务端事件映射为客户端可见的只读状态投影。                                                                       |
| 阻塞调用                       | MUST     | SDK 必须支持适合一次性响应的调用模式；阻塞结果必须来自服务端标准 Task / Message 表面或 Gateway 投影。                                                                         |
| 异步状态观察                   | MUST     | SDK 必须支持通过 Gateway 投影、状态查询或受治理映射观察异步任务进展；不得只返回本地缓存状态。                                                                                 |
| 取消任务                       | MUST     | SDK 必须提供 cancel 能力，并通过 Gateway / IngressGateway 映射到 runtime 取消控制路径；取消结果以 runtime Task 状态为准。                                                     |
| 补充输入提交                   | MUST     | SDK 必须支持在 runtime 请求补充输入后，由业务应用主动提交用户补充输入或本地确认；恢复点校验和 Task 状态推进由 runtime 控制。                                                   |
| 幂等处理                       | MUST     | SDK 必须支持提交或复用幂等键；重试不得造成重复 Task。                                                                                                                         |
| 错误分类（错误映射非错误管理） | MUST     | SDK 必须区分网络错误、路由错误、服务端 A2A / Task 错误、业务失败、取消和拒绝。                                                                                                |
| 本地状态边界                   | MUST     | SDK 只能保存调用过程所需的最小本地上下文、幂等键和只读状态投影；不得实现服务端 TaskStore 或决定 Task 终态。                                                                   |

## 3. 外部接口与入口要求

| 入口                                      | 类型           | 事实要求                                                                                                                                                        |
| ----------------------------------------- | -------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| unified client facade                     | SDK facade     | 以统一调用模型封装 A2A 兼容的提交、流式观察、查询、取消和补充输入语义；SDK 可以提供方法级便捷 API，但不得形成与 A2A Task 脱钩的第二套入口协议。                   |
| call mode / observe mode                  | SDK 调用语义   | 同一次调用可选择阻塞返回、流式观察；<br />流式观察消费 Gateway / runtime A2A SSE 或投影事件。                                                                   |
| `agentClient.cancel`                    | SDK facade     | 基于服务端任务句柄发起取消请求，由平台侧转发到 runtime 取消控制路径。                                                                        |
| 服务端任务句柄                            | 服务端任务 id  | runtime 接受调用后返回，是查询、取消、订阅和续接服务端 Task 的权威句柄。                                                                                        |
| runtime A2A 语义                          | 平台标准依赖   | Gateway、IngressGateway 或目标服务最终应对齐 `/a2a`、Task、SSE、查询、取消、订阅和补充输入等标准语义。                                          |

## 4. 场景与用户旅程

| 场景                 | 前置条件                                                        | 用户/系统动作                                                                                                                                                                                     | 期望行为                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| -------------------- | --------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 业务应用发起标准调用 | 应用已集成 SDK，平台入口可用                                    | 应用通过统一 client facade 发起调用并携带业务输入、凭证上下文和幂等键                                                                                                                             | SDK 标准化请求并提交 Gateway；调用被接受后返回服务端任务句柄；服务端 Task 由 runtime 持有。                                                                                                                                                                                                                                                                                                                                                                                 |
| 流式观察长任务       | 调用方需要实时状态，目标 runtime 支持 A2A SSE 或 Gateway 投影流 | 应用在同一 client facade 下选择流式观察或订阅已有服务端任务                                                                                                                                       | SDK 消费 SSE / 服务流，将执行中、等待输入、完成、失败、取消等服务端事实映射为业务可见只读投影。<br />Agent 流式输出经 Gateway 透传到 Client，链路通过长连接保持。                                                                                                                                                                                                                                                                                                      |
| 阻塞获取结果         | 请求规模适合一次性响应                                          | 应用调用阻塞 facade                                                                                                                                                                               | SDK 返回服务端 Task / Message 表面或 Gateway 投影；<br />超时或平台未返回明确结果时给出可继续查询的状态，不伪造成功或失败。<br />说明：<br />（1）Gateway 与 runtime 之间出现链路异常时，由 Gateway 统一向 Client 返回错误控制状态消息。<br />（2）其他业务状态由 runtime 控制，并经 Gateway 透传给 Client。<br />（3）Client 自身链路异常由 Client 按本地异常处理策略处理。                                                                                                                |
| 用户补充输入         | runtime 请求用户补充信息或等待确认                              | 用户在业务界面对话或等待执行任务时，需要补充信息或确认动作                                                                                                                                        | Client 通过 A2A 标准协议与 Gateway 交互，再由 Gateway 将消息转发到 runtime。SDK 将终端用户补充输入或确认信息连同服务端任务上下文提交给 Gateway，由 runtime 校验并推进下一轮业务处理。                                                                                                                                                                                                                                                                                  |
| 客户端取消任务       | 用户希望停止执行中任务，SDK 已获得服务端任务句柄                 | 应用调用 cancel facade                                                                                                                                                                           | SDK 通过 Gateway 转发取消请求；runtime 按自身 Task 生命周期决定取消结果并返回 canceled、not_cancelable 或当前 Task 状态。当前客户端取消任务的业务场景可以形成 runtime 中止任务并向 Gateway、Client 返回终止消息的业务闭环；针对 Agent 实际业务终止，依赖 Agent core AI 框架能力，本次需求不要求验证规划。                                                                                                                           |

## 5. 行为语义与边界

### 5.1 核心行为语义

#### 5.1.0 客户端入口等价语义

- 不同业务应用通过 `agent-client` 调用 Agent 时，应看到同一组调用、观察、查询、取消和补充输入语义。
- SDK 可以根据部署选择 HTTP、SSE、Gateway facade 或总线异步入口，但客户端状态必须最终对齐服务端 Task。
- SDK 不得因为底层经过 Gateway、Event Bus、agent-service 或 runtime 而暴露互相漂移的业务状态机。

#### 5.1.1 调用提交语义

- 提交请求必须携带可关联的业务上下文、幂等键和输入内容。
- SDK 必须复用幂等键处理提交重试；重复提交不得造成重复 Task。
- 请求未确认是否被 runtime 接受时，SDK 必须返回可继续查询或重试的状态，而不是伪造成功或失败。

#### 5.1.2 流式观测语义

- SDK 消费到的流式事件必须映射为客户端可理解状态，但状态来源仍是 Gateway / runtime 投影。
- SSE 中断不等于 Task 失败；SDK 应提示调用方通过平台状态查询确认任务进展。
- 流式 token、progress 和终态必须与 runtime Task/SSE 语义保持一致。

#### 5.1.3 查询与取消语义

- 查询能力必须基于服务端任务句柄读取服务端 Task 或受治理投影，不得只读取本地调用对象。
- `cancel` 必须映射到 runtime 取消控制路径；底层执行是否立即中断由 runtime/adapter 能力决定。
- Task 进入 completed、failed、canceled 等终态后，SDK 不得继续提交普通续接输入。

#### 5.1.4 续接输入语义

- 用户补充输入、本地确认和客户端能力结果都必须由客户端主动提交。
- 补充输入请求必须携带服务端任务句柄、callbackId、correlationId 或等价上下文，便于 runtime 校验恢复点。
- 过期、重复、跨租户、跨用户或终态后的续接必须返回明确错误。

#### 5.1.5 错误、状态与可观测结果

| 场景         | 事实要求                                                                         |
| ------------ | -------------------------------------------------------------------------------- |
| 网络失败     | SDK 返回可重试网络错误，并保留幂等键用于恢复判断。                               |
| 路由失败     | SDK 暴露 route not found、permission denied、service unavailable 等平台错误。    |
| 服务端失败   | SDK 展示 failed Task 或 JSON-RPC error，不把业务失败包装成网络失败。             |
| 接受未确认   | SDK 返回可继续查询或重试的状态，并保留幂等键用于恢复判断。                       |
| SSE 中断     | SDK 提示调用方通过平台状态查询确认任务进展。                                     |
| 用户取消     | SDK 转发取消请求，并以服务端 Task 状态为准。                                     |

### 5.2 显式边界与不承诺项

| 边界               | 当前版本不承诺                                                         |
| ------------------ | ---------------------------------------------------------------------- |
| 服务端 TaskStore   | `agent-client` 不保存或复制服务端 Task 权威状态。                    |
| 直接内部调用       | SDK 不直接调用 agent-runtime、agent-core、agent-middleware 内部 SPI。  |
| 租户认证           | SDK 不认证租户身份；租户认证、清洗和注入由 Gateway 或企业入口完成。    |
| Agent 执行         | SDK 不执行 Agent 推理、工具调用、记忆、知识库或模型访问。              |
| 强制取消底层模型   | SDK cancel 不承诺强制中断已进入模型客户端的阻塞调用。                  |
| 多 Agent 编排      | SDK 只发起和观察调用，不定义服务端多 Agent 路由、聚合和调度策略。      |

## 6. 对下游设计与实现的约束

- L2 设计必须把本特性作为业务应用侧标准 Agent 调用入口事实来源，不得把 `agent-client` 降级为示例代码或直接 runtime API 包装。
- `agent-client` SDK、Gateway facade、IngressGateway、agent-bus 转发和 runtime A2A 入口必须保持同一套 Task、SSE、取消、续接和错误语义。
- SDK 状态模型必须区分服务端 Task 状态、平台投影状态和本地 UI 展示状态。
- 测试必须覆盖标准提交、流式观察、阻塞响应、异步查询、取消、用户补充输入、幂等重试和错误分类。
- 开发指南不得要求业务应用理解 runtime 内部 handler、TaskStore、Agent-core loop 或 agent-middleware 工具实现。
- 任何对客户端 TaskStore、租户认证或服务端编排能力的新增承诺，都必须先回到本特性或新的 version-scope 特性文档更新事实要求。
- 本特性使用的术语必须保持稳定：agent-client、Gateway、IngressGateway、SSE、Task、Cancel。

## 7. 关联文档

- `agent-sdk/Docs/Agent-client组件标准化智能体服务调用特性设计.md`
- `Docs/FEAT_Design/FEAT-001-standardized-agent-service-entrypoint.md`
- `Docs/FEAT_Design/FEAT-2026-011-agent-gateway-client-invocation-route-forwarding.md`
- `Docs/FEAT_Design/FEAT-2026-012-agent-gateway-client-invocation-bus-forwarding.md`
