# impl-capability-research — Tasks

## 1. 纯逻辑组件（可独立先行）

- [ ] 1.1 实现 `request_config.go`：mode/depth/manual_subtopics/manual_max_iterations/confirmed_outline 校验与 runtime config 构建
- [ ] 1.2 实现 `queue.go`：`DynamicTopicQueue`（max_length 8、APPEND 校验：空标题/满/标题相似度去重——对照基线 `queue.py` 度量并固化 golden case）、按队列顺序输出 researched 列表
- [ ] 1.3 实现 `citations.go`：CitationManager 采集（citation_id 生成）、首现顺序编号、正文锚点链接化、References `<details>/<ol>` 渲染与 HTML 转义（未知类型回退 tool/query/note 摘要）
- [ ] 1.4 拷贝 `agents/research/prompts/{en,zh}/pipeline.yaml` 接入 PromptStore（`research_step.system` 的 topic/block_title/…/tool_list 注入、语言指令追加、`kb_system_note`、加载失败逐键回退）

## 2. Phase 1+2 与两段式

- [ ] 2.1 实现 rephrase mini loop：仅挂 `ask_user`、迭代 8 / ask_user 3 轮（每轮 ≤3 问合并）、非 ask_user tool call 拒绝注入指导消息、轮次耗尽指示 FINISH、同一 call_id 归卡、关闭/无 ask_user 时原样返回 topic、无合法 FINISH 回退原 topic、图片附件多模态注入
- [ ] 2.2 实现 decompose：OUTLINE labeled step（无工具、max_tokens 2000、stage metadata `research_status_key:"decompose_target"`）、容错解析（数组或 sub_topics|subtopics、title|topic、overview|description、字符串项纯 title、封顶 initial_subtopics）、失败单元素兜底
- [ ] 2.3 实现两段式分派：首调 Phase1+2 → outline 预览 result（`outline_preview:true` + `research_config` 回传，**不经** `EmitCapabilityResult`，代码注释标注 spec 例外）；确认调用跳段直入 Phase3+4

## 3. Phase 3 researching

- [ ] 3.1 实现调度器：series（批 1）/ parallel（批 `max_parallel_topics` 默认 3，errgroup 并发）、APPEND 块进入后续批、安全上限 `max(20, queue_max_length*4)` 轮
- [ ] 3.2 实现 block loop：THINK/TOOL/APPEND/FINISH 协议（迭代默认 5、有工具时至少 4、max_tokens 6000、temperature 0.3、`stream_body_live=false`）、证据型工具面（排除 ask_user/write_memory 等）、force-finish 3 次修复、失败 block 空 knowledge 兜底
- [ ] 3.3 实现 `on_intermediate` APPEND 处理：校验结论作为下轮 user 消息注入、通过后发 progress 告知新增 topic
- [ ] 3.4 实现摘要 + citation 采集：note-summarisation 调用（max_tokens 1500）替换 tool 消息、失败降级保留原文、`MAX_PARALLEL_TOOL_CALLS` 上限、`tool_traces` 保留
- [ ] 3.5 实现部分失败降级：`partial_results` warning（failed/total）、result metadata `partial/failed_block_count/failed_block_titles`、未捕获异常先 error + ⚠ content 再抛

## 4. Phase 4 reporting 与收敛

- [ ] 4.1 实现 report OUTLINE 步 + 确定性 coverage 修复（无效 id 过滤、空 section token 重叠配对、未覆盖 block 归最相近 section、Additional findings 附录节），表驱动单测
- [ ] 4.2 实现迭代产出：H1 → INTRO(3000) → 逐节 SECTION(6000，evidence 单 block 4000/总 12000) → CONCLUSION(3000，recap ≤300)；`research_status_key`/`report_part`/`section_*` metadata 保持基线取值；步骤间空行分隔 content
- [ ] 4.3 实现引用链接化与 References：`reference_list` content 事件、`response = body + "\n\n" + references`
- [ ] 4.4 最终 result 经 `EmitCapabilityResult`：`metadata.mode="agentic_research"` + topic/block_count/citation_count/partial 系列 + cost_summary 全 pipeline 汇总

## 5. 验收（spec Scenario 落测试 + 协议对等）

- [ ] 5.1 spec 全部 12 个 Scenario 落为 Go 测试（replay provider）：outline 预览、确认恢复、澄清 mini loop、解析兜底、并行批次、APPEND 拒绝、citation 入库、partial 报告、coverage 修复、References、最终 result、kb_system_note
- [ ] 5.2 stage 事件 fixtures 对等验收（acceptance.md M3 矩阵「deep_research」行）：录制基线「首调 outline 预览」与「确认后全程」两条统一 WS fixtures，Go mock LLM 回放逐事件 diff（首调无 researching/reporting 事件、`outline_preview` result 无 cost_summary）
- [ ] 5.3 并行交错白名单登记：series 模式 fixture 全序对齐 + parallel 模式块间交错豁免记录（acceptance.md 3.1）
