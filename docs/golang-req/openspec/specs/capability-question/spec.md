# capability-question Specification

## Purpose
本模块定义 Go 版 `deep_question` capability 与出题相关 API 面的目标行为。capability 将一次用户 turn 路由到三条路径之一：followup（针对单个既有题目的单次问答）、custom（QuestionPipeline：explore → plan → 逐题 quiz loop）、mimic（同一 pipeline，但先把试卷解析为模板并跳过 explore + plan）。API 面包括独立的出题 WS（`/api/v1/question/generate`）、仿题 WS（`/api/v1/question/mimic`）与判题 WS（`/api/v1/question/judge`）。本 spec 以 Python 基线 v1.5.2 行为为对等目标；试卷 PDF 解析（MinerU）经 sidecar-contract 提供。

- 参考实现（基线）：`deeptutor/agents/question/capability.py`、`deeptutor/agents/question/pipeline.py`、`deeptutor/agents/question/mimic_source.py`、`deeptutor/agents/question/history.py`、`deeptutor/agents/question/agents/followup_agent.py`、`deeptutor/agents/question/prompts/{en,zh}/`、`deeptutor/api/routers/question.py`、`deeptutor/api/routers/quiz_judge.py`
- 依赖 spec / 里程碑：依赖 capability-chat-solve（agentic loop 原语）、runtime-registry、llm-provider、sidecar-contract（MinerU 解析）、api-unified-ws；里程碑：M3

## Requirements

### Requirement: capability manifest 与三路分发
系统 SHALL 注册名为 `deep_question` 的 capability：manifest 声明 `stages=["ideation", "generation"]`、`tools_used=[rag, web_search, code_execution]`、`cli_aliases=["quiz"]`。注意：manifest 的 stage 声明是对外描述；实际流上的 stage 事件由各路径决定（见下），Go 版 SHALL 保持同样的名实分离以兼容前端。`run()` 分发规则：`metadata.question_followup_context.question` 非空 → followup 路径；否则按 `config_overrides.mode`（默认 `custom`）选 custom 或 mimic。custom 模式 topic 为空 SHALL 发 `error` 事件并终止。请求参数 SHALL 解析：`num_questions`（默认 1）、`difficulty`、`question_types`（白名单，未知项静默丢弃）、`per_type_counts`（非正数/越界键丢弃）、会话上下文文本、session 的 quiz history。

#### Scenario: followup 上下文触发单次问答
- **WHEN** turn 携带 `question_followup_context={"question": "...", "question_id": "q_3"}`
- **THEN** 走 FollowupAgent 单次调用：在 `stage="generation"` 内流式回答，最终 result payload 为 `{response, mode: "followup", question_id}` 且经 `emit_capability_result()` 附 `cost_summary`

### Requirement: custom pipeline 阶段与 stage 事件
QuestionPipeline SHALL 按三阶段运行，每阶段以 StreamBus stage 上下文包裹（自动发出 `stage_start`/`stage_end`，`source="deep_question"`）：
1. `exploring`：agentic loop 探索题目素材；
2. `planning`：一次 PLAN labeled step 产出模板计划；
3. `quizzing`：逐模板生成题目。
mimic 模式（携带 `templates_override`）SHALL 跳过前两个阶段，直接进入 `quizzing`，并以空 analysis 合成 Plan 包络使下游路径完全一致。计划模板数与请求数不一致时 SHALL 在 `planning` 阶段发 warning progress（`notices.plan_count_mismatch`）。pipeline 任一未捕获异常 SHALL 先以可见失败事件（error + 带 ⚠ 前缀的 content）呈现再抛出。

#### Scenario: custom 一次完整生成
- **WHEN** 用户请求生成 3 道题
- **THEN** 流上依次出现 `exploring`、`planning`、`quizzing` 三对 stage 事件，最后一个 result 事件

### Requirement: explore 阶段（Phase 1）
explore SHALL 是 `THINK / TOOL / FINISH` 标签协议的 agentic loop：默认最大 8 次迭代（`exploring.max_iterations` 可配），native tool calling 可用时挂载与 chat 相同组合策略得到的工具（含 KB 自动挂载 `rag`），`stream_body_live=true`——FINISH 文本作为探索前言实时流入聊天气泡。每个 tool result SHALL 经 Tool Summarizer 反思步骤（默认开启，单独一次 LLM 调用，`max_tokens` 默认 800、temperature 0.2）压缩后替换 loop 缓冲区中的原始 tool 消息；下游 plan / quiz 提示词消费的 `exploration_trace` SHALL 由初始消息之后的迭代消息序列渲染而来（含压缩后的工具结果）。达到迭代上限 SHALL 走 force-finish：发 warning、注入收尾指令、至多 2 次修复尝试要求产出合法 FINISH。

