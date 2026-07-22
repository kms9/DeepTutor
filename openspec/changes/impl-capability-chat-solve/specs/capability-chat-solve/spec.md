## ADDED Requirements

### Requirement: chat capability manifest 与合并 loop 语义
系统 SHALL 注册名为 `chat` 的 capability，manifest 声明 `stages=["exploring", "responding"]`、`cli_aliases=["chat"]`、`tools_used` 为 chat 可选工具白名单（`CHAT_OPTIONAL_TOOLS`）。实际运行 SHALL 是单一合并 loop：整个 turn 只打开一个 stage（`LOOP_STAGE = "responding"`），所有轮次的事件（content / thinking / tool_call / tool_result / progress）均在 `stage="responding"` 下发出，SHALL NOT 单独发出 `exploring` 的 `stage_start`/`stage_end`。manifest 中的 `exploring` 声明仅为前端展示兼容保留。`stage_start` / `stage_end` 事件由 StreamBus 的 stage 上下文自动发出，包裹整个 loop。

#### Scenario: 一次 chat turn 的 stage 事件
- **WHEN** 用户发送一条消息触发 chat capability
- **THEN** 流上首先出现 `stage_start(stage="responding", source="chat")`
- **AND** loop 结束后出现 `stage_end(stage="responding")`，全程没有 `exploring` 的 stage 事件

### Requirement: 单 loop 轮次协议（narration / finish）
系统 SHALL 按以下轮次协议运行 loop：每轮一次 LLM 流式调用；本轮文本一律实时以 `content` 事件流出（metadata 携带 `trace_kind: "llm_chunk"`）；若本轮产生 tool calls，则该轮为 narration（前导叙述），assistant 消息（文本 + tool calls）与 `role=tool` 结果追加进同一会话后继续下一轮；若本轮没有 tool calls，则该轮为 finish，其文本即最终答案，loop 结束。每轮 LLM 调用完成时 SHALL 发出 `progress` 事件，metadata 含 `trace_kind: "call_status"`、`call_state: "complete"` 与 `call_role`（有 tool calls 为 `"narration"`，无为 `"finish"`）；调用开始时发 `call_state: "running"`。首轮即无 tool calls 属正常快速路径。轮次预算默认 8（`chat` 配置 `max_rounds` 可覆盖），并 SHALL 支持 `context.metadata["_min_loop_rounds"]` 抬高本 turn 的下限。

#### Scenario: 模型第二轮直接作答
- **WHEN** 第一轮调用了 `web_search`，第二轮 LLM 输出纯文本且无 tool calls
- **THEN** 第二轮文本已实时流出，其 `call_status` progress 事件带 `call_role: "finish"`，loop 结束
- **AND** 第一轮的 `call_status` 事件带 `call_role: "narration"`

#### Scenario: 轮次预算耗尽强制收尾
- **WHEN** 连续 `max_rounds` 轮均请求 tool calls
- **THEN** 发出 `progress`（`trace_kind: "warning"`，预算耗尽文案），向会话追加 finish-exhausted 指令，并做一次禁用 tools 的最终 LLM 调用，其文本作为最终答案

#### Scenario: 中途 LLM 调用失败的抢救
- **WHEN** loop 已完成至少 1 轮后某轮 LLM 调用抛异常（超时/网络）
- **THEN** 不丢弃已收集的工作：走与预算耗尽相同的强制收尾路径（warning 文案为 error 变体）
- **AND** 若失败发生在第 0 轮（尚无任何成果），异常向上传播

### Requirement: 消息装配与 system prompt 字节稳定
系统 SHALL 每 turn 构建一份会话消息：`[system prompt] + 历史消息 + [尾部 user 消息]`，loop 各轮向其追加。system prompt 在整个 turn 内 SHALL 字节稳定（KB 缓存前缀不被破坏）；KB seed、capability pre-loop seed / briefing 等动态内容 SHALL 注入尾部 user 消息而非 system prompt。历史中 `role in {user, assistant}` 的条目原样进入；`role=system` 的压缩历史摘要 SHALL 加上 prompts 中 `notices.conversation_summary_header` 头部后紧随 system prompt 传递。附件 SHALL 经多模态消息准备注入（按 binding/model 能力）。KB seed 检索 SHALL 限制最多 3 个 KB、每 KB 4000 字符（超出截断加 `...[truncated]`），命中来源以 `sources` 事件发出。

