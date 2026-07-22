# turn-runtime Delta Specification

> 事实源：`docs/golang-req/openspec/specs/turn-runtime/spec.md`（基线 v1.5.2 对等；含 D-002/D-003 有意差异）。

## ADDED Requirements

### Requirement: start_turn 载荷处理与准入
`start_turn(payload)` SHALL 依次完成：(1) capability 默认 `chat`；(2) 从 `config` 剥离 runtime-only 键（`_persist_user_message`、`_regenerate`、`_regenerated_from_message_id`、`_superseded_turn_id`、`followup_question_context`、`subagent_consult_budget`），其余公共 config 按 capability 的 request schema 校验，校验失败 SHALL 拒绝该轮（错误上抛为可发送给客户端的 rejected error）；(3) `ensure_session`（无 session_id 则新建）；(4) persona 解析：payload 显式携带 `persona` 键（含空串 = 清除）时以其为准并持久化到 session preferences，缺席时回退 session 存储偏好；(5) llm_selection 解析：payload → session preference，非空时经多用户模型授权门校验（无权 SHALL 拒绝）；非 admin 且无 selection 时，无任何 LLM grant SHALL 以明确错误拒绝，否则 SHALL 固定为首个已授权且可用的模型；(6) `tools` 为 `None` 时 SHALL 回填用户设置中启用的可选工具（显式传入的列表包括 `[]` 保持原样），随后统一经 admin 施加的 per-user 可选工具白名单过滤；(7) 更新 session preferences（capability/tools/knowledge_bases/language/llm_selection/显式 persona）；(8) 创建 turn 行并启动后台执行。

#### Scenario: config 校验失败
- **WHEN** payload 的公共 config 不符合目标 capability 的 request schema
- **THEN** 不创建 turn，调用方收到 `status: rejected` 的终态错误

#### Scenario: 非 admin 无模型授权
- **WHEN** 非 admin 用户未携带 llm_selection 且账户无任何 LLM grant
- **THEN** 该轮被拒绝，错误文案提示联系管理员分配模型

### Requirement: Turn 状态机（含 waiting_input，有意差异）
turn 持久化状态 SHALL 为：`running`、`waiting_input`、`completed`、`failed`、`cancelled`。（有意差异）基线在 `ask_user` 暂停期间仍标 `running`；Go 版 SHALL 在 loop 进入 `ask_user` 等待时将 turn 置为 `waiting_input`，`submit_user_reply` 接受后回到 `running`。合法迁移：`running ↔ waiting_input`；`running`/`waiting_input` → `completed`/`failed`/`cancelled`（终态不可再迁移）。同一 session SHALL 至多一个非终态 turn：`create_turn` 在存在活跃 turn 时 SHALL 拒绝。对外查询「活跃 turn」的语义 SHALL 同时覆盖 `running` 与 `waiting_input`。

#### Scenario: ask_user 暂停置为 waiting_input
- **WHEN** 某 turn 的 agent loop 触发 `ask_user` 暂停并开始等待回复
- **THEN** 该 turn 持久化状态变为 `waiting_input`
- **AND** 用户提交回复被接受后状态回到 `running`

#### Scenario: 同 session 并发起轮
- **WHEN** session 已有 `running` 或 `waiting_input` 的 turn 时再次 start_turn
- **THEN** 新轮被拒绝（基线错误：`Session already has an active turn`）

### Requirement: seq 分配唯一权威
runtime 的事件发布函数 SHALL 是 turn 内 seq 的唯一分配权威：每个事件发布时补齐 `session_id`/`turn_id`，seq 从 1 起按 turn 单调递增；事件自带 seq>0 时保留原值并把计数器推进到其后。任何旁路写入（如 ContextBuilder 的摘要 trace 事件）都 SHALL 经同一发布函数取号。同一 turn 的事件序 SHALL 全序、无重号。

