# impl-capability-chat-solve — Proposal

## Why

deeptutor-go 需要交付 `chat`（默认 capability，M1 先行）与 `deep_solve`（M3）两个核心 capability 的 Go 实现。行为事实源为 `docs/golang-req/openspec/specs/capability-chat-solve/spec.md`；按 `docs/golang-req/openspec/ROADMAP.md` 定位，本模块属 Wave 3（capability-chat-solve），其中 chat 部分是 M1 退出条件「前端 Chat 页全功能可用」的直接载体，solve 部分归 M3。chat 是全部上层 capability（mastery / question / research 等）复用的 agentic loop 宿主，必须最先冻结其 stage 事件与 result envelope 语义，前端才能零改动复用。

## What Changes

- 新增 `internal/capabilities/chat`：chat capability——manifest（`stages=["exploring","responding"]` 名实分离，实际只在 `responding` 单 stage 内跑合并 loop）、消息装配（system prompt 字节稳定、KB seed 注入尾部 user 消息）、单 loop 轮次协议（narration / finish、`call_status` progress、轮次预算 8 + `_min_loop_rounds` 地板、预算耗尽/中途失败强制收尾）、工具组合策略接入、`ask_user` 暂停恢复、上下文窗口守卫与 `_context_checkpoint` 折叠、thinking 分流与空 finish 补救、provider 兼容回退。
- 新增 `internal/capabilities/shared`：`EmitCapabilityResult()` 统一收敛点（对应基线 `deeptutor/agents/_shared/capability_result.py`），所有 capability 的 result envelope（response payload + `metadata.cost_summary`）经此发出。
- 新增 `internal/capabilities/solve`：`deep_solve` capability——复用 chat loop，仅打 `solve_mode` 标记、注入 solve system prompt、追加 `solve_plan` / `solve_finish_step` / `solve_replan` 三件套工具；内存态 `SolveSession`（有界 LRU 256）状态机。
- 从基线原样拷贝 prompts 资源：`agents/chat/prompts/{en,zh}/agentic_chat.yaml`、`capabilities/solve/prompts/{en,zh}/system.md`，并实现 `{var}` 命名占位符注入与语言归一（`zh*` → zh）。

## Capabilities

### New Capabilities
- `capability-chat-solve`: chat 默认 capability（单 agentic loop、流式协议、工具组合、结果收敛点）与 deep_solve capability（solve 模式标记 + 三件套工具 + SolveSession），行为对等 Python 基线 v1.5.2。

### Modified Capabilities
（无——本 change 只新增能力，不修改既有 spec 的 Requirement。）

## Impact

- 依赖的其他 change（按 ROADMAP 依赖图）：
  - `impl-runtime-registry`（CapabilityRegistry 注册、ToolRegistry 分发、Orchestrator 路由与 StreamBus 生命周期）
  - `impl-agent-loop`（单循环轮结构、context-gated tool mounting、事件时机等 loop 原语——本 change 是其宿主与首个消费者）
  - `impl-turn-runtime`（UnifiedContext 组装、`wait_for_user_reply` waiter、事件持久化）
  - `impl-tools-builtin`（挂载的内置工具面：`rag`/`web_search`/`ask_user` 等）
  - `impl-memory`（`read_memory`/`write_memory` 挂载条件与实现）
  - `impl-skills-persona`（`read_skill`/`load_tools` 挂载条件与实现）
  - 间接依赖 Wave 0：`impl-llm-provider`（eino ChatModel 流式）、`impl-foundation-stream`（StreamEvent/StreamBus）、`impl-foundation-config`。
- 受影响面：统一 WS `/api/v1/ws` 的 chat turn 事件序列（M1 fixtures）；所有后续 capability change 复用本 change 的 loop 与收敛点。
- 被解锁的 change：`impl-capability-mastery`、`impl-capability-question`、`impl-capability-research`（均复用 chat loop 原语）、`impl-partners`。
