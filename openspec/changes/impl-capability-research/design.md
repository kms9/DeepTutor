# impl-capability-research — Design

## Context

`deep_research` 是四阶段 pipeline，且是唯一「两段式」capability：首次调用只跑 rephrasing + decomposing 并以 outline 预览 result 结束 turn；用户确认（可编辑）后携带 `confirmed_outline` 再次调用，跳过前两阶段执行 researching + reporting。researching 阶段是动态队列 + 并行 block loop（支持 loop 内 APPEND 扩展子问题）。事实源：`docs/golang-req/openspec/specs/capability-research/spec.md`。关键契约：**outline 预览 result 不经统一收敛点 `EmitCapabilityResult()`**（无 cost_summary 合并，payload 顶层 `outline_preview: true` + `research_config` 回传）——前端据此识别，Go 版必须保持该例外。

## Goals / Non-Goals

**Goals:**
- 四阶段行为、两段式调用、APPEND 队列、citation/References、partial 降级全部对等基线。
- 并行 block 研究用 goroutine + errgroup 实现 `asyncio.gather` 等价语义（批内并发、事件仍归同一 `researching` stage）。

**Non-Goals:**
- 不实现 ask_user 工具与 waiter（impl-turn-runtime）；不实现证据工具本体（impl-tools-builtin）；不新增 outline 编辑 UI 协议（前端零改动，沿用 `research_config` 字段）。

## Decisions

### D1. Go 包结构

```
internal/capabilities/research/
├── capability.go     # DeepResearchCapability：manifest + Run()（config 校验、两段式分派、outline 预览 result）
├── pipeline.go       # 四阶段编排（顺序函数调用）
├── rephrase.go       # Phase 1：ask_user mini loop（同一 call_id 归卡）
├── decompose.go      # Phase 2：OUTLINE labeled step + 容错解析 + 单元素兜底
├── research.go       # Phase 3：调度器（series/parallel 批次）+ block loop（THINK/TOOL/APPEND/FINISH）
├── queue.go          # DynamicTopicQueue + TopicBlock + APPEND 校验去重
├── citations.go      # CitationManager：采集、编号、链接化、References HTML 安全渲染
├── report.go         # Phase 4：OUTLINE/coverage 修复/INTRO/SECTION/CONCLUSION 迭代产出
├── request_config.go # mode/depth/manual_* /confirmed_outline 校验与 runtime config
└── prompts/          # {en,zh}/pipeline.yaml（基线原样拷贝）
```

### D2. Capability interface 实现签名

```go
type DeepResearchCapability struct { /* deps: llm binding, tool composition, prompts */ }
func (c *DeepResearchCapability) Manifest() core.CapabilityManifest
// stages=["rephrasing","decomposing","researching","reporting"],
// tools_used=[rag, web_search, paper_search, code_execution], cli_aliases=["research"]
func (c *DeepResearchCapability) Run(ctx context.Context, uc *core.UnifiedContext, bus *core.StreamBus) error
// 无 confirmed_outline：Phase1+2 → bus.Result(payload{outline_preview:true, topic, sub_topics,
//   response:"", output_dir:""}, metadata 附 research_config) —— 不经 shared.EmitCapabilityResult
// 有 confirmed_outline：Phase3+4 → shared.EmitCapabilityResult(...)

type DynamicTopicQueue struct{ maxLength int; blocks []*TopicBlock; mu sync.Mutex }
func (q *DynamicTopicQueue) Append(title, overview string) (blockID string, err error) // 空标题/满/相似度重复拒绝
func (q *DynamicTopicQueue) PendingBatch(n int) []*TopicBlock
```

### D3. 阶段编排落位（eino Graph vs 自定义）——取舍结论

**结论：自定义阶段函数 + goroutine 并发，不用 eino Graph。** 论证：(1) 四阶段虽多，但拓扑是「线性 + 一个跨 turn 的断点」——两段式断点发生在**进程间**（首调 turn 结束、确认后新 turn 进入），任何进程内编排框架都表达不了这个断点，Graph 的核心卖点（静态拓扑 + 状态传递）用不上；(2) researching 的并行度是**运行时动态**的（APPEND 随时加块、批次大小随 execution_mode 变化、安全上限轮询），用 `errgroup.Group` + 队列显式表达比把动态 fan-out 硬塞进 Graph 更可审计；(3) 与 chat/question 的结论一致（见各自 design D3），全项目统一「eino 管 LLM 调用（ChatModel.Stream + labeled step 解析），编排用手写 Go」。block loop 复用 impl-capability-chat-solve 交付的标签协议 loop 原语（THINK/TOOL/FINISH + intermediate 标签扩展点 `on_intermediate` 承接 APPEND）。