#### Scenario: 事件统一取号
- **WHEN** 一个 turn 依次发布 SESSION 事件、上下文摘要 progress 事件、loop 事件
- **THEN** 所有事件共享同一 seq 序列（1, 2, 3, …），无跳号来源歧义

### Requirement: 事件持久化（append-before-publish，有意差异）
（有意差异）基线把 turn 事件缓存在内存，仅在终态一次性批量 flush 到 `turn_events` 表——进程崩溃会丢失整轮已推送事件。Go 版 SHALL 采用 append-before-publish 或等价机制：事件在推送给任何订阅者之前（或事务等价地）先追加持久化，保证崩溃后已向客户端提交（publish）过的事件不丢失、重启后 `subscribe_turn` 回放可见。事件持久化后 SHALL 继续镜像一份到该 turn 工作区的 `events.jsonl`（`data/user/workspace` 下按 capability/turn 分目录），镜像失败 SHALL 静默容忍。turn 行被删除（session 删除竞态）时 SHALL 跳过该事件持久化而不级联失败。

#### Scenario: 崩溃后回放不丢已提交事件
- **WHEN** 某 turn 推送了 seq ≤ N 的事件后进程崩溃
- **THEN** 重启后 `subscribe_turn(turn_id, after_seq=0)` 能回放出全部 seq ≤ N 的事件

### Requirement: 后台执行与直播扇出
每个 turn SHALL 以独立后台任务运行：任务持有本 turn 的订阅者集合，事件发布时非阻塞投递到每个订阅者队列（队列满丢弃该订阅者本条而不阻塞流）。start_turn 返回前 SHALL 已发布首条 SESSION 事件（metadata 含 `session_id`/`turn_id`，regenerate 轮附 `regenerated_from_message_id`/`superseded_turn_id`/`regenerate` 标记）。orchestrator 产出的 SESSION 事件 SHALL 被吞掉（runtime 已自行发布）；DONE 事件 SHALL 暂扣，待 assistant 消息与事件持久化、turn 置 `completed` 之后再发布（orchestrator 未产出 DONE 时合成一条 `status: completed` 的 DONE）。

#### Scenario: DONE 后置发布
- **WHEN** capability 流结束
- **THEN** runtime 先持久化 assistant 消息与全部事件、更新 turn 状态为 completed，再向订阅者发布 DONE
- **AND** 客户端收到 DONE 时刷新会话即可看到已落库的 assistant 消息

### Requirement: subscribe_turn(after_seq) 回放
`subscribe_turn(turn_id, after_seq)` SHALL 先读 store 中 seq > after_seq 的持久化事件并按序产出，随后挂接直播队列继续产出后续事件；挂接瞬间 SHALL 原子地补发内存缓冲中 seq 大于已回放最大值的事件，且全程按 seq 去重（不重复、不乱序）。无本地执行时 SHALL 再做一次 store catch-up。订阅结束前若从未产出过 DONE：turn 已终态（`completed`/`failed`/`cancelled` 或不存在）SHALL 合成一条 DONE（metadata 含 `status` 与 `synthesized: true`；`failed` 时先合成携带错误文案的 ERROR 事件）；turn 仍为 `running`/`waiting_input` 时 SHALL NOT 合成 DONE（可能正暂停等待输入或订阅被替换）。

#### Scenario: 断线重连回放
- **WHEN** 客户端携带 `after_seq=42` 重新订阅一个仍在运行的 turn
- **THEN** 先收到持久化/缓冲中 seq>42 的事件，再无缝续接直播事件，无重复 seq

#### Scenario: 订阅已失败的历史 turn
- **WHEN** 订阅一个状态为 `failed` 且回放中无 DONE 的 turn
- **THEN** 依次收到回放事件、合成 ERROR（错误文案）、合成 DONE（`status: failed`）

### Requirement: 孤儿 running turn 恢复
持久化状态为非终态但本进程无对应执行体的 turn（服务器重启遗留）SHALL 被判定为孤儿：start_turn 建新轮前 SHALL 先把该 session 的孤儿轮置为 `failed`（错误文案提示因重启中断、请重试）；`subscribe_turn` 遇到孤儿轮 SHALL 同样先终态化再按终态流程合成收尾事件。执行体存活性判定 SHALL 由 runtime 内存态负责，SHALL NOT 依赖 store。

