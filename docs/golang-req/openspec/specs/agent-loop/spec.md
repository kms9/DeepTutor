# agent-loop Specification

## Purpose
本模块定义 chat capability 的单 Agent Loop：一次用户轮 = 在同一条不断增长的会话上跑一个循环，每轮做一次流式 LLM 调用，有 tool_call 则并行分发执行并把 `role=tool` 结果回写会话后继续，无 tool_call 则该轮文本即终答。同时覆盖 context-gated tool mounting（按轮上下文自动挂载工具）、narration/thinking/tool_call/tool_result/sources 等事件的产生时机、轮次预算耗尽的强制收敛，以及 `ask_user` 在 loop 侧的暂停/恢复行为（运行时侧的 wait 队列见 turn-runtime spec）。

- 参考实现（基线）：`deeptutor/agents/chat/agentic_pipeline.py`、`deeptutor/agents/chat/agent_loop.py`、`deeptutor/core/agentic/tool_dispatch.py`、`deeptutor/agents/_shared/tool_composition.py`、`deeptutor/tools/builtin/__init__.py`（工具集合常量）
- 依赖 spec / 里程碑：依赖 runtime-registry、llm-provider、foundation-stream；里程碑：M1

## Requirements

### Requirement: 单循环轮结构与终止判定
系统 SHALL 以「一轮 = 一次 LLM 调用（携带全部已启用工具 schema，`tool_choice: "auto"`）」运行循环：该轮返回 tool_call 时，SHALL 把 assistant 消息（文本 + tool_calls）与全部 `role=tool` 结果追加进会话并继续下一轮（该轮文本为 narration）；该轮不含 tool_call 时即为 finish，其文本（已实时流出）即用户可见终答，循环结束。首轮即无 tool_call 是合法的快速路径。系统 SHALL NOT 设置独立的 respond 阶段——每轮文本一律实时流向用户，由轮完成时的 `call_role` 标记（`narration` / `finish`）告知前端渲染方式。

#### Scenario: 无需探索的快速终答
- **WHEN** 第一轮 LLM 调用返回纯文本、无 tool_call
- **THEN** 循环立即结束，该文本即终答，`call_role` 标记为 `finish`

#### Scenario: 工具轮后继续
- **WHEN** 某轮返回文本 + 2 个 tool_call
- **THEN** assistant 消息与 2 条 `role=tool` 结果按 `tool_call_id` 配对追加进会话
- **AND** 循环进入下一轮，本轮文本标记为 `narration`

### Requirement: 轮次预算与强制收敛
循环轮数上限 SHALL 来自 chat 配置 `max_rounds`（默认 8，下限 1），并 SHALL 尊重 `context.metadata["_min_loop_rounds"]` 地板值（取两者较大者，供 subagent 等 capability 保证余量）。预算耗尽仍在请求工具时，系统 SHALL 强制收敛：发一条 warning progress 事件（预算耗尽提示），向会话追加一条收敛指令 user 消息，再做一次**关闭工具**（不传 tools）的 LLM 调用，其文本为终答。

#### Scenario: 预算耗尽强制收敛
- **WHEN** 连续 `max_rounds` 轮都产生 tool_call
- **THEN** 系统发出 warning progress 事件并追加收敛指令
- **AND** 追加一次不携带 tool schema 的 LLM 调用，其输出作为终答（`call_kind` 为 `llm_final_response`）

### Requirement: 中途失败的挽救收敛
非首轮的 LLM 调用失败（超时/瞬时网络错误）SHALL NOT 丢弃已积累的工具成果：系统 SHALL 走与预算耗尽相同的强制收敛路径（warning 文案区分 error 原因）。首轮失败（尚无任何成果）SHALL 原样向上传播。挽救调用本身也失败时 SHALL NOT 让整轮失败，而是发出协议兜底终答（fallback 文案 content 事件）。

#### Scenario: 第三轮 LLM 调用超时
- **WHEN** 已完成 2 轮工具调用后第 3 轮 LLM 请求失败
- **THEN** 系统发出「a step failed」类 warning 并执行关闭工具的收敛调用，产出终答

#### Scenario: 首轮即失败
- **WHEN** 第一轮 LLM 调用抛错
- **THEN** 错误向调用方传播（由 orchestrator 转为 error 事件）

### Requirement: 空终答处理
finish 轮清洗 thinking 标签后文本为空时：若尚未提示过，系统 SHALL 保留该轮原始文本进会话（模型的计划仍在），追加一条「继续执行或直接作答」的 nudge user 消息并继续循环（每轮上限内只 nudge 一次），同时发出 warning progress 事件；若已 nudge 过或收敛调用产出为空，SHALL 以固定 fallback 文案作为终答并以 content 事件发出（`call_kind: llm_final_response`、metadata 标记 fallback）。

