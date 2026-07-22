# impl-capability-chat-solve — Tasks

## 1. 骨架与资源

- [ ] 1.1 建立 `internal/capabilities/{shared,chat,solve}` 包骨架，定义 `LoopCapability` 接口（IsActive / SystemPromptBlock / OwnedTools / InjectKwargs / Exclusive）并评审冻结
- [ ] 1.2 从基线原样拷贝 prompts：`agents/chat/prompts/{en,zh}/agentic_chat.yaml`、`capabilities/solve/prompts/{en,zh}/system.md`，配置 `go:embed`
- [ ] 1.3 实现 `shared.PromptStore`：YAML 键路径寻址、`{var}` 命名占位符注入、缺键回退 fallback、语言归一（`zh*`→zh）；对全部键跑渲染快照测试对齐 Python `str.format`

## 2. 收敛点与 chat capability 装配

- [ ] 2.1 实现 `shared.EmitCapabilityResult()`：payload + UsageTracker `cost_summary` 合并（保留已有 metadata 键）
- [ ] 2.2 实现 `ChatCapability.Manifest()`：`stages=["exploring","responding"]`、`cli_aliases=["chat"]`、`tools_used=CHAT_OPTIONAL_TOOLS`
- [ ] 2.3 实现 `chat/pipeline.go` 消息装配：system prompt 字节稳定、压缩历史摘要 header、附件多模态注入、KB seed（≤3 KB / 每 KB 4000 字符截断 / sources 事件）注入尾部 user 消息
- [ ] 2.4 接入工具组合策略（impl-agent-loop 的 mounting 原语）：CHAT_OPTIONAL_TOOLS 白名单、ToolMountFlags 自动挂载、imagegen/videogen 未配置剔除、partner 抑制、exec 门控、rag schema 的 `kb_name` enum 注入
- [ ] 2.5 `Run()` 内以 `bus.Stage("responding", source="chat")` 包裹整个 loop；确认全程无 `exploring` stage 事件；loop 结束后聚合 sources → `EmitCapabilityResult`

## 3. loop 行为接线（消费 impl-agent-loop 原语）

- [ ] 3.1 轮次协议接线：narration/finish `call_role`、`call_status` running/complete progress、轮次预算默认 8 + `_min_loop_rounds` 地板
- [ ] 3.2 强制收尾路径：预算耗尽 warning + finish-exhausted 指令 + 禁工具终调；中途失败（≥1 轮后）复用同路径、第 0 轮失败上抛
- [ ] 3.3 ask_user 暂停恢复：waiter await、回复替换 `role=tool` 消息、`user_reply` progress（`reply_preview` 截 200）、放弃/无 waiter 的 `completed=false` 收尾
- [ ] 3.4 上下文窗口守卫（90% 预算 snip + warning）与 `_context_checkpoint.summary` 折叠
- [ ] 3.5 thinking 分流（reasoning 通道 + 内联 `<think>` 增量剥离、24 字符回扣）、空 finish nudge（每 turn 一次）与 `empty_final_response` fallback、provider 兼容回退三则

## 4. deep_solve

- [ ] 4.1 实现 `SolveSession` + 有界 LRU(256) store（写锁、超容淘汰最旧）
- [ ] 4.2 实现三件套工具：`solve_plan`（服务端生成 S1..、≤12 步、空 steps 失败）、`solve_finish_step`（未知/无计划失败、`_context_checkpoint` metadata、all_done）、`solve_replan`（预算内替换 +1、耗尽返回 `budget_exhausted` 不改计划）、无 `_solve_session_id` 一律失败
- [ ] 4.3 实现 `DeepSolveCapability`：manifest（`stages=["responding"]`、三件套+rag/code_execution/geogebra_analysis/reason、`cli_aliases=["solve"]`）；`Run()` 置 `solve_mode`、白名单清洗生成 `solve_session_id`、读 solve 设置（`max_replans` 默认 2）后委托 chat pipeline
- [ ] 4.4 实现 `SolveLoopCapability`：solve system prompt block（可被 `solve.system` 键覆盖）、chat 全量工具面 + 三件套追加、`InjectKwargs` 注入 `_solve_session_id`/`max_replans`
- [ ] 4.5 确认 solve result envelope 与 chat 完全一致（同 `EmitCapabilityResult`，无 solve 专属形状、无额外 stage 事件）；会话延续（计划留存于 tool 消息）用集成测试覆盖

## 5. 验收（spec Scenario 落测试 + 协议对等）

- [ ] 5.1 spec 全部 15 个 Scenario 落为 Go 单测/集成测试（mock/replay eino ChatModel）：stage 事件、轮次协议、KB seed、工具剔除、ask_user、checkpoint 折叠、think 分流、result envelope、prompts 回退、solve 工具面、replan 耗尽、all_done、solve 追问延续
- [ ] 5.2 stage 事件 fixtures 对等验收（acceptance.md §4 M1/M3 矩阵）：录制基线普通 chat turn / ask_user 暂停恢复 / solve turn 的 `/api/v1/ws` JSONL fixtures，Go 侧 mock LLM 回放逐事件 diff（忽略 timestamp，seq 连续、顺序一致）
- [ ] 5.3 收敛点契约测试：`EmitCapabilityResult` 的 envelope 形状 + `cost_summary` 合并规则固化为共享测试，供后续 capability change 复用