#### Scenario: 重启后订阅旧轮
- **WHEN** 服务器重启后客户端订阅一个 DB 中仍为 running 的旧 turn
- **THEN** 该 turn 被置为 failed 并回放出错误与合成 DONE，前端得以解除 streaming 状态

### Requirement: cancel_turn
`cancel_turn(turn_id)` SHALL：本进程有存活执行体时取消该任务并等待其收尾完成后返回 true；无执行体但 store 中为非终态时直接置 `cancelled` 返回 true；否则返回 false。取消收尾 SHALL：若 DONE 未发出则依次发布 ERROR（`Turn cancelled`，metadata `turn_terminal: true`、`status: cancelled`）与 DONE（`status: cancelled`）；尽力持久化已产生的部分成果（已流出的答案文本剔除 narration、已生成的文件附件、已捕获的 trace 事件）为一条 assistant 消息；flush 事件；turn 置 `cancelled`。各收尾步骤 SHALL 相互独立容错，状态更新必须执行（否则遗留 running 会被误判为重启孤儿）。取消 SHALL 同时解除 `ask_user` 等待（waiter 得到空值）。

#### Scenario: 取消暂停中的轮
- **WHEN** 用户对一个 `waiting_input` 的 turn 发起取消
- **THEN** loop 的等待被解除、turn 置 `cancelled`，订阅者收到 ERROR + DONE

#### Scenario: 取消保留部分成果
- **WHEN** 取消时模型已流出部分答案且 exec 已生成文件
- **THEN** 部分答案与生成文件作为 assistant 消息（含 `generated: true` 附件）落库

### Requirement: ask_user 暂停 / submit_user_reply 恢复
runtime SHALL 在 orchestrator 启动前为每个 turn 建立回复等待队列，并把「等待用户回复」的可等待句柄经 `context.metadata["wait_for_user_reply"]` 交给 loop。`submit_user_reply(turn_id, text, answers)` SHALL：目标 turn 存在等待队列时入队 `{text, answers}` 载荷（`answers` 为 `[{questionId, text}]` v2 形状，空 `text` 合法）并返回 true；turn 已结束/取消/从未提问时返回 false。回复的格式化与回写进 `role=tool` 消息由 loop 侧完成（见 agent-loop spec）。turn 收尾时 SHALL 先移除队列再清理执行体，使迟到回复确定性地得到 false。

#### Scenario: 对已完成的轮提交回复
- **WHEN** turn 已终态后调用 `submit_user_reply`
- **THEN** 返回 false，调用方（WS 层）可回错误「not awaiting a user reply」

### Requirement: regenerate 与消息分支
`regenerate_last_turn(session_id, overrides)` SHALL：session 有活跃 turn 时报 `regenerate_busy`；无历史 user 消息时报 `nothing_to_regenerate`；否则删除末尾 assistant 消息（若有，并记下其 turn_id 作为被替代轮），以最后一条 user 消息为内容、按「overrides → 该消息的 request snapshot → session preferences」优先级重建载荷，经 start_turn 派发新轮，config 注入 `_persist_user_message=false`、`_regenerate=true`、`_regenerated_from_message_id`、`_superseded_turn_id`（原 user 消息保留不重复落库）。消息分支：payload 显式携带 `parent_message_id` 键（值可为 null）时，新 user 消息 SHALL 挂接到该父节点（形成兄弟分支），本轮 LLM 上下文 SHALL 只取该父节点的祖先链；键缺席时按 legacy 线性追加。assistant 消息 SHALL 链到本轮新建的 user 消息之后（regenerate 未建 user 行时链到显式 parent，或 legacy 追加）。regenerate 轮 SHALL NOT 触发会话标题生成。

