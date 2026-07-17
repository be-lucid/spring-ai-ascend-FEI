---
version: 0715
module: agent-core
feature_type: functional
feature_id: FEAT-2026-010
status: active
---
# Agent-core 组件端侧工具动态注册与调用特性文档

## 1. 特性定位

FEAT-2026-010 定义 `agent-core` 当前版本支持任务粒度动态添加端侧工具，并在 Agent 选择该工具时触发调用移交给 runtime 代理执行的事实。core 的职责是让 Agent 在当前任务上下文中“看见可用端侧工具”，并在模型或流程节点选择工具后输出调用移交意图；runtime 的职责是把该意图转化为 A2A Task 可观察的端侧工具请求，并完成 Client-Gateway-runtime 的结果闭环。

本特性解决的问题是：不同任务、不同页面、不同流程节点下，Agent 可使用的客户端侧工具并不固定。例如当前页面摘要、终端插件动作、业务确认按钮或流程节点临时能力，只能在当前任务范围内生效，不能成为平台全局工具。core 必须基于 runtime / host 提供的客户端能力视图和任务上下文，动态形成当前任务可见工具集合；当 Agent 触发端侧工具调用时，core 不直接执行客户端工具，而是把调用请求移交给 runtime。

在总体架构中，本特性位于 `agent-core` 与 `agent-runtime` 的协作边界。core 负责 Agent 执行过程中的任务粒度工具可见性、工具选择和调用移交；runtime 负责 Task owner、A2A 中断响应、客户端结果接收和 Task 恢复，具体由 FEAT-2026-009 承接；agent-client 负责本地能力声明和真实执行，具体由 FEAT-2026-007 承接。

本特性面向以下角色：

- Agent 应用开发者：声明哪些任务、页面或 workflow 节点可以开放端侧工具。
- agent-core 框架开发者：让 Agent 在任务上下文中获得动态工具可见性，并在工具被调用时输出移交意图。
- Runtime 集成开发者：接收 core 的工具调用移交意图，并按 FEAT-2026-009 投影给 client。
- agent-client 集成方：提供能力视图并执行 runtime 投影出的端侧工具请求。
- 测试与验收团队：验证任务粒度动态工具添加、调用移交和 runtime 恢复闭环。

本特性不定义客户端工具真实执行，不定义 runtime A2A Task / SSE 状态机，也不定义平台全局 Tool Registry、MCP 或 Skill Hub 服务。core 不能绕过 runtime 直接调用客户端，也不能把端侧工具注册为全局服务端工具。

## 2. 当前版本能力要求

| 能力                 | 要求级别 | 事实要求                                                                                           |
| -------------------- | -------- | -------------------------------------------------------------------------------------------------- |
| 任务粒度工具添加     | MUST     | core 必须支持在当前任务范围内动态加入端侧工具，使 Agent 可以在该任务中选择这些工具。               |
| 能力视图接收         | MUST     | core 必须能从 runtime / host 上下文获得客户端当前可用能力视图或等价任务上下文。                    |
| 任务范围约束         | MUST     | 动态端侧工具只在当前任务节点范围内生效，不得自动升级为全局工具。                                   |
| 工具可见性控制       | MUST     | core 必须根据任务上下文、业务配置和运行策略决定哪些端侧工具进入 Agent 可见面。                     |
| 调用移交 runtime     | MUST     | 当 Agent 选择端侧工具时，core 必须输出工具调用移交意图，由 runtime 代理进入 A2A 端侧工具响应闭环。 |
| 不直接执行客户端工具 | MUST     | core 不得访问客户端资源，不得调用 agent-client 内部 API，不得直接执行页面、插件或人工确认动作。    |
| 结果回灌             | MUST     | runtime 恢复 Task 后，core 必须能接收客户端工具结果对应的 observation，并继续当前 Agent 执行。     |
| 全局工具注册         | OUT      | 当前版本不把动态端侧工具注册为平台全局工具、Skill Hub、MCP 或 middleware 工具。                    |

## 3. 外部接口与入口要求

| 入口                         | 类型                   | 事实要求                                                                                               |
| ---------------------------- | ---------------------- | ------------------------------------------------------------------------------------------------------ |
| 客户端能力视图               | runtime / host -> core | 提供当前任务可用的端侧工具名称、用途、必要输入约束和可见范围。                                         |
| Agent 工具可见面             | core -> Agent loop     | core 将当前任务允许使用的端侧工具呈现给 Agent，使模型或流程节点可选择调用。                            |
| 工具调用移交意图             | core -> runtime / host | core 在 Agent 选择端侧工具后输出工具名称、参数和必要上下文，由 runtime 代理执行后续 A2A 端侧工具请求。 |
| 工具结果 observation（回灌） | runtime / host -> core | runtime 恢复 Task 后，将客户端工具结果、拒绝或错误以 observation 形式交回 core，继续 Agent 执行。      |

## 4. 场景与用户旅程

