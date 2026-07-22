# impl-capability-research — Proposal

## Why

deeptutor-go 需要交付 `deep_research` capability 的 Go 实现：四阶段 pipeline（rephrasing → decomposing → researching → reporting）、outline 预览确认的两段式调用、动态子问题队列（APPEND 扩展）、并行 block 研究、citation 管理与迭代式报告。行为事实源为 `docs/golang-req/openspec/specs/capability-research/spec.md`；按 `docs/golang-req/openspec/ROADMAP.md` 定位属 Wave 3 / 里程碑 M3。spec 已冻结的关键契约：outline 预览 result **不走** `emit_capability_result()` 统一收敛点（无 cost_summary，前端按 `metadata.outline_preview` 识别），这是全项目唯一例外，必须在本 change 内如实保持。

## What Changes

- 新增 `internal/capabilities/research`：`deep_research` capability——manifest（`stages=["rephrasing","decomposing","researching","reporting"]`）、request config 校验（mode/depth/manual_subtopics/manual_max_iterations/confirmed_outline）、两段式调用（首调仅 Phase 1+2 出 outline 预览；确认调用跳过 Phase 1+2）。
- Phase 1 rephrasing：仅挂 `ask_user` 的 THINK/TOOL/FINISH mini loop（迭代 8 / ask_user 3 轮、同一 call_id 归卡）；Phase 2 decomposing：OUTLINE labeled step 容错解析 + 单元素兜底。
- Phase 3 researching：`DynamicTopicQueue`（max_length 8）+ 并行批次调度（series/parallel、max_parallel_topics 3、安全上限）；block loop（THINK/TOOL/APPEND/FINISH）；APPEND 校验去重；tool result 摘要 + `CitationManager` 采集；部分失败降级与 partial 告警。
- Phase 4 reporting：OUTLINE → 确定性 coverage 修复 → H1 → INTRO → 逐节 SECTION → CONCLUSION 迭代产出（`research_status_key`/`report_part` metadata 保持基线取值）；引用编号链接化与 References 渲染（`reference_list` content 事件）。
- 从基线原样拷贝 prompts：`agents/research/prompts/{en,zh}/pipeline.yaml`。

## Capabilities

### New Capabilities
- `capability-research`: deep_research 四阶段 pipeline（两段式 outline 确认、动态队列并行研究、citation 与迭代报告），行为对等基线 v1.5.2。

### Modified Capabilities
（无）

## Impact

- 依赖的其他 change（按 ROADMAP 依赖图）：
  - `impl-capability-chat-solve`（`run_agentic_loop` / LabelProtocol 原语、工具组合策略、`EmitCapabilityResult`）
  - `impl-agent-loop`、`impl-turn-runtime`（ask_user waiter、UnifiedContext）、`impl-runtime-registry`
  - `impl-llm-provider`（eino ChatModel 流式与 labeled step）
  - `impl-api-unified-ws`（两段式调用经统一 WS 的 `research_config` 回传与确认调用）
- 受影响面：前端 Research 页（outline 预览卡、Rephrasing 卡、逐节报告渲染、References 折叠块）；M3 fixtures（`deep_research` stage 事件）。
