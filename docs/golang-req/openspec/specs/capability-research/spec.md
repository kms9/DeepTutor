# capability-research Specification

## Purpose
本模块定义 Go 版 `deep_research` capability 的目标行为：四阶段 pipeline（rephrasing → decomposing → researching → reporting），含 outline 预览确认的两段式调用、动态子问题队列（支持 loop 内 APPEND 扩展）、并行 block 研究、来源引用管理与迭代式报告产出。工具面复用 chat 的组合策略（用户开关 + KB 自动挂载 `rag`），无独立 sources 开关。本 spec 以 Python 基线 v1.5.2 行为为对等目标。

- 参考实现（基线）：`deeptutor/agents/research/capability.py`、`deeptutor/agents/research/pipeline.py`、`deeptutor/agents/research/request_config.py`、`deeptutor/agents/research/queue.py`、`deeptutor/agents/research/citations.py`、`deeptutor/agents/research/prompts/{en,zh}/pipeline.yaml`
- 依赖 spec / 里程碑：依赖 capability-chat-solve（`run_agentic_loop` / LabelProtocol 原语）、runtime-registry、llm-provider、api-unified-ws；里程碑：M3

## Requirements

### Requirement: capability manifest 与两段式 outline 确认流程
系统 SHALL 注册名为 `deep_research` 的 capability：manifest 声明 `stages=["rephrasing", "decomposing", "researching", "reporting"]`、`tools_used=[rag, web_search, paper_search, code_execution]`、`cli_aliases=["research"]`。`run()` SHALL 校验 request config（`mode`、`depth`、`manual_subtopics`、`manual_max_iterations`、`confirmed_outline`）并构建 runtime config。首次调用（无 `confirmed_outline`）SHALL 只跑 Phase 1+2，随后 capability 以 `stream.result()` 发出 outline 预览事件：payload 顶层含 `outline_preview: true`、`topic`（精炼后）、`sub_topics: [{title, overview}]`、`response: ""`、`output_dir: ""`，并附 `research_config`（回传 `mode`/`depth`/manual 参数供确认调用复用）；此路径 SHALL NOT 经 `emit_capability_result()`（无 cost_summary 合并，前端按 `event.metadata.outline_preview` 识别）。第二次调用携带 `confirmed_outline` SHALL 跳过 Phase 1+2 直接执行 Phase 3+4。

#### Scenario: 首次调用返回 outline 预览
- **WHEN** 用户发起 deep_research 且 config 无 `confirmed_outline`
- **THEN** 流上出现 `rephrasing`、`decomposing` 两对 stage 事件后即收到 `outline_preview` result，turn 结束，无 `researching`/`reporting` 阶段

#### Scenario: 确认后恢复执行
- **WHEN** 前端携带用户编辑后的 `confirmed_outline` 再次调用
- **THEN** pipeline 跳过 Phase 1+2，直接进入 `researching` 与 `reporting` 阶段并产出最终报告

### Requirement: Phase 1 — rephrasing（ask_user 澄清 mini loop）
`rephrasing` 阶段 SHALL 以 StreamBus stage 上下文包裹（自动 `stage_start`/`stage_end`，`source="deep_research"`）。行为：`planning.rephrase.enabled`（默认 true）关闭或 `ask_user` 不在 registry 时 SHALL 直接返回原 topic（trim 后）。否则运行 `THINK / TOOL / FINISH` 协议的 mini agentic loop：唯一可用工具为 `ask_user`，迭代上限默认 8，`ask_user` 轮次上限 3（每轮最多 3 个问题合并为一张卡片）。host SHALL：拒绝任何非 `ask_user` 的 tool call（注入指导性 tool 消息）；轮次耗尽后对再次 `ask_user` 回复"用现有信息 FINISH"的 tool 消息；全部迭代复用同一 `call_id` 使前端把 ask_user 前后的 THINK 归入同一张 Rephrasing 卡片。FINISH 后文本为精炼 topic；loop 未产出合法 FINISH SHALL 回退原 topic。图片附件 SHALL 注入 Phase 1/2 的多模态消息。

#### Scenario: 模糊 topic 触发澄清
- **WHEN** 用户输入宽泛主题且模型调用 `ask_user` 提出 2 个澄清问题
- **THEN** turn 暂停等待用户回复，恢复后模型基于回答 FINISH 出精炼 topic；超过 3 轮后模型被指示直接 FINISH