#### Scenario: 挂 KB 时的种子检索
- **WHEN** turn 挂载 2 个 rag KB 且 user_message 非空
- **THEN** 对每个 KB 并行执行一次 `rag`（mode=hybrid）检索，结果拼为 `[Knowledge Base Context]` 块注入尾部 user 消息
- **AND** 聚合来源以 `sources` 事件（`trace_kind: "sources"`）发出

### Requirement: 工具组合策略
系统 SHALL 按共享组合策略确定本 turn 挂载的工具：用户显式请求的可选工具（`CHAT_OPTIONAL_TOOLS` 白名单内）+ 依 `ToolMountFlags` 自动挂载的 context-gated 工具（`has_kb` → `rag`，`has_memory` → memory 工具，`has_notebooks` → notebook 工具，`has_skills` → `read_skill`，`has_deferred_tools` → `load_tools`，`has_exec`/`has_code` → `exec`/`code_execution` 等）+ 活跃 loop capability 的 owned tools。约束：`imagegen`/`videogen` 在对应服务未配置激活模型时 SHALL 从工具列表剔除；partner turn SHALL 强制挂载 partner 内置工具并抑制 `read_memory`/`write_memory`；`exec` 的可用性 SHALL 经沙箱隔离级别 + 用户授权门控（门控失败一律禁用）。native tool calling 不可用（binding/model 不支持）时 SHALL 不传 tools schema，改为 prompt 内清单。`rag` schema 的 `kb_name` SHALL 注入本 turn 挂载 KB 的 enum。

#### Scenario: 生成类工具未配置被剔除
- **WHEN** 用户开启了 `imagegen` 但管理员未配置 imagegen 激活模型
- **THEN** 本 turn 的工具列表与 schema 中均不含 `imagegen`

### Requirement: ask_user 暂停与恢复
当工具分发返回 pause（`ask_user`）时，系统 SHALL 经 `context.metadata["wait_for_user_reply"]` 等待用户回复：回复到达后 SHALL 将格式化后的回复正文 + 继续作答指令替换进对应 `tool_call_id` 的 `role=tool` 消息，发出一条 `progress`（metadata 含 `trace_kind: "user_reply"`、`ask_user_resolved: true`、`reply_preview` 截断 200 字符、结构化 `answers` 如有），然后继续下一轮。回复为 `None`（用户放弃）或 waiter 不可用时 SHALL 停止 loop：waiter 不可用时把问题摘要作为终结响应 content 发出；最终 result 的 `completed` 为 `false`。

#### Scenario: 用户回答 ask_user 的选择题
- **WHEN** 模型调用 `ask_user` 提出带选项的问题，用户在卡片上作答
- **THEN** 对应 tool 消息内容被替换为 "User answered:" 格式化正文加继续指令，loop 继续
- **AND** 最终 result 事件 `completed: true`

### Requirement: 上下文窗口守卫与 checkpoint 折叠
每次 LLM 调用前系统 SHALL 估算消息 token 数，超过有效上下文窗口的 90% 时，按顺序把 `role=tool` 消息内容替换为 snip 标记文案直至回到预算内，并发出一条 `progress` warning。当某轮工具结果 metadata 携带 `_context_checkpoint.summary` 时（如 `solve_finish_step`），系统 SHALL 把 checkpoint 边界之后的全部消息折叠为一条 `role=system` 的 `[Context checkpoint]\n<summary>` 消息，并推进边界——这是 solve 逐步收敛上下文的机制。

#### Scenario: solve_finish_step 触发折叠
- **WHEN** solve turn 中模型调用 `solve_finish_step` 完成 S1
- **THEN** 该步骤积累的中间 tool 消息被折叠为一条 `[Context checkpoint] [S1] <goal> — done. <summary>` system 消息

