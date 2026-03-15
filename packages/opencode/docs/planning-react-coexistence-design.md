# OpenCode Planning 模式实现分析与 ReAct 共存/切换设计文档

## 1. 背景与目标

本文档回答两个问题：

1. OpenCode 的 planning 模式（plan agent）是如何实现的。
2. ReAct（Reason + Act，推理 + 工具调用循环）与 planning 如何在同一会话中共存，并在 plan/build 间切换。

结论先行：

- **Planning 不是单独的执行引擎**，而是通过“agent 配置 + 权限策略 + 系统提醒 + plan_exit 工具”叠加在同一消息循环上。
- **ReAct 是底层统一运行时**（LLM 流式输出 reasoning、tool-call、text、step），plan/build 都跑在这套循环里。
- **切换本质是 agent 切换**（`plan` ↔ `build`），不是引擎切换；模型可保持不变。

---

## 2. 关键组件总览

### 2.1 Agent 层：build / plan 的定义

OpenCode 内置了 `build` 和 `plan` 两个 primary agent：

- `build`：默认可执行代理，允许 `question` 与 `plan_enter`。
- `plan`：规划代理，权限层面禁止常规编辑，仅允许计划文件路径写入，并允许 `question` 与 `plan_exit`。

这两个 agent 都在 `Agent` 注册表中声明，差异主要体现在 `permission` 合并规则上。`plan` 对 `edit` 做了白名单，仅放行 `.opencode/plans/*.md`（以及 data plans 路径）。

### 2.2 Session 层：统一循环（loop）

`SessionPrompt.loop()` 是统一编排器，负责：

- 拉取消息上下文
- 选择当前 user message 指定的 agent/model
- 组装 system prompt、tools
- 调用 `SessionProcessor.process()` 消费 LLM 流
- 根据 finish reason 决定继续、停止或触发 compact

**重点**：plan/build 都使用这一个 loop。

### 2.3 Processor 层：ReAct 流执行

`SessionProcessor` 处理流事件：

- `reasoning-start/delta/end` → 写入 reasoning part
- `tool-call/tool-result/tool-error` → 执行工具并落库
- `text-start/delta/end` → 汇总最终文本
- `start-step/finish-step` → 记录 step 生命周期、tokens/cost、快照 patch

这就是 OpenCode 的 ReAct 执行内核。

### 2.4 Tool 层：plan_exit 驱动“从计划到实施”

实验模式下会注册 `plan_exit` 工具（CLI client）。plan agent 完成计划后调用该工具，工具通过 `Question.ask` 询问用户是否切到 build。若用户确认，会创建一条 synthetic user message，agent 设为 `build`，内容是“计划已批准，开始执行”。

---

## 3. Planning 模式实现细节

### 3.1 开关与注册条件

planning 模式受 `OPENCODE_EXPERIMENTAL_PLAN_MODE` 控制（实验开关）。

- flag 开启后：启用新的 plan reminder 逻辑，并在 tool registry 中注入 `plan_exit`（CLI 场景）。
- flag 关闭时：走旧逻辑（仅注入简化 `PROMPT_PLAN/BUILD_SWITCH` 提醒）。

### 3.2 Plan 文件路径策略

`Session.plan(session)` 根据仓库环境决定计划文件位置：

- Git/VCS 项目：`<worktree>/.opencode/plans/<timestamp>-<slug>.md`
- 非 VCS：`<global data>/plans/<timestamp>-<slug>.md`

这为 plan agent 的“唯一可写文件”约束提供落点。

### 3.3 进入 plan 时的强约束提醒

在 `insertReminders()` 中，当检测到当前 agent=plan 且上一条 assistant 不是 plan 时，会注入 system-reminder（synthetic text），约束包括：

- 禁止任何非只读行为
- 只能写/改 plan file
- 明确 5 阶段规划流程（探索、设计、复核、落盘、`plan_exit`）

这与权限策略形成“双保险”：

- Prompt 约束负责模型行为引导。
- Permission 约束负责工具层硬拦截。

### 3.4 从 plan 返回 build

两条路径：