#### Scenario: 模型整轮只输出内部推理
- **WHEN** 某轮无 tool_call 且清洗后的用户可见文本为空，且此前未 nudge 过
- **THEN** 发 warning progress，追加 nudge user 消息，循环继续

### Requirement: 会话消息组装
每轮循环运行前，系统 SHALL 一次性组装本轮会话：`system` prompt（整轮字节稳定，以保 KB 前缀缓存）→ 压缩历史摘要（`conversation_history` 中 leading `role=system` 条目，冠以摘要 header 紧随 system prompt 之后）→ 历史 `user`/`assistant` 消息 → 末尾本轮 user 消息。KB seed、capability pre-loop seed/briefing SHALL 拼入末尾 user 消息而非 system prompt。附件 SHALL 经多模态消息准备（图片内联、文档以 extracted_text 呈现）。

#### Scenario: 携带压缩摘要的会话
- **WHEN** `conversation_history` 首条为 `role=system` 的压缩摘要
- **THEN** 组装结果中该摘要作为带 `[Conversation summary]` 类 header 的 system 消息，位于 system prompt 之后、历史消息之前

### Requirement: context-gated tool mounting
系统 SHALL 按以下顺序组装本轮启用工具表，结果有序去重：
1. 用户 toggle 的工具（`context.enabled_tools` ∩ 可 toggle 白名单：`brainstorm`、`web_search`、`paper_search`、`reason`、`geogebra_analysis`、`imagegen`、`videogen`；未知名跳过）；`enabled_tools=None` 表示未指定（由 turn-runtime 用用户设置回填），`[]` 表示显式全关。
2. 条件自动挂载：`rag` ← 有 rag 类 KB（PageIndex KB 不算，见下）；`read_source` ← 恒为 false（chat 的源阅读由 explore 前置 pass 拥有）；`read_memory` ← 用户有 L3 memory 内容；`list_notebook`/`write_note` ← 用户有 notebook；`read_skill` ← skills manifest 非空；`load_tools` ← deferred 工具池非空；`exec`/`code_execution` ← sandbox 策略允许。
3. 活跃 loop capability 声明的 owned tools。
4. 恒挂载：`write_memory`、`web_fetch`、`github`、`ask_user`、`cron`。
`allowed_builtin_tools` 白名单（非 None 时）SHALL 只裁剪第 2、4 步的内置挂载，SHALL NOT 增加工具；`forced` 工具绕过全部门控追加，`suppressed` 工具最终移除（partner 轮以此换用 partner_* memory 工具）。exclusive capability 活跃时 SHALL 只保留 capability owned tools + `ask_user`。`imagegen`/`videogen` 在对应生成服务未配置模型时 SHALL 从最终列表剔除。

#### Scenario: 有 KB 且用户开启 web_search
- **WHEN** 本轮挂 1 个 rag 类 KB，`enabled_tools=["web_search"]`，用户无 memory/notebook/skills，sandbox 关闭
- **THEN** 启用工具为 `web_search`、`rag`、`write_memory`、`web_fetch`、`github`、`ask_user`、`cron`（顺序如上）

#### Scenario: exclusive capability 接管
- **WHEN** 某 knowledge capability（如 obsidian/subagent）以 exclusive 方式活跃
- **THEN** 启用工具仅为该 capability 的 owned tools 加 `ask_user`，不含用户 toggle 与条件挂载

#### Scenario: 空列表显式关闭可选工具
- **WHEN** `enabled_tools=[]` 且上下文满足 rag 挂载条件
- **THEN** 可选工具全关但 `rag` 与恒挂载工具仍在（toggle 只控制第 1 步）

### Requirement: deferred 工具与 load_tools
标记 `deferred` 的工具（全部 MCP 工具）SHALL NOT 进入初始 schema 列表；system prompt SHALL 携带每个 deferred 工具的一行清单，模型经 `load_tools` 按需加载完整 schema，加载结果 SHALL 在本会话内生效（会话级已加载集持久化）。deferred 池 SHALL 受 caller 白名单（`context.metadata["mcp_tools_filter"]`）与用户 MCP grant 的交集裁剪；partner 轮以 metadata 白名单为准（不查用户 grant）。挂载 PageIndex KB 时，内置 pageindex MCP server 的工具 SHALL 隐式授权并预加载（无需 load_tools 往返）。deferred 准备失败 SHALL 静默降级为无 deferred 工具。