### Requirement: 流式细节（thinking 通道、内联 think 标签、空 finish 补救）
系统 SHALL 把 delta 中的 `reasoning_content`/`reasoning` 路由到 `thinking` 事件；content 通道中内联的 `<think>`/`<thinking>` 段 SHALL 被增量切分器实时剥离转投 `thinking` 通道（原始文本仍原样回填 LLM 会话），切分器 SHALL 容忍跨 chunk 的部分标签（回扣最长 24 字符）。当 finish 轮清洗后文本为空（模型整轮只输出内部推理）时，系统 SHALL 发一条 warning progress 并向会话注入一次 nudge user 消息要求模型继续（每 turn 至多一次）；若最终答案仍为空，SHALL 发出 prompts 中 `notices.empty_final_response` 的兜底文案作为 fallback 最终响应（metadata 标记 `fallback: true`）。provider 兼容性回退 SHALL 保持：`stream_options` 不支持时去掉重试、tool schema 被拒时去 tools 重试并发 warning、图片输入不支持时剥离图片重试并发 warning。

#### Scenario: 整轮输出都在 think 标签内
- **WHEN** 某轮无 tool calls 且全部文本位于 `<think>...</think>` 内
- **THEN** 用户侧 content 通道为空、thinking 通道收到推理文本，随后发生一次 empty-finish nudge，loop 继续

### Requirement: 统一结果 envelope（收敛点）
loop 结束后系统 SHALL 经共享的 `emit_capability_result()`（对应基线 `deeptutor/agents/_shared/capability_result.py`）发出唯一 result 事件：payload 为 `{response: 最终答案文本, completed: bool, engine: "agent_loop", rounds: 轮次数, tool_steps: 工具分发批次数}`；当 `UsageTracker` 有记录时 SHALL 把 `usage.summary()` 合并进 `payload.metadata.cost_summary`（保留已有 metadata 键）。所有 capability（含 deep_solve、mastery_path 及其它模块的 pipeline）SHALL 经同一函数收敛，保证 envelope 形状一致（response payload + cost_summary）。累计的 sources SHALL 在 result 之前以一条聚合 `sources` 事件发出。

#### Scenario: 带用量统计的最终 result
- **WHEN** 一次 turn 消耗了 3 次 LLM 调用
- **THEN** result 事件 payload 含 `response`/`completed`/`engine`/`rounds`/`tool_steps`，且 `metadata.cost_summary` 含成本/token/调用次数汇总

### Requirement: prompts 资源与模板注入
chat 的全部提示词与状态文案 SHALL 来自 `agents/chat/prompts/{en,zh}/agentic_chat.yaml`（YAML 键路径寻址，如 `notices.context_window_guard`、`labels.exploring`、`loop.finish_empty_nudge`）；solve 的 system prompt SHALL 来自 `capabilities/solve/prompts/{en,zh}/system.md`（整文件读取，允许被 chat prompts 中 `solve.system` 键覆盖）。Go 版 SHALL 原样拷贝这些资源文件，并实现与 Python `str.format(**kwargs)` 等价的 `{var}` 命名占位符注入；键缺失或格式化失败时 SHALL 回退到调用点默认文案而非报错。语言解析规则：`zh` 前缀归一为 `zh`，其余为 `en`。

#### Scenario: 缺失 prompt 键回退默认值
- **WHEN** zh prompts 文件缺少 `notices.start_retrieval` 键
- **THEN** 使用代码内默认英文文案，turn 正常进行

