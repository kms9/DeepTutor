# impl-capability-question — Tasks

## 1. 骨架与 prompts

- [ ] 1.1 建立 `internal/capabilities/question` 包骨架与 `PipelineRequest`/`PipelineResult`/`QuestionTemplate`/`QAPair` 模型（题型全集 `{choice, concept, fill_in_blank, short_answer, written, coding}`）
- [ ] 1.2 从基线原样拷贝 `agents/question/prompts/{en,zh}/{pipeline.yaml, followup_agent.yaml}`，接入 shared.PromptStore（`{var}` 注入、`append_language_directive` 等价、缺键回退）

## 2. QuestionPipeline 三阶段

- [ ] 2.1 实现 explore（Phase 1）：THINK/TOOL/FINISH loop（默认 8 迭代可配）、chat 组合策略工具面、`stream_body_live=true`（FINISH 前言实时流入）、迭代耗尽 force-finish（warning + 至多 2 次修复）
- [ ] 2.2 实现 Tool Summarizer：每个 tool result 单独一次 LLM 压缩调用（max_tokens 800 / temperature 0.2）就地替换缓冲区消息；`exploration_trace` 由迭代消息序列渲染
- [ ] 2.3 实现 plan（Phase 2）：PLAN labeled step（无工具、max_tokens 2000）、容错解析（templates|ideas、topic 大小写去重、类型回退 short_answer→允许集首项→written、difficulty 归一、`q_N` 服务端 id、模板数封顶）
- [ ] 2.4 实现 quiz（Phase 3）：逐题 loop（≤5 迭代）、D7 归一校验器、一次修复调用（FINISH-only、max_tokens 2500）、issues 记录与兜底文案、逐题 `quiz_question_emitted` content 事件
- [ ] 2.5 实现 pipeline 编排：三对 stage 事件（`source="deep_question"`）、`plan_count_mismatch` warning、mimic 跳段（空 analysis 合成 Plan 包络）、未捕获异常先 error + ⚠ content 再抛

## 3. capability 分发与 mimic 输入

- [ ] 3.1 实现 `DeepQuestionCapability`：manifest（名实分离的 `stages=["ideation","generation"]`）、三路分发、参数解析（num_questions/difficulty/types 白名单/per_type_counts 越界丢弃/quiz history）、custom 空 topic error
- [ ] 3.2 实现 followup 路径：FollowupAgent 单次流式调用（`generation` stage 内），result `{response, mode:"followup", question_id}` 经 `EmitCapabilityResult`
- [ ] 3.3 实现 `mimic_source.go`：PDF 附件 → 临时文件 → `sidecar.ParseDocument`（进度桥接 `exploring` thinking）；`paper_path` 直接抽题；`[Attached Documents]` 退化 custom + 模仿前缀；三者皆无 error；模板携带 `source:"mimic"` 与 reference_question/answer；解析错误 error 终止
- [ ] 3.4 实现结果 envelope：`summary{success, source, requested, template_count, completed, failed, templates, results, analysis}` + `mode`；custom response=探索前言 / mimic 回退汇总 markdown / 空时 "No questions generated."

## 4. legacy WS 三端点

- [ ] 4.1 实现 `/api/v1/question/judge`：WS 鉴权、字段校验（question 空/无答案 error）、language 归一、`data:` 剥离、vision 能力探测（多图多模态 / 降级纯文本）、`started → text* → done` 序列
- [ ] 4.2 实现 `/api/v1/question/generate`（差异 D-008）：requirement 校验、`question_gen` task id、QuestionPipeline custom 路径换芯、消息序列 `task_id → status(started) → 进度 → batch_summary → complete` 保持不变、产物落 task batch 目录、异常 error + task 状态更新
- [ ] 4.3 实现 `/api/v1/question/mimic`：upload（base64 解码 → 文件名/大小/后缀校验 → 批次目录 + stage 进度）/ parsed（paper_path 必填）双模式、结构化日志/事件实时转发（不做 stdout 截取）、逐题 ws_callback 推送、`complete`/`error` 终态
- [ ] 4.4 三端点统一：gin handler 内 gorilla upgrade、单写 goroutine、错误关闭语义

## 5. 验收（spec Scenario 落测试 + 协议对等）

- [ ] 5.1 spec 全部 11 个 Scenario 落为 Go 测试（replay provider）：followup 分发、三对 stage、摘要替换、类型回退、choice 修复、部分失败 envelope、PDF 仿题、generate WS 序列、parsed 仿题、多模态判题、语言指令
- [ ] 5.2 stage 事件 fixtures 对等验收（acceptance.md M3 矩阵「deep_question」行）：录制基线 custom / mimic / followup turn 的统一 WS fixtures，Go 回放逐事件 diff（stage_start/stage_end/progress 一致）
- [ ] 5.3 legacy WS fixtures 对等（acceptance.md 3.1：`question/generate`、`question/mimic`、`question/judge` 按前端消费子集对比）；D-008 验收记录（消息面不变证据）归档