### Requirement: Phase 2 — decomposing（大纲分解）
`decomposing` 阶段 SHALL 是一次 `OUTLINE` 标签 labeled step（无工具，`max_tokens` 2000，stage metadata 含 `research_status_key: "decompose_target"`）：提示词注入 `topic` 与 `num_subtopics`（默认 5，depth/manual 可调）。解析 SHALL 容错：接受 JSON 数组或 `{sub_topics|subtopics: [...]}`；每项取 `title`（或 `topic`）与 `overview`（或 `description`），字符串项视为纯 title；数量封顶 `initial_subtopics`；解析失败或为空 SHALL 以原 topic 为唯一 sub-topic 兜底，保证 outline 预览 UX 始终可用。

#### Scenario: 分解 JSON 解析失败
- **WHEN** OUTLINE 步骤返回无法解析的文本
- **THEN** 返回单元素大纲 `[{title: <topic>, overview: ""}]`，流程不中断

### Requirement: Phase 3 — researching（动态队列 + 并行 block loop）
`researching` 阶段 SHALL 用 `DynamicTopicQueue`（`max_length` 默认 8）承载 confirmed outline 的每个 sub-topic 为一个 TopicBlock。调度器 SHALL 循环取 pending blocks：`researching.execution_mode` 为 `series` 时批大小 1，否则（默认 `parallel`）批大小为 `max_parallel_topics`（默认 3），批内以并发（`asyncio.gather` 等价）执行；APPEND 新增的 block 自然进入后续批次；安全上限 `max(20, queue_max_length*4)` 轮防失控。单个 block SHALL 运行 `THINK / TOOL / APPEND / FINISH` 协议的 agentic loop：迭代上限默认 5（有工具时至少 4），`max_tokens` 6000，temperature 0.3，`stream_body_live=false`；工具面为 chat 组合策略产出中的证据型研究工具（`rag`/`web_search`/`paper_search`/`code_execution` 等；`ask_user`、`write_memory` 等便捷工具 SHALL NOT 挂载）。FINISH 后文本为该 block 的 consolidated knowledge；异常或迭代耗尽（force-finish 至多 3 次修复后仍失败）SHALL 标记 block failed 并以空 knowledge 兜底。

#### Scenario: 并行批次执行
- **WHEN** confirmed outline 有 5 个 sub-topic 且 `execution_mode="parallel"`、`max_parallel_topics=3`
- **THEN** 第一批并发研究 3 个 block，完成后第二批研究剩余 2 个；全部 block 的 stage 事件均在同一个 `researching` stage 内

### Requirement: APPEND 队列扩展与去重
block loop 中 `APPEND` SHALL 为 intermediate 标签，由 host 的 `on_intermediate` 处理：payload 第一行为 title（剥离 markdown 标题符），其余为可选 overview。SHALL 依次执行校验并把结论作为下一轮 user 消息注入 loop：title 为空 → 拒绝；队列已满 → 拒绝并指示继续当前 block；与既有 block 标题过近（相似度去重）→ 拒绝并给出既有 block id。通过后 SHALL 追加子 block 并发一条 `researching` 阶段的 progress 事件告知新增 topic。最终 researched 列表 SHALL 按队列顺序排列（父块在 APPEND 子块之前）。

#### Scenario: APPEND 重复主题被拒
- **WHEN** 某 block loop 发出与现存 block 标题几乎相同的 APPEND
- **THEN** 队列不变，loop 收到含既有 block id 的拒绝消息并继续研究当前 block

### Requirement: 工具结果摘要与 citation 采集
block loop 的每个 tool result SHALL 经一次 note-summarisation LLM 调用（`max_tokens` 1500）压缩为摘要：摘要连同 `tool_type`/`query`/来源信息记入 `CitationManager`（生成 citation_id），并以摘要替换回传给模型的原始 tool 消息内容以控制上下文预算；摘要失败 SHALL 降级保留原始内容（记 warning 日志）。单次迭代的并行 tool call 数 SHALL 受 `MAX_PARALLEL_TOOL_CALLS` 上限约束，超限部分拒绝。block 的 `tool_traces`（citation_id、tool_type、query、summary）SHALL 保留供 Phase 4 渲染 evidence。

#### Scenario: web_search 结果入引用库
- **WHEN** 某 block 调用 `web_search` 得到 5 条结果
- **THEN** 结果被摘要化，`CitationManager` 记录该次调用为一条可引用来源，loop 缓冲区中只保留摘要

### Requirement: 部分失败的降级与可见告警
`researching` 结束后 SHALL 检查未达 COMPLETED 的 block：存在时 SHALL 发一条 `researching` 阶段、`metadata.trace_kind="warning"` 的 progress 事件（`notices.partial_results` 文案，含 failed/total 数），并继续用剩余证据写报告；最终 result 的 `metadata` SHALL 携带 `partial: true`、`failed_block_count`、`failed_block_titles`，使调用方能区分完整成功与部分成功。pipeline 任一未捕获异常 SHALL 先发 error 事件 + 带 ⚠ 前缀的 content（labelled error trace card）再抛出。