#### Scenario: 工具结果被摘要替换
- **WHEN** explore 中 `rag` 返回 6000 字符的检索结果
- **THEN** 摘要器输出的压缩文本替换缓冲区中的原始结果，`exploration_trace` 只含压缩版

### Requirement: plan 阶段（Phase 2）
plan SHALL 是单次 `PLAN` 标签 labeled step（无工具，`max_tokens` 2000），提示词注入 `user_message`、`exploration_trace`、`num_questions`、`allowed_types`、`per_type_counts`、`difficulty`。解析 SHALL 容错（JSON 提取 + fence 剥离）：接受 `templates` 或 `ideas` 数组；每项取 `topic`（去重，大小写不敏感）与 `question_type`——类型不在允许集合内时回退（优先 `short_answer`，否则允许集合首项，否则 `written`）；difficulty 归一到 `{easy, medium, hard}` 默认 `medium`；`question_id` 服务端顺序生成（`q_1`、`q_2`…）；模板数封顶请求数。题型全集为 `{choice, concept, fill_in_blank, short_answer, written, coding}`。

#### Scenario: 计划含越界类型
- **WHEN** 请求限定 `question_types=["choice"]` 但计划里某项标了 `coding`
- **THEN** 该模板类型被强制回退到允许集合内（`choice`），不产生越界题型

### Requirement: quiz 阶段（Phase 3，逐题）
每道题 SHALL 运行一个 `THINK / TOOL / FINISH` 协议的 loop（每题最多 5 次迭代，挂载同一工具面），FINISH 文本按 JSON 解析为题目 payload。归一化与校验 SHALL 与基线一致：`question_type` 强制回填模板类型；choice 需要恰好 A–D 四个选项且 `correct_answer` 为选项键（正文可反查为键）；concept（判断题）答案归一为小写 `true`/`false`（容忍 T/F、对/错误、yes/no、1/0 等变体）且不得携带 options；fill_in_blank 题面必须含 `____` 且无 options；其余类型不得携带 options 且答案不得形如单字母选项键。校验不过 SHALL 发 warning 并执行一次修复调用（FINISH-only 协议，`max_tokens` 2500），修复后仍有问题则记入 `metadata.issues` 并发 repair_failed warning；题面缺失时兜底为 `[Generation failed] <topic>`，答案/解析缺失兜底 `N/A`。每题完成 SHALL 发出一条 content 事件：正文为该题渲染 markdown，metadata 为 Question 卡片 trace（`call_kind: "quiz_question_emitted"`、`trace_role: "quiz_question"`、`trace_group: "quiz"`、`question_index`、`total_questions`、内嵌完整 `qa_pair`）。

#### Scenario: choice 题选项残缺触发修复
- **WHEN** 某 choice 题首次生成只有 A/B/C 三个选项
- **THEN** 发出 repair warning，执行一次修复调用；修复成功后四选项齐备、issues 清空

### Requirement: 结果 envelope（收敛点）
pipeline 结束 SHALL 经 `emit_capability_result()` 发出唯一 result：payload 为 `{response, summary: {success, source: "exam"|"topic", requested, template_count, completed, failed, templates, results: [{qa_pair, metadata}], analysis}, mode: "mimic"|"custom"}`，`metadata.cost_summary` 由 UsageTracker 合并。`response` 规则：custom 模式放 explore FINISH 前言（题目 markdown 只活在 QuizViewer，避免重复渲染）；mimic 模式（无 Phase 1）回退为题目汇总 markdown，空时为 "No questions generated."。`summary.success` 为全部题目无 error 且非空。

#### Scenario: 部分题目失败的 envelope
- **WHEN** 5 道题中 1 道带 issues
- **THEN** result 的 `summary.completed=4`、`summary.failed=1`、`summary.success=false`，失败题的 `metadata.issues` 保留在 results 中

### Requirement: mimic 模式输入解析
mimic SHALL 支持三种输入：(1) 上传的 PDF 附件——写入临时文件后经 MinerU 解析为模板（解析进度行 SHALL 桥接为 `exploring` 阶段的 thinking 事件实时可见；解析在 sidecar 中执行为有意差异，进度经 sidecar 事件流转发）；(2) `config_overrides.paper_path` 指向的已解析目录——跳过解析直接抽题；(3) 均无但 user_message 含 `[Attached Documents]`——退化为 custom pipeline，并在 user_message 前注入"尽量模仿附带材料"的提示前缀。三者皆无 SHALL 发 error（mimic 需要 PDF 或已解析目录）。解析产出的模板 SHALL 携带 `source: "mimic"` 与原题参考（`reference_question`/`reference_answer`），quiz 步骤据此仿写而非凭空出题；`num_questions` 以模板数为准。MinerU 解析错误 SHALL 以 error 事件呈现并终止。

#### Scenario: 上传 PDF 走完整仿题
- **WHEN** turn 携带 PDF 附件且 `mode="mimic"`、`max_questions=10`
- **THEN** `exploring` 阶段可见解析进度 thinking 事件；解析出 N(≤10) 个模板后直接进入 `quizzing`，result 的 `summary.source="exam"`、`mode="mimic"`

