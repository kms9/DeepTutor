# impl-capability-question — Design

## Context

`deep_question` 把一次 turn 路由到三条路径：followup（单次流式问答）、custom（explore → plan → 逐题 quiz 的 QuestionPipeline）、mimic（试卷解析为模板后直入 quizzing）。另有三个 legacy WS 面（generate / mimic / judge）独立于统一 WS。事实源：`docs/golang-req/openspec/specs/capability-question/spec.md`。关键契约：manifest 声明 `stages=["ideation","generation"]` 仅为对外描述，实际流上 stage 事件为 `exploring`/`planning`/`quizzing`（custom）或 `generation`（followup）——名实分离必须保持；`/api/v1/question/generate` 按已登记差异 **D-008**（acceptance.md §6）换实现保消息面。

## Goals / Non-Goals

**Goals:**
- capability 三路分发与 QuestionPipeline 三阶段行为对等基线；题目归一/校验/修复规则逐条一致。
- 三个 legacy WS 端点消息序列逐字段对等（gorilla/websocket，gin handler 内 upgrade，WS 层鉴权）。
- mimic 的 MinerU 解析走 Sidecar，解析进度实时桥接为 `exploring` thinking 事件（spec 已登记有意差异）。

**Non-Goals:**
- 不实现 loop 原语 / 工具本体 / MinerU 解析器本身（Sidecar 契约属 impl-sidecar-contract）；不迁移旧版 `AgentCoordinator`（D-008 明确消除双实现）；不做 QuizViewer 前端改动。

## Decisions

### D1. Go 包结构

```
internal/capabilities/question/
├── capability.go    # DeepQuestionCapability：manifest + Run() 三路分发 + 参数解析
├── pipeline.go      # QuestionPipeline：exploring/planning/quizzing 三阶段编排
├── explore.go       # Phase 1：THINK/TOOL/FINISH loop + Tool Summarizer + force-finish
├── plan.go          # Phase 2：PLAN labeled step + 容错解析/类型回退/q_N id
├── quiz.go          # Phase 3：逐题 loop、归一校验、修复调用、兜底
├── mimic_source.go  # mimic 输入解析（PDF→Sidecar / paper_path / 退化 custom）
├── followup.go      # FollowupAgent 单次流式问答
├── history.go       # session quiz history 读取
└── prompts/         # {en,zh}/pipeline.yaml、followup_agent.yaml（基线原样拷贝）
internal/api/routers/
├── question_ws.go   # /api/v1/question/generate + /api/v1/question/mimic（gorilla upgrade）
└── quiz_judge_ws.go # /api/v1/question/judge
```

### D2. Capability interface 实现签名

```go
type DeepQuestionCapability struct { /* deps: llm binding, tool composition, sidecar client */ }
func (c *DeepQuestionCapability) Manifest() core.CapabilityManifest
// stages=["ideation","generation"]（仅对外描述）, tools_used=[rag, web_search, code_execution], cli_aliases=["quiz"]
func (c *DeepQuestionCapability) Run(ctx context.Context, uc *core.UnifiedContext, bus *core.StreamBus) error
// 分发：metadata.question_followup_context.question 非空 → followup；否则 config_overrides.mode（默认 custom）
// custom 且 topic 空 → error 事件终止；解析 num_questions/difficulty/question_types(白名单)/per_type_counts(越界丢弃)

type QuestionPipeline struct { ... }
func (p *QuestionPipeline) Run(ctx, uc, bus, req PipelineRequest) (*PipelineResult, error)
// req.TemplatesOverride 非空（mimic）→ 跳过 explore/plan，以空 analysis 合成 Plan 包络
```

### D3. 阶段编排落位（eino Graph vs 自定义）——取舍结论

**结论：自定义阶段函数**（pipeline.go 内顺序调用 explore → plan → quiz），不用 eino Graph。理由：(1) 三阶段是**线性**流程，唯一分支是 mimic 跳段——一个 `if` 即可，Graph 拓扑无增益；(2) explore/quiz 内部是标签协议（THINK/TOOL/FINISH）的迭代 loop，含 force-finish 修复、Tool Summarizer 就地替换缓冲区消息等可变状态操作，与 chat loop 同类（见 impl-capability-chat-solve design D3 结论，保持全项目一致）；(3) 阶段间传递的 `exploration_trace` 是纯字符串渲染，无需 Graph 的 state 通道。eino 只承担 LLM 调用：labeled step 与 quiz loop 均经 llm-provider 的 eino ChatModel（`Stream`），标签协议解析在流出口做。

### D4. stage 事件发射点