#### Scenario: 模型按需加载 MCP 工具
- **WHEN** 模型调用 `load_tools` 请求某个 deferred 工具
- **THEN** 该工具 schema 加入本轮后续 LLM 调用的 tools 列表，并记入会话级已加载集

### Requirement: KB seed 预检索
循环首轮 LLM 调用前，若挂有 rag 类 KB 且用户消息非空且无 exclusive capability，系统 SHALL 并行对每个 KB 做一次 `rag`（hybrid）检索作为 seed：最多取前 3 个 KB，每 KB 文本截断 4000 字符；命中的 sources SHALL 立即以 `sources` 事件发出；seed 文本拼入末尾 user 消息。检索失败 / 需要重建索引 / 空结果的 KB SHALL 跳过。

#### Scenario: 双 KB seed
- **WHEN** 本轮挂 2 个 rag KB 且均检索成功
- **THEN** user 消息含 KB seed 区块（`[Knowledge Base Context]` header + 每 KB 一节）
- **AND** 循环开始前流上已出现一条包含两个 KB 引用的 `sources` 事件

### Requirement: 流式事件产生时机
每次 LLM 调用 SHALL 依次产生：调用开始一条 `progress`（`trace_kind: call_status`、`call_state: running`、携带 call_id 的 trace metadata）；流式 delta 中 `content` 增量逐段以 `content` 事件发出（`trace_kind: llm_chunk`），`reasoning_content`/`reasoning` 增量以 `thinking` 事件发出；调用结束一条 `progress`（`call_state: complete`、`call_role: narration|finish`，同一 call_id）。工具分发时每个非抑制的 tool_call SHALL 先发 `tool_call` 事件（工具名 + 展示参数，`_` 前缀的服务端注入私有参数 SHALL 剥离），完成后发 `tool_result` 事件（结果文本，metadata 附 `tool_metadata`）。整个 loop SHALL 包裹在 stage `responding` 内（STAGE_START/STAGE_END）；轮内累计的 sources SHALL 在 stage 结束后合并为一条 `sources` 事件发出；最后 SHALL 发出 capability result（`response`/`completed`/`rounds`/`tool_steps` + UsageTracker 成本汇总）。

#### Scenario: 一轮带工具调用的完整事件序列
- **WHEN** 某轮流式返回 narration 文本 + 1 个 tool_call 并执行完成
- **THEN** 事件顺序为：call_status running → 若干 content（llm_chunk）→ call_status complete（call_role=narration）→ tool_call → tool_result

### Requirement: 内联 thinking 标签分流
部分 provider 把推理内联在 content 通道的 `<think>`/`<thinking>` 标签中。系统 SHALL 在流式阶段增量拆分：标签内文本走 `thinking` 事件，标签外走 `content` 事件；跨 chunk 的不完整尾部标签 SHALL 暂扣至下一 chunk（有限回看窗口），流结束时冲刷余量。回写进 LLM 会话的原始文本 SHALL 保留标签不动；终答与持久化文本 SHALL 再做一次标签清洗兜底。

#### Scenario: think 标签跨 chunk 切断
- **WHEN** `</think` 出现在一个 chunk 末尾、`>` 在下一 chunk 开头
- **THEN** 标签被正确识别，其后文本以 `content` 事件发出，不向用户泄漏标签片段

### Requirement: 并行工具分发与去重
一轮内的 tool_call SHALL 并行执行，上限 8 个；超限部分 SHALL 截断并发 warning progress。参数 SHALL 从 tool_call 的 JSON `arguments` 解析（解析失败回退 `{}`），并经服务端 kwargs 注入器补全上下文参数（如 `rag` 默认 `mode=hybrid`、`exec`/`code_execution` 的 sandbox 工作目录与挂载、`cron` 的 owner 路由等）。同批重复调用 SHALL 去重：一般工具以（工具名 + JSON 规范化参数）判重，`ask_user` 更严格——同批第二个及以后的 `ask_user` 一律视为重复（参数不同也算）。重复项 SHALL NOT 真正执行，而以说明性 stub 文本充当其 `role=tool` 结果（保持 tool_call/tool 消息一一配对），重复的 `ask_user` 还 SHALL 从用户可见事件流中抑制（不发 tool_call/tool_result 事件）。工具执行抛异常 SHALL 转为 `success=false` 的错误结果文本回写给模型，SHALL NOT 中断该批其余工具。

