## Why

deeptutor-go 需要一个与 Python 基线（v1.5.2）行为对等的运行时注册与路由层，作为整个 Go 后端的内核骨架：ToolRegistry（Level 1 工具的注册/别名/schema/分发）、CapabilityRegistry（Level 2 capability 清单）与 ChatOrchestrator（CLI / WebSocket / SDK 三入口的统一路由）。行为规格见 `docs/golang-req/openspec/specs/runtime-registry/spec.md`；按 `docs/golang-req/openspec/ROADMAP.md`，本模块位于 **Wave 1（运行时内核）/ 里程碑 M1**，是 agent-loop、turn-runtime、tools-builtin 等全部上层模块的直接上游，必须最先落地。

## What Changes

- 新增 `ToolRegistry`（进程级单例）：按 `ToolDefinition.name` 注册/注销工具；内置工具惰性加载（单个失败仅告警跳过）；静态别名表 `TOOL_ALIASES` 解析与默认参数合并；OpenAI function-calling schema 生成（含 `raw_parameters` 优先、array 缺省 `items` 回退）；`get_enabled` 有序去重；`ToolPromptHints` 五种 prompt 格式组装（en/zh）；`execute` 统一分发并透传 `ToolResult` 契约（`content`/`sources`/`metadata`/`success`/`terminate_turn`/`pause_for_user`）。
- 新增 `CapabilityRegistry`：内置 7 个 capability（`chat`/`deep_solve`/`deep_question`/`deep_research`/`math_animator`/`visualize`/`mastery_path`）清单加载（失败跳过、内置优先），`CapabilityManifest` 查询与 `description_i18n` 本地化。
- 新增 `ChatOrchestrator.Handle(context)`：session_id 补齐、capability 选择（默认 `chat`）、未知 capability 转 `error` 事件；每轮独立 StreamBus 生命周期（turn_id 注册全局 bus 表供 `user_input` 路由）、capability 事件按序转发、异常转 `error` 事件、末尾必发 `DONE`；轮结束向全局 EventBus 发布 `CAPABILITY_COMPLETE`（失败静默容忍）。

## Capabilities

### New Capabilities

- `runtime-registry`: Go 版运行时内核的注册与路由层——ToolRegistry / CapabilityRegistry / ChatOrchestrator 的全部行为契约（以基线 v1.5.2 为对等目标）。

### Modified Capabilities

（无——本 change 不修改既有 spec 的需求。）

## Impact

- **依赖的其他 change（按 ROADMAP 依赖图）**：`impl-foundation-config`（settings JSON 读取，registry 加载与 hint 本地化依赖运行时配置）。此外在接口类型层面消费 `impl-foundation-stream` 定义的 `StreamEvent`/`StreamBus`（orchestrator 事件转发面）。
- **下游解锁**：`impl-agent-loop`（经 registry 取工具 schema 与分发）、`impl-turn-runtime`（经 orchestrator 驱动单轮）、`impl-tools-builtin`（工具注册面）。
- **新增代码**：`internal/runtime/registry/`（tool/capability registry）、`internal/runtime/orchestrator.go`、`internal/core/`（`Tool`/`Capability`/`ToolResult` 协议类型）。
- **基线映射**：`deeptutor/runtime/orchestrator.py`、`deeptutor/runtime/registry/`、`deeptutor/runtime/bootstrap/builtin_capabilities.py`、`deeptutor/core/tool_protocol.py`、`deeptutor/core/capability_protocol.py`、`deeptutor/tools/builtin/__init__.py`（`TOOL_ALIASES` 等常量）。
- **协议/数据影响**：无 REST/WS 面直接暴露；无 schema 变更。
