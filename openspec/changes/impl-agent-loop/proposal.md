## Why

chat capability 的单 Agent Loop 是 deeptutor-go 的核心执行引擎：一次用户轮 = 在同一条不断增长的会话上跑 tool-calling 循环，覆盖 context-gated tool mounting、轮次预算强制收敛、narration/thinking/tool_call 事件时序与 `ask_user` 暂停/恢复。行为规格见 `docs/golang-req/openspec/specs/agent-loop/spec.md`；按 `docs/golang-req/openspec/ROADMAP.md`，本模块位于 **Wave 1（运行时内核）/ 里程碑 M1**（M1 退出条件之一：「Agent Loop——tool_call 循环、轮次预算强制收敛、context-gated mounting 与基线一致」），是 turn-runtime 与全部 capability 的执行底座。

## What Changes

- 新增基于 cloudwego/eino 的 Agent Loop：每轮一次流式 LLM 调用（携带全部启用工具 schema、`tool_choice: "auto"`），有 tool_call 则并行分发（上限 8、同批去重、stub 配对回写）后继续，无 tool_call 即终答（`call_role: narration|finish` 标记，无独立 respond 阶段）。
- 新增轮次预算（`max_rounds` 默认 8、`_min_loop_rounds` 地板）与强制收敛路径（warning progress + 收敛指令 + 关闭工具的收尾调用）；中途失败挽救收敛与协议兜底终答；空终答 nudge。
- 新增会话消息组装（system prompt 字节稳定 → 压缩摘要 → 历史 → 末尾 user 消息，KB seed / capability seed 拼 user 消息）与多模态附件准备。
- 新增 context-gated tool mounting 四步组装（用户 toggle ∩ 白名单 → 条件挂载 → capability owned → 恒挂载）+ `allowed_builtin_tools`/`forced`/`suppressed`/exclusive 裁剪 + deferred 工具与 `load_tools` 渐进披露 + PageIndex 预加载。
- 新增 KB seed 预检索（≤3 KB、每 KB 截断 4000 字符、sources 事件先行）。
- 新增流式事件产生管线：call_status progress、content/thinking 增量、内联 `<think>` 标签跨 chunk 分流、tool_call/tool_result 事件、stage `responding` 包裹、sources 合并、capability result + UsageTracker 成本汇总；retrieve 类工具子 trace 进度流。
- 新增上下文窗口守护（90% 预算 snip 最早 tool 消息）与 `_context_checkpoint` 折叠。
- 新增 `ask_user` loop 侧暂停/恢复（waiter 句柄、回复替换进 `role=tool` 消息、`ask_user_resolved` progress、waiter 空值结束循环）与 provider 兼容性回退（stream_options / 工具 schema / 图片输入 / 无原生 tool calling）。

## Capabilities

### New Capabilities

- `agent-loop`: chat capability 的单 Agent Loop 全部行为契约——循环结构、工具挂载、事件时序、预算收敛、ask_user 暂停/恢复、provider 回退（以基线 v1.5.2 为对等目标）。

### Modified Capabilities

（无——本 change 不修改既有 spec 的需求。）

## Impact

- **依赖的其他 change（按 ROADMAP 依赖图）**：`impl-llm-provider`（eino ChatModel 适配 + mock/replay provider）、`impl-runtime-registry`（工具 schema 与执行分发）；类型层面消费 `impl-foundation-stream`（`StreamEvent`/`StreamBus`）。
- **下游解锁**：`impl-turn-runtime`（经 orchestrator 驱动 loop 并托管 `wait_for_user_reply`）、`impl-tools-builtin`、`impl-capability-chat-solve`。
- **新增代码**：`internal/agents/chat/`（pipeline、loop、消息组装、tool mounting）、`internal/llm/agentic/`（tool 分发、think 标签分流、窗口守护）。
- **新增依赖**：`github.com/cloudwego/eino`、`github.com/cloudwego/eino-ext`（model 组件）。
- **基线映射**：`deeptutor/agents/chat/agentic_pipeline.py`、`deeptutor/agents/chat/agent_loop.py`、`deeptutor/core/agentic/tool_dispatch.py`、`deeptutor/agents/_shared/tool_composition.py`、`deeptutor/tools/builtin/__init__.py`（工具集合常量）。
- **协议/数据影响**：无 REST/WS 面直接暴露；事件形状经 foundation-stream 的 `StreamEvent` 契约约束，最终经 WS fixtures 回放验收。