### Requirement: deep_solve capability 装配
系统 SHALL 注册名为 `deep_solve` 的 capability：manifest 声明 `stages=["responding"]`、`tools_used=[solve_plan, solve_finish_step, solve_replan, rag, code_execution, geogebra_analysis, reason]`、`cli_aliases=["solve"]`。`run()` SHALL：置 `context.metadata["solve_mode"]=true`；由 `turn_id`（回退 `session_id`、`message_id`）经字符白名单 `[A-Za-z0-9_-]` 清洗生成 `solve_session_id`；读取 solve 设置（`max_replans` 默认 2、`max_rounds`、`temperature`、`max_tokens`），把 `max_replans` 写入 `context.metadata["solve_max_replans"]`，其余作为覆盖参数构造 chat pipeline 并运行同一 loop。SolveLoopCapability（`is_active` 由 `solve_mode` 判定）SHALL 注入 solve system prompt block、把工具面保持为 chat 全量（用户 composer 开关照常生效）并在其上追加三件套；对三件套调用 SHALL 服务端注入 `_solve_session_id` 与来自设置的 `max_replans`，模型永不提供 session id。

#### Scenario: solve turn 的工具面
- **WHEN** 用户在 solve 模式发起 turn 且开启了 `web_search`
- **THEN** 挂载工具 = chat 常规组合（含 `web_search`）+ `solve_plan`/`solve_finish_step`/`solve_replan`
- **AND** 每次三件套调用的 kwargs 含服务端注入的 `_solve_session_id`

### Requirement: solve 工具三件套与内存状态机
系统 SHALL 实现单 turn、进程内的 `SolveSession`（有界 LRU 存储，上限 256 个 session，超出淘汰最旧），字段为 `analysis`、`steps`（每步 `{id, goal, done, summary}`）、`replans`、`max_replans`。三个工具语义：
- `solve_plan`：校验模型给出的 `steps`（每项需非空 `goal`），SHALL 服务端生成步骤 id（`S1`、`S2`…），最多保留 12 步；写入 session 后返回 JSON `{status: "planned", analysis, steps, next, instruction}`。空 steps SHALL 返回失败。
- `solve_finish_step`：无计划时 SHALL 失败提示先调 `solve_plan`；未知 `step_id` SHALL 失败并列出合法 id；成功时标记该步 done、记录 summary，返回 `{status: "step_done", completed, next, all_done, instruction}`，且 metadata 附 `_context_checkpoint.summary = "[Sx] <goal> — done. <summary>"` 供 loop 折叠。
- `solve_replan`：预算内（`replans < max_replans`）替换计划并把 `replans` 加一，返回 `{status: "replanned", reason, replans_used, replans_max, steps, next}`；预算耗尽 SHALL 返回失败 JSON `{status: "budget_exhausted", instruction}` 且不改动计划。
三者在无活跃 solve session（`_solve_session_id` 空）时 SHALL 一律返回失败提示。next_step SHALL 返回首个未完成步骤。

#### Scenario: replan 预算耗尽
- **WHEN** `max_replans=2` 且模型第 3 次调用 `solve_replan`
- **THEN** 工具返回 `success=false`、内容为 `budget_exhausted` JSON，session 中的计划与 `replans=2` 保持不变

#### Scenario: 完成全部步骤
- **WHEN** 模型对最后一个未完成步骤调用 `solve_finish_step`
- **THEN** 返回 `all_done: true`、`next: null`，`instruction` 要求直接写最终答案

### Requirement: solve turn 的结果 envelope 与会话延续
solve turn SHALL 与 chat 共用同一 loop 与同一 `emit_capability_result()` 收敛点（envelope 同 chat：`response`/`completed`/`engine: "agent_loop"`/`rounds`/`tool_steps` + `metadata.cost_summary`），SHALL NOT 发出 solve 专属 result 形状。SolveSession 不持久化——计划以 `solve_plan` 工具结果的形式留存在会话消息里，后续普通 chat turn 依然可读到该上下文。（有意差异说明：基线 manifest 未声明 planning/reasoning/writing 阶段，solve 的"阶段感"由工具与 checkpoint 表达，Go 版保持同构，不新增 stage 事件。）

#### Scenario: solve 结束后的后续 chat 追问
- **WHEN** solve turn 完成后用户在同一会话追问某一步细节
- **THEN** 新 turn 的历史消息中包含此前 `solve_plan`/`solve_finish_step` 的 tool 结果 JSON，模型可据此作答，无需 SolveSession 存活