#### Scenario: 一个 block 失败仍出报告
- **WHEN** 5 个 block 中 1 个迭代耗尽未 FINISH
- **THEN** 出现 partial_results warning，报告基于 4 个 block 生成，result metadata `partial=true`、`failed_block_titles` 含该 block 标题

### Requirement: Phase 4 — reporting（迭代式报告产出）
`reporting` 阶段（stage metadata 初始含 `research_status_key: "report_outline"`、`report_part: "outline"`）SHALL 按子步骤顺序执行，每步为一次 labeled step 且正文实时流入聊天气泡（content 事件），步骤间以空行分隔符 content 衔接：
1. `OUTLINE`（max_tokens 2000）：基于每个 block 的标题 + 知识片段（首段 ≤400 字符）产出 `{title, sections: [{id, title, intent, block_ids}]}`；解析容错，block_ids 支持按 id 或标题反查；解析失败回退每 block 一节。随后 SHALL 执行确定性 coverage 修复：无效 id 过滤、空 section 按 token 重叠度配最佳 block、未覆盖 block 归入最相近 section、仍无归属者聚合为 "Additional findings" 附录节。
2. 标题块（`# <title>` H1）→ `INTRO`（max_tokens 3000，metadata `report_part: "intro"`）→ 逐节 `SECTION`（max_tokens 6000，metadata 含 `report_part: "section"`、`section_index`、`section_count`、`section_title`；evidence 按节渲染，单 block 上限 4000 字符、总量 12000 字符）→ `CONCLUSION`（max_tokens 3000，`report_part: "conclusion"`，输入为各节首段 recap ≤300 字符）。
progress/content 事件的 `research_status_key` 与 `report_part` 字段 SHALL 保持基线取值，供前端渲染分步状态。

#### Scenario: outline 遗漏 block 被修复
- **WHEN** 报告 OUTLINE 的 sections 未引用某个 researched block
- **THEN** coverage 修复把该 block 并入重叠度最高的 section（或追加 Additional findings 节），报告不丢证据

### Requirement: 引用编号、链接化与 References 渲染
报告正文拼装后 SHALL：按引用首次出现顺序为 citation_id 分配序号，把正文中的引用标记链接化为锚点 markdown 链接；随后渲染 References 块——`<details id="references" open>` 包裹 `<summary>`（i18n heading）+ `<ol>` 条目，条目由 CitationManager 的 HTML 安全格式化产出（未知类型回退 tool/query/note 摘要且 SHALL HTML 转义用户可控字段）。References 非空时 SHALL 以独立 content 事件发出（`metadata.trace_kind="reference_list"`），并拼入最终 `response`（`body + "\n\n" + references`）。

#### Scenario: 报告含引用
- **WHEN** 报告正文引用了 3 条来源
- **THEN** 引用按首现顺序编号 1–3 并链接到 References 锚点，流上出现一条 `reference_list` content 事件

### Requirement: 结果 envelope（收敛点）
Phase 3+4 完整执行后 SHALL 经 `emit_capability_result()` 发出唯一 result：payload 为 `{response: <报告全文含 References>, output_dir: "", metadata: {mode: "agentic_research", topic, block_count, citation_count, partial, failed_block_count, failed_block_titles}}`，UsageTracker 的 `cost_summary` 由收敛点合并进 `metadata.cost_summary`。

#### Scenario: 最终 result 事件
- **WHEN** reporting 阶段完成
- **THEN** 收到唯一 result 事件，`metadata.mode="agentic_research"`、`metadata.cost_summary` 汇总全 pipeline（rephrase/decompose/block/摘要/报告各步）的 LLM 用量

### Requirement: prompts 资源与模板注入
全部提示词 SHALL 来自 `agents/research/prompts/{en,zh}/pipeline.yaml`（键组：`rephrase.*`、`decompose.*`、`research_step.*`、`report.outline|intro|section|conclusion.*`、`protocol.*`、`notices.*`、`labels.*`、`empty.*`、`system.*`）。Go 版 SHALL 原样拷贝 prompts 并实现相同 `{var}` 占位符注入（如 `research_step.system` 注入 `topic`/`block_title`/`block_overview`/`mode`/`max_iterations`/`kb_note`/`tool_list`）；block system prompt SHALL 追加语言指令；KB 存在时 SHALL 注入 `kb_system_note`（约束 `rag` 的 `kb_name` 实参）；prompts 加载失败 SHALL 记 warning 并逐键回退默认文案。

#### Scenario: KB 挂载时的 system note
- **WHEN** turn 附带 KB `physics-101`
- **THEN** 每个 block 的 system prompt 含 kb_system_note，指明调用 `rag` 时 `kb_name` 必须为 `'physics-101'`