#### Scenario: 同批两个相同参数的 web_search
- **WHEN** 模型一条 assistant 消息里发出两个参数完全相同的 `web_search` tool_call
- **THEN** 仅第一个真正执行；第二个得到「duplicate parallel tool_call — skipped」stub 结果
- **AND** 两个 tool_call_id 都有对应 `role=tool` 消息

#### Scenario: 同批两个不同参数的 ask_user
- **WHEN** 模型同批发出两个问题不同的 `ask_user`
- **THEN** 只有第一个生效并可触发暂停；第二个以 stub 回写且不产生用户可见的 Ask 卡片事件

### Requirement: retrieve 类工具的子 trace 与进度流
`rag`（含 seed 检索）、`imagegen`/`videogen`、`consult_subagent` 等长时/检索类工具 SHALL 派生 retrieve 风格 trace metadata：执行前发 `call_state: running` progress（含 query），完成发 `call_state: complete`，失败发 `error` 事件（`call_state: error`）；工具内部进度经 event_sink 以 progress 事件流出（同一子 trace call_id），以保证长任务期间事件流不静默（喂前端 idle 看门狗）。

#### Scenario: videogen 轮询进度
- **WHEN** `videogen` 执行期间通过 event_sink 上报轮询进度
- **THEN** 每条进度以 progress 事件流向客户端，metadata 归属该工具的子 trace call_id

### Requirement: 上下文窗口守护与 checkpoint 折叠
每次 LLM 调用前，系统 SHALL 估算消息 token 总量；超过有效上下文窗口的 90% 时，SHALL 从最早的 `role=tool` 消息开始逐条替换为固定 snip 标记文本，直至回到预算内，并发一条 warning progress。工具结果 metadata 携带 `_context_checkpoint.summary` 时，系统 SHALL 把会话折叠回本轮初始边界并追加一条 `[Context checkpoint]` system 消息（摘要内容），后续轮次基于折叠后的会话继续。

#### Scenario: 长工具输出触发窗口守护
- **WHEN** 组装后的消息估算超过窗口预算
- **THEN** 较早的 tool 消息内容被替换为 snip 标记，客户端收到一条 warning progress

### Requirement: ask_user 暂停与恢复（loop 侧）
分发结果携带 pause 信号时（首个 `pause_for_user` 者胜出；同批其他工具照常执行、结果随行）：系统 SHALL 从 `context.metadata["wait_for_user_reply"]` 取运行时提供的 waiter 并 await 用户回复。waiter 不存在（无运行时托管，如直跑 SDK）时 SHALL 把问题摘要作为终答文本发出并结束循环。回复到达后 SHALL 把回复格式化为「User answered: …」正文（v2 多问答形状逐问渲染）拼接续跑指令，**替换**进匹配 `pause_tool_call_id` 的那条 `role=tool` 消息内容，再发一条 metadata 含 `trace_kind: user_reply`、`ask_user_resolved: true` 的 progress 事件，然后继续同一循环（回复对模型在协议内可见）。waiter 返回空（用户放弃/轮被取消）时 SHALL 结束循环且 `completed=false`——待答问题即本轮最终产物。pause 与 terminate 互斥：pause 生效时 SHALL NOT 触发 terminator 终答。

#### Scenario: 用户回答后同轮续跑
- **WHEN** `ask_user` 触发暂停且用户经 `submit_user_reply` 提交答案
- **THEN** 答案被替换进对应 `role=tool` 消息、发出 `ask_user_resolved` progress
- **AND** 循环在同一 turn 内继续下一轮，最终产出正常终答

#### Scenario: 暂停未获回复
- **WHEN** waiter 返回空值（用户取消）
- **THEN** 循环结束，capability result 的 `completed=false`，无终答文本

### Requirement: provider 兼容性回退
LLM 流式请求失败时系统 SHALL 按错误类型回退重试（各一次）：`stream_options` 不受支持 → 去掉该参数重试；工具 schema 不受支持 → 发 warning progress 后去掉 tools/tool_choice 重试，且本轮剩余轮次不再携带工具；图片输入不受支持且允许降级 → 剥离消息中的图片部件、发 warning 后重试。provider/model 不支持原生 tool calling 时（能力探测），整轮 SHALL 直接不携带 tool schema 运行（工具清单也不注入 system prompt）。

#### Scenario: provider 拒绝工具 schema
- **WHEN** 首轮请求因工具 schema 报错（识别为 schema 不兼容）
- **THEN** 发出 tool_schema_fallback warning，本轮及后续轮次均不带 tools 重试，循环退化为纯文本作答