| 场景                     | 前置条件                                                         | 用户/系统动作                           | 期望行为                                                                                                 |
| ------------------------ | ---------------------------------------------------------------- | --------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| 任务开始时添加页面工具   | client 在调用时上报当前页面可用能力，runtime 将能力视图交给 core | core 构建当前任务可见工具集合           | Agent 只能看到当前任务允许的端侧工具；<br />工具不会自动成为其他任务或全局工具。                         |
| Agent 选择端侧工具       | Agent 推理或 workflow 节点决定使用某个端侧工具                   | core 生成工具调用移交意图并交给 runtime | core 不执行客户端工具；<br />runtime 按 FEAT-2026-009 把请求投影为 A2A 等待状态，client 执行后提交结果。 |
| runtime 恢复后继续 Agent | client 已执行本地工具并把结果提交给 runtime                      | runtime 校验后把 observation 回灌 core  | core 接收 observation，继续当前 Agent loop；<br />Agent 可以完成任务、继续推理或再次请求工具。           |
| 多轮任务能力变化         | client 在下一轮调用或任务续接时提供新的能力视图                  | runtime / host 更新 core 的能力视图     | core 按新的能力视图刷新当前任务可见工具集合；旧能力不会自动保留。                                        |

### 4.1 runtime 与 core 配合闭环

1. Client 发起智能体调用，并通过 agent-client / Gateway 提供当前可用端侧能力视图。
2. runtime 接收调用并启动 Agent 执行，将能力视图和任务上下文交给 core。
3. core 基于任务上下文形成当前任务可见工具集合，供 Agent 推理或 workflow 节点选择。
4. Agent 选择端侧工具时，core 输出调用移交意图给 runtime，不直接执行客户端工具。
5. runtime 把移交意图转换为 A2A Task 可观察等待状态，返回给 Gateway / client。
6. client 执行本地工具并提交结果；runtime 校验并恢复 Task。
7. runtime 将结果 observation 交回 core，core 继续 Agent 执行。

## 5. 行为语义与边界

### 5.1 核心行为语义

#### 5.1.0 动态工具可见性语义

- 动态端侧工具只属于当前任务、回合或 workflow 节点上下文，不是平台全局工具。
- core 只能把当前上下文允许的端侧工具呈现给 Agent。
- core 必须按当前上下文计算 Agent 可见工具集合。

#### 5.1.1 调用移交语义

- core 在 Agent 选择端侧工具时只产生调用移交意图。
- runtime 才负责把调用移交意图变成 A2A Task 可观察等待状态。
- client 才负责本地工具真实执行和用户交互。

#### 5.1.2 结果回灌语义

- core 接收的是 runtime 校验后的 observation。
- 用户拒绝、权限不足和执行错误都可以作为 observation 进入 Agent loop。
- core 内部执行继续不替代 runtime Task 状态，也不决定客户端可见 Task 终态。

#### 5.1.3 错误、状态与可观测结果

| 场景              | 事实要求                                                      |
| ----------------- | ------------------------------------------------------------- |
| 未授权工具        | 不进入 Agent 可见工具集合。                                   |
| 移交 runtime 失败 | Agent loop 获得明确错误或失败 observation。                   |
| 客户端拒绝        | runtime 回灌拒绝 observation 后，core 继续按 Agent 逻辑处理。 |
| 结果不符合预期    | core 不应把异常结果当作成功 observation 继续。                |

### 5.2 显式边界与不承诺项

| 边界           | 当前版本不承诺                                                            |
| -------------- | ------------------------------------------------------------------------- |
| 客户端真实执行 | core 不执行页面读取、插件动作或人工确认。                                 |
| FEAT-009 职责  | A2A 等待状态、客户端结果校验和 Task 恢复由 runtime 端侧工具响应特性承接。 |
| 全局工具注册   | 动态端侧工具不自动成为 Skill Hub、MCP 或 middleware 工具。                |
| 绕过 runtime   | core 不绕过 runtime 直连 client，也不调用 agent-client 内部 API。         |
| 跨租户共享     | 动态工具可见性不跨租户、用户或任务共享。                                  |
| 业务事实写入   | tool observation 不自动写入业务系统事实。                                 |

## 6. 对下游设计与实现的约束

- L2 设计必须把本特性作为 agent-core 任务粒度端侧工具可见性和调用移交的事实来源，不得把 core 描述为客户端工具执行器或 runtime Task owner。
- FEAT-010 与 FEAT-009 的边界必须保持清晰：core 负责动态工具可见性和调用移交，runtime 负责 A2A 中断响应、客户端结果校验和 Task 恢复。
- agent-core、runtime 端侧工具响应和 agent-client 本地能力设计必须共享“能力视图、端侧工具请求、调用移交、observation、任务范围”等核心语义。
- 测试必须覆盖任务开始时工具添加、Agent 选择工具、调用移交 runtime、runtime 回灌结果、用户拒绝和多轮能力变化。
- 开发指南必须强调 core 不访问客户端资源、不拥有 A2A Task、不定义 Gateway / Event Bus 投影协议。
- 任何对全局工具市场、客户端直连执行或绕过策略工具注入的新增承诺，都必须先更新本特性或新增 version-scope 特性。
- 本特性术语必须保持稳定：任务粒度工具、能力视图、Agent 工具可见面、调用移交、runtime 代理执行、observation、workflow 节点范围。

## 7. 关联文档

- `agent-sdk/Docs/Agent-core组件端侧工具动态注册与调用特性设计.md`
- `Docs/FEAT_Design/FEAT-2026-007-agent-client-local-tool-registration-remote-driven-invocation.md`
- `Docs/FEAT_Design/FEAT-2026-009-agent-runtime-client-side-tool-response.md`