#### Scenario: 编辑分支
- **WHEN** 载荷携带 `parent_message_id: 17`（17 已有子消息）
- **THEN** 新 user 消息成为 17 的又一子节点
- **AND** 本轮上下文只包含 17 及其祖先，兄弟分支内容不进入 LLM 上下文

#### Scenario: 有轮在跑时 regenerate
- **WHEN** session 存在活跃 turn 时收到 regenerate
- **THEN** 拒绝并报 `regenerate_busy`

### Requirement: 附件接收、存储与解析
本轮附件 SHALL 逐条规范化（type/url/base64/filename/mime_type/id，缺 id 自动生成）；未托管（无 url）且携带 base64 的附件 SHALL 先把原始字节写入附件存储并记回 url（上传失败非致命，解析仍走内存 base64；base64 非法则跳过上传）。随后 SHALL 对文档类附件做文本抽取（Go 版经 Python Sidecar 的解析契约），抽取文本写回 `extracted_text` 供多模态消息与预览使用。落库副本 SHALL 无条件清空 base64（url 为预览的稳定来源）。

#### Scenario: PDF 附件全链路
- **WHEN** 用户随消息上传一个 base64 PDF
- **THEN** 原始字节入附件存储、`extracted_text` 由 Sidecar 解析填充、消息行中该附件 base64 为空且带可取回的 url

### Requirement: 历史压缩与上下文预算
`ContextBuilder` SHALL 产出有界的 `conversation_history`：历史 token 预算为有效上下文窗口的 35%（下限 256），其中摘要预算 40%、近期消息预算为余量。session 存有 `compressed_summary` 与水位 `summary_up_to_msg_id` 时 SHALL 只取水位之后的未摘要消息；水位不在本分支祖先链上时（编辑分支切换后）SHALL 丢弃存量摘要重建。总量未超预算 SHALL 原样返回（摘要作为 leading `role=system` 条目）。超预算时 SHALL 触发 LLM 摘要：原始前缀 transcript 不超过窗口一半时从原文重建（防摘要套摘要漂移），否则把旧摘要与更早未摘要轮次折叠进新摘要；摘要成功才推进水位并持久化；失败 SHALL 本轮降级（保留旧摘要 + 尽量多的原文，不推水位，下轮重试）。最终仍超预算时 SHALL 从最老的非摘要条目逐条丢弃。摘要过程的 trace 事件（stage `summarize_context` 的 STAGE_START/PROGRESS/CONTENT/STAGE_END）SHALL 经 runtime 发布进本 turn 事件流。

#### Scenario: 分支切换后的摘要水位失效
- **WHEN** 本轮 leaf 的祖先链不含 `summary_up_to_msg_id` 指向的消息
- **THEN** 存量摘要被弃用，摘要从本分支自身消息重建

#### Scenario: 摘要调用失败
- **WHEN** 压缩摘要的 LLM 调用抛错
- **THEN** 本轮使用旧摘要 + 未摘要消息降级组装，水位不变，本轮正常继续

### Requirement: UnifiedContext 组装
runtime SHALL 为每轮组装 `UnifiedContext`：`session_id`、有效 user message、有界 `conversation_history`、`enabled_tools`、`active_capability`、`knowledge_bases`、解析后的 `attachments`、剥离 runtime 键后的 `config_overrides`、`language`；`memory_context` 仅当载荷 `memory_references` 非空时读取 L3 memory 全量拼接；`persona_context` 按「用户自有 persona → 非 admin 回退 admin 工作区 persona」解析（未命中则视为无 persona）；`skills_manifest` 为用户可见 skills 的单行清单 + `always` skills 的完整正文（非 admin 追加 admin 指派 skills）；chat capability 轮 SHALL 构建 source inventory（附件、notebook、book、历史会话、题库条目的统一清单 + 全文索引 `source_index`，历史来源按分支用户轮序标注 `first_seen_turn`），manifest 进 system prompt、全文经 `read_source` 按需读取；非 chat capability SHALL 走 legacy 拼接（`[Attached Documents]`/`[Book Context]`/`[Notebook Context]`/`[History Context]`/`[Question Bank Context]` + `[User Question]` 直接拼进 user message，notebook/history 内容先经分析 agent 压缩）。`metadata` SHALL 至少携带 `turn_id`、`wait_for_user_reply` 句柄、`source_index`、conversation summary 与 token 统计、llm_selection 及生效模型/provider、各类 references。quiz 追问轮（`followup_question_context`）在会话尚无消息时 SHALL 先落一条渲染后的 system 消息。