- custom：`bus.Stage("exploring", source="deep_question")` → `bus.Stage("planning", ...)`（模板数不符时段内发 `plan_count_mismatch` warning progress）→ `bus.Stage("quizzing", ...)`；三对 stage_start/stage_end。
- mimic：仅 `quizzing` 一对（PDF 解析进度以 `exploring` 阶段 thinking 事件桥接——解析发生在 pipeline 之前，stage 上下文按基线行为对齐 fixtures）。
- followup：仅 `generation` 一对，FollowupAgent 文本流式其中。
- 每题完成在 `quizzing` 内发 content 事件（`call_kind: "quiz_question_emitted"`、`trace_role: "quiz_question"`、`trace_group: "quiz"`、`question_index`/`total_questions`、内嵌 `qa_pair`）。
- 未捕获异常：先 error 事件 + ⚠ 前缀 content，再向 orchestrator 抛出。
- result：经 `shared.EmitCapabilityResult`——payload `{response, summary{success, source, requested, template_count, completed, failed, templates, results, analysis}, mode}`；custom 的 response 为 explore FINISH 前言，mimic 回退题目汇总 markdown。

### D5. 差异 D-008：generate WS 复用 QuestionPipeline

`/api/v1/question/generate` 不移植旧 `AgentCoordinator`：handler 组装 requirement → 走 QuestionPipeline custom 路径 → 把内部 StreamBus 事件转译为基线消息面 `{task_id} → {status: started} → 进度/日志 JSON → {batch_summary: {requested, completed, failed}} → {complete}`；task id 按 `question_gen` 前缀生成，产物落该 task 的 question batch 目录。消息序列与字段以录制 fixtures 验证不变（acceptance.md D-008 行：前端影响无）。

### D6. legacy WS 实现要点（gin + gorilla/websocket）

- 三端点均在 gin handler 内 `upgrader.Upgrade`，WS 层鉴权（首帧前校验，失败即关闭）；写侧单 goroutine 序列化（gorilla 并发写限制）。
- mimic WS：`mode=upload` 先 base64 解码校验 → 文件名/大小/`.pdf` 后缀安全校验 → 存 mimic 批次目录，`{type:"status", stage:"upload"|"parsing"}` 进度；`mode=parsed` 必填 `paper_path`；未知 mode error。（有意差异，已写入 spec）进度可见性用结构化事件转发实现，不做基线的 stdout 截取。
- judge WS：`question` 空 → error 关闭；`language` 归一（zh/en → UI 语言 → en）；`data:` 头剥离；文字与图片皆空 error；模型支持视觉时多模态消息（多图全注入），否则降级纯文本；消息序列 `started → text* → done`。
- MinerU 解析：`sidecar.ParseDocument`（impl-sidecar-contract 契约），sidecar 事件流的进度行经回调桥接为 thinking/status 消息。

### D7. quiz 归一校验规则落位

`quiz.go` 内实现确定性校验器（纯函数，可单测）：`question_type` 强制回填模板类型；choice 恰好 A–D 且 `correct_answer` 可归一为选项键；concept 答案归一 `true`/`false`（T/F、对/错、yes/no、1/0 变体）、禁 options；fill_in_blank 必含 `____`、禁 options；其余禁 options 且答案不得形如单字母键。失败 → warning + 一次修复调用（FINISH-only，max_tokens 2500）→ 仍失败记 `metadata.issues` + repair_failed warning；题面兜底 `[Generation failed] <topic>`，答案/解析兜底 `N/A`。

### D8. 与 Python 基线文件映射表

| Python 基线 | Go 落位 |
| --- | --- |
| `deeptutor/agents/question/capability.py` | `internal/capabilities/question/capability.go` |
| `deeptutor/agents/question/pipeline.py` | `internal/capabilities/question/{pipeline,explore,plan,quiz}.go` |
| `deeptutor/agents/question/mimic_source.py` | `internal/capabilities/question/mimic_source.go` |
| `deeptutor/agents/question/history.py` | `internal/capabilities/question/history.go` |
| `deeptutor/agents/question/agents/followup_agent.py` | `internal/capabilities/question/followup.go` |
| `deeptutor/agents/question/prompts/{en,zh}/` | `internal/capabilities/question/prompts/{en,zh}/`（原样拷贝） |
| `deeptutor/api/routers/question.py` | `internal/api/routers/question_ws.go`（generate 部分按 D-008 换芯） |
| `deeptutor/api/routers/quiz_judge.py` | `internal/api/routers/quiz_judge_ws.go` |

## Risks / Trade-offs

- [D-008 换实现后进度/日志消息的条数与措辞与基线不完全一致] → 验收锚定「消息类型序列 + 关键字段」而非逐条日志文本（acceptance.md 3.1 错误文案白名单同理）；fixtures 录制时标注可忽略段。
- [Tool Summarizer 额外 LLM 调用引入不确定性] → 契约测试用 replay provider 固定摘要输出；摘要失败降级保留原文（与基线一致）。
- [gorilla/websocket 并发写 panic] → 每连接单写 goroutine + channel 汇聚，评审清单项。
- [Sidecar 解析大 PDF 超时] → 契约层带超时与进度心跳；解析错误按 spec 以 error 事件终止。

## Migration Plan

实现顺序：pipeline（mock provider 单测）→ capability 分发 → legacy WS 三端点 → fixtures 对齐。三个 legacy WS 与统一 WS 互不影响，可独立灰度；回滚 = 不挂路由。

## Open Questions

- generate WS 的「结构化进度/日志消息」在 D-008 下的最小保真集合（前端实际消费哪些字段）——以录制 fixtures + 前端代码走查确定。