### D4. stage 事件发射点

- 每阶段 `bus.Stage(name, source="deep_research")` 包裹：`rephrasing` / `decomposing`（stage metadata 含 `research_status_key:"decompose_target"`）/ `researching` / `reporting`（初始 metadata `research_status_key:"report_outline"`、`report_part:"outline"`）。
- 首调仅出现 `rephrasing`、`decomposing` 两对事件即收 outline 预览 result；确认调用仅 `researching`、`reporting` 两对。
- researching 内：APPEND 通过后发 progress 告知新增 topic；结束时有失败 block 则发 `partial_results` warning（`trace_kind:"warning"`）。全部 block 事件在同一 `researching` stage 内（批内 goroutine 共享 stage 上下文，bus 事件序列化由 StreamBus 保证）。
- reporting 内：各子步正文以 content 实时流出、步骤间空行分隔符 content；progress/content 的 `research_status_key` 与 `report_part`（outline/intro/section/conclusion + section_index/section_count/section_title）保持基线取值。References 非空发 `reference_list` content 事件。
- 异常：先 error + ⚠ content 再抛。

### D5. outline 预览不走收敛点（spec 冻结例外）

`Run()` 首调路径直接 `bus.Result(...)`：payload 顶层 `{outline_preview: true, topic, sub_topics: [{title, overview}], response: "", output_dir: ""}`，metadata 附 `research_config`（mode/depth/manual 参数回传供确认调用复用）；**不合并 cost_summary**。确认调用的最终 result 才走 `shared.EmitCapabilityResult`（payload `{response, output_dir:"", metadata:{mode:"agentic_research", topic, block_count, citation_count, partial, failed_block_count, failed_block_titles}}` + cost_summary）。在代码中以显式注释标注该例外引用本 spec Requirement，防后人「顺手统一」。

### D6. 摘要与 citation

block loop 每个 tool result 经一次 note-summarisation 调用（max_tokens 1500）压缩：摘要 + tool_type/query/来源记入 `CitationManager`（生成 citation_id），摘要替换回传模型的 tool 消息；摘要失败降级保留原文 + warning 日志。`MAX_PARALLEL_TOOL_CALLS` 约束单迭代并行工具数。`tool_traces` 保留供 Phase 4 evidence 渲染（单 block 4000 字符 / 总量 12000 字符上限）。References 渲染：首现顺序编号 → 正文锚点链接化 → `<details id="references" open>` + `<ol>`，用户可控字段 HTML 转义（`html.EscapeString`）。

### D7. 与 Python 基线文件映射表

| Python 基线 | Go 落位 |
| --- | --- |
| `deeptutor/agents/research/capability.py` | `internal/capabilities/research/capability.go` |
| `deeptutor/agents/research/pipeline.py` | `internal/capabilities/research/{pipeline,rephrase,decompose,research,report}.go` |
| `deeptutor/agents/research/request_config.py` | `internal/capabilities/research/request_config.go` |
| `deeptutor/agents/research/queue.py` | `internal/capabilities/research/queue.go` |
| `deeptutor/agents/research/citations.py` | `internal/capabilities/research/citations.go` |
| `deeptutor/agents/research/prompts/{en,zh}/pipeline.yaml` | `internal/capabilities/research/prompts/{en,zh}/pipeline.yaml`（原样拷贝） |

## Risks / Trade-offs

- [并行 block 的事件交错顺序与基线 asyncio 调度不完全一致] → fixtures 验收锚定「stage 边界 + 每 block 内事件顺序 + 离散事件字段」，块间交错列入白名单（acceptance.md 3.1 机制）；series 模式 fixture 做全序对齐兜底。
- [APPEND 相似度去重的算法阈值需与基线一致] → 实现时对照基线 `queue.py` 的具体度量（复用 impl-capability-mastery 的 ratcliff 实现如基线用 difflib），golden case 固化。
- [coverage 修复的 token 重叠度配对属确定性算法但细节多] → 纯函数化 + 表驱动单测（无效 id/空 section/未覆盖 block/附录节四分支）。
- [两段式之间的 config 回传丢失导致确认调用参数漂移] → `research_config` 字段集合契约测试 + 统一 WS fixture 覆盖「首调 → 确认」全链。

## Migration Plan

实现顺序：queue/citations（纯逻辑单测）→ labeled step 各阶段（replay provider）→ 并行调度 → 两段式接线 → fixtures。回滚 = 不注册 capability。

## Open Questions

- 基线 APPEND 去重的相似度实现（difflib ratio vs 归一化词集）需读 `queue.py` 确认后固化 golden——列入 tasks 1.2。