#### Scenario: chat 轮携带多源
- **WHEN** chat 轮同时带附件、notebook 引用与历史会话引用
- **THEN** context 携带统一 source manifest 与 `source_index`，user message 保持原文不拼接全文

#### Scenario: 非 chat 轮的 legacy 拼接
- **WHEN** `deep_solve` 轮携带 notebook 引用
- **THEN** notebook 内容经分析压缩后以 `[Notebook Context]` 块拼入有效 user message

### Requirement: assistant 消息与 trace 持久化
turn 正常结束时 SHALL 持久化一条 assistant 消息：内容为按序拼接的答案 content 段（仅取无 call_id 或 `call_kind ∈ {agent_loop_round, llm_final_response}` 的 content 事件），SHALL 剔除被 `call_role: narration` 标记的轮的文本，并做 thinking 标签清洗兜底；`events` 字段 SHALL 保存本轮全部事件（`done`/`session` 除外）作为可回放 trace；本轮生成的文件工件（tool_result 的 `tool_metadata.artifacts` 与 SOURCES 中 `type=artifact` 条目，按 url 去重）SHALL 作为 `generated: true` 附件挂到 assistant 消息。用户消息落库时 SHALL 附 request snapshot metadata（content/capability/enabledTools/knowledgeBases/language/attachments/config/各类 references/persona/memoryReferences/llmSelection），供 regenerate 与前端上下文 chips 复原。

#### Scenario: narration 不入答案
- **WHEN** 某轮 narration 文本「Let me search…」随 tool_call 流出、末轮 finish 文本为正式答案
- **THEN** 持久化的 assistant content 只含 finish 文本，narration 仅存在于 events trace 中

### Requirement: 失败路径与终态事件
DONE 尚未发出时发生异常：SHALL 依次发布 ERROR（异常文案，metadata `turn_terminal: true`、`status: failed`）与 DONE（`status: failed`），flush 事件后 turn 置 `failed`。DONE 已发出后的持久化失败：SHALL 只记录错误、尽力 flush 并把 turn 置 `failed`，SHALL NOT 再向订阅者发终态事件。所有路径 SHALL 保证 turn 最终离开非终态。

#### Scenario: capability 异常
- **WHEN** 某轮执行中抛出未捕获异常
- **THEN** 订阅者收到 `turn_terminal` ERROR + `status: failed` 的 DONE，turn 状态为 `failed`

### Requirement: 会话标题生成
非 regenerate 轮完成并发出 DONE 后，若 session 标题仍为「New conversation」哨兵值，系统 SHALL 用首组 user/assistant 消息经 LLM 生成短标题（限时 20s；输出经清洗：去引号/前缀/Markdown 记号/尾标点，截断 80 字符），失败 SHALL 回退为首条 user 消息截断（50 字符）。写入成功后 SHALL 在本 turn 事件流上发布一条 `session_meta` 事件（stage `title`，metadata 含 `title` 与 `session_id`）。用户已改名或已有标题时 SHALL 短路跳过。

#### Scenario: 首轮完成生成标题
- **WHEN** 新会话第一轮完成且标题为哨兵值
- **THEN** DONE 之后订阅者额外收到一条 `session_meta` 标题事件，session 标题更新

#### Scenario: 标题模型超时
- **WHEN** 标题 LLM 调用超过 20s
- **THEN** 使用首条 user 消息截断作为标题，不影响该轮结果