1. **工具路径（主路径）**：plan agent 调用 `plan_exit`，用户确认后自动插入 agent=build 的 user message。
2. **上下文提醒路径（辅路径）**：当 loop 检测到“上一条 assistant 是 plan、当前 agent 非 plan”，会给 user message 注入 BUILD_SWITCH + plan 文件存在提示，推动执行阶段读取计划。

### 3.5 TUI 本地状态切换

TUI 监听 tool 完成事件：

- `plan_exit` 完成 → `local.agent.set("build")`
- （历史）`plan_enter` 完成 → `local.agent.set("plan")`

注：当前 registry 实际只注入 `plan_exit`，`plan_enter` 在 `plan.ts` 中是注释态历史代码。

---

## 4. ReAct 与 Planning 的共存模型

### 4.1 分层关系

可抽象为三层：

1. **执行层（ReAct Runtime）**：`SessionProcessor + LLM.stream`，统一处理 reasoning/tool/text/step。
2. **策略层（Agent Policy）**：`Agent.permission + resolveTools + PermissionNext`，控制“能做什么”。
3. **流程层（Planning Workflow）**：`insertReminders + plan file + plan_exit`，控制“应该按什么阶段做”。

因此 planning 与 ReAct 不冲突：

- ReAct 负责“怎么跑”。
- Planning 负责“跑什么、先后顺序和边界”。

### 4.2 为什么能共存

- 同一个会话 loop 下，agent 是消息级选择项（`lastUser.agent`）。
- 同一模型可在 plan/build 重用；模型能力（如 reasoning）与 agent 权限正交。
- 工具系统统一，但每次根据当前 agent 权限过滤与执行 ask/deny。

### 4.3 切换机制（状态机）

建议用如下简化状态机理解：

- `BuildActive`
  - 用户要求先做方案 / agent 设为 plan → `PlanActive`
- `PlanActive`
  - 输出问题给用户（`question`）→ `PlanActive`
  - 写完 plan 并 `plan_exit` 且用户同意 → `BuildActive`
  - 用户拒绝切换 → `PlanActive`

其中 ReAct 循环在每个状态内都持续存在。

---

## 5. “react 模型”和 planning 的切换策略建议

如果你说的“react 模型”是“具备强工具调用/推理能力的模型”，建议按下面策略：

1. **模型与 agent 解耦**
   - 模型选择放在 session/user message 的 `model`/`variant`。
   - plan/build 仅切 agent，不强绑模型切换。

2. **按阶段调 effort（可选）**
   - Plan 阶段可用更高 reasoning effort（如 `high`）提高方案质量。
   - Build 阶段降到 `medium/low` 控制成本与延迟。

3. **保持单循环，不做双引擎**
   - 继续沿用当前统一 loop/processor 架构，避免“plan 引擎 + react 引擎”分裂导致状态同步复杂化。

4. **把切换点显式化**
   - 以 `plan_exit` 作为唯一“计划完成”信号。
   - 可新增统一事件 `session.mode.changed(plan|build)`，让 TUI/ACP/插件无歧义消费。

---

## 6. 现状评估与改进建议

### 6.1 优点

- 复用统一 ReAct 运行时，架构简单。
- Prompt 约束 + Permission 硬约束，安全性较好。
- `plan_exit` 形成明确的“人类确认门”。

### 6.2 当前可见问题

- `plan_enter` 在代码中是注释态，但 TUI 和权限仍保留相关痕迹，存在认知噪音。
- plan mode 逻辑存在“实验旧/新分支”，可维护性一般。

### 6.3 建议落地项

1. 清理 `plan_enter` 历史残留（或恢复并正式接入 registry）。
2. 将 `insertReminders` 中大段 plan workflow 文案外置到独立模板，减少核心代码噪声。
3. 在 ACP 侧补充显式 mode 变更事件，统一前端/IDE 联动。
4. 为 plan/build 切换补充端到端测试：
   - 计划文件创建
   - 非计划文件编辑拒绝
   - `plan_exit` 同意/拒绝分支

---

## 7. 一句话总结

OpenCode 的 planning 是**同一 ReAct 引擎上的“受限代理工作流”**：通过 agent 权限和系统提醒把 ReAct 约束到“只读 + 计划落盘 + 用户确认切换”，而不是另起一套执行系统。