### Requirement: 出题 WS /api/v1/question/generate
系统 SHALL 提供该 WebSocket（WS 层鉴权）：客户端首帧 `{requirement: {knowledge_point, preference, difficulty, question_type}, kb_name, count}`；`requirement` 缺失 SHALL 回 `{type: "error", content: "Requirement is required"}`。服务端 SHALL 依次发送：`{type: "task_id", task_id}`（task id 按 `question_gen` 前缀生成）→ `{type: "status", content: "started"}` → 生成过程中的结构化进度/日志消息（进程日志事件绑定该 task 上下文异步推送）→ `{type: "batch_summary", requested, completed, failed}` → `{type: "complete"}`；异常时 `{type: "error", content}` 并更新 task 状态。生成主体为按 topic 批量出题的协调器流程，输出落盘到该 task 的 question batch 目录。（有意差异）Python 基线经旧版 `AgentCoordinator` 实现该 WS 的生成逻辑；Go 版 SHALL 复用 QuestionPipeline 的 custom 路径实现等价语义，但对客户端的消息序列与字段 SHALL 保持不变。

#### Scenario: 一次 WS 出题会话
- **WHEN** 客户端连上 `/api/v1/question/generate` 并发送合法 requirement
- **THEN** 依次收到 `task_id`、`status(started)`、若干进度消息、`batch_summary`、`complete`，随后连接关闭

### Requirement: 仿题 WS /api/v1/question/mimic
系统 SHALL 提供该 WebSocket（WS 层鉴权）：首帧 `{mode: "upload"|"parsed", ...}`。upload 模式必填 `pdf_data`（base64）+ `pdf_name`：SHALL 先 base64 解码校验、再做文件名/大小/后缀（仅 `.pdf`）安全校验，失败回 error；通过后保存到本次 mimic 批次目录并回 `{type: "status", stage: "upload"|"parsing", content}` 进度。parsed 模式必填 `paper_path`。未知 mode 回 error。执行期间 SHALL 把解析/生成的进度与日志作为 JSON 消息实时推送（基线经 stdout 截取 + 进程日志捕获；（有意差异）Go 版 SHALL 用结构化日志/事件转发实现等价的实时可见性，不做 stdout 截取），逐题结果经 `ws_callback` 事件推送。成功结束发 `{type: "complete"}`；失败发 `{type: "error", content}`。

#### Scenario: parsed 目录仿题
- **WHEN** 客户端发送 `{mode: "parsed", paper_path: "dir", kb_name: "kb", max_questions: 5}`
- **THEN** 服务端跳过解析阶段，推送生成进度与逐题结果，最终收到 `complete`

### Requirement: 判题 WS /api/v1/question/judge
系统 SHALL 提供该 WebSocket（挂载在 `/api/v1` 前缀下、自带 WS 鉴权而非 router 级 HTTP 依赖）：首帧为 `{question(必填), question_type, options, correct_answer, explanation, user_answer, user_answer_images: [{base64|url, filename, mime_type}] | null, user_answer_image/image_filename(legacy 单图), language}`。行为约束：`question` 为空 → error 并关闭；`language` 非 `zh`/`en` 时回退 UI 语言再回退 `en`；`data:` 前缀的 base64 SHALL 剥离头部；文字答案与图片皆空 → error（"No answer to judge"）。判题为单次流式 LLM 调用：语言对应的判题 system prompt + 按字段拼装的 user prompt；有图片且模型支持视觉时 SHALL 构建多模态消息（多图全部注入），不支持视觉时 SHALL 降级为纯文本判题并保留服务可用。服务端消息序列：`{type: "started"}` → 零或多条 `{type: "text", content}` → `{type: "done"}`；异常 `{type: "error", content}`。

#### Scenario: 带手写答案图片判题
- **WHEN** 客户端提交 `user_answer_images` 两张 base64 图片且当前模型支持视觉
- **THEN** 判题调用为多模态消息（两张图全部注入），客户端按 started → text* → done 顺序收流

### Requirement: prompts 资源与模板注入
pipeline 提示词 SHALL 来自 `agents/question/prompts/{en,zh}/pipeline.yaml`（键如 `explore.system`、`plan.user_template`、`quiz_step.system`、`protocol.force_finish`、`notices.*`、`labels.*`、`empty.*`），followup 来自 `followup_agent.yaml`。Go 版 SHALL 原样拷贝并实现 `{var}` 命名占位符注入；system prompt SHALL 追加语言指令（`append_language_directive` 等价行为）；键缺失回退调用点默认文案。

#### Scenario: 语言指令追加
- **WHEN** 以 `language="zh"` 运行 explore
- **THEN** explore system prompt 末尾追加中文作答指令，全流程状态文案取 zh prompts
