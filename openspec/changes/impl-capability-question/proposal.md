# impl-capability-question — Proposal

## Why

deeptutor-go 需要交付 `deep_question` capability（followup / custom / mimic 三路出题）与出题相关的三个 legacy WebSocket 面（generate / mimic / judge）的 Go 实现。行为事实源为 `docs/golang-req/openspec/specs/capability-question/spec.md`；按 `docs/golang-req/openspec/ROADMAP.md` 定位属 Wave 3 / 里程碑 M3。该模块承载前端 QuizViewer 与出题页面，manifest stage 声明（`ideation`/`generation`）与实际流上 stage 事件（`exploring`/`planning`/`quizzing`）名实分离已写入 spec，Go 版必须保持同样结构以兼容前端。

## What Changes

- 新增 `internal/capabilities/question`：`deep_question` capability——三路分发（followup 单次问答 / custom 三阶段 QuestionPipeline / mimic 跳过前两阶段）；explore（THINK/TOOL/FINISH loop + Tool Summarizer）、plan（PLAN labeled step 容错解析）、quiz（逐题 loop + 归一校验 + 一次修复）阶段；结果 envelope 经 `EmitCapabilityResult`。
- 新增 mimic 输入解析：PDF 附件经 Sidecar MinerU 解析（进度桥接为 `exploring` thinking 事件）、`paper_path` 已解析目录、`[Attached Documents]` 退化路径。
- 新增三个 legacy WS 端点（gin handler 内 gorilla/websocket upgrade）：`/api/v1/question/generate`（消息序列 `task_id → status → 进度 → batch_summary → complete`，实现按差异 D-008 复用 QuestionPipeline）、`/api/v1/question/mimic`（upload/parsed 双模式）、`/api/v1/question/judge`（started → text* → done，多模态判题）。
- 从基线原样拷贝 prompts：`agents/question/prompts/{en,zh}/{pipeline.yaml, followup_agent.yaml}`。

## Capabilities

### New Capabilities
- `capability-question`: deep_question capability（三路分发 + explore/plan/quiz pipeline）与出题/仿题/判题三个 legacy WS 面，行为对等基线 v1.5.2（generate WS 按已登记差异 D-008 换实现保消息面）。

### Modified Capabilities
（无）

## Impact

- 依赖的其他 change（按 ROADMAP 依赖图）：
  - `impl-capability-chat-solve`（agentic loop 原语 / 工具组合策略 / `EmitCapabilityResult`）
  - `impl-agent-loop`、`impl-turn-runtime`、`impl-runtime-registry`
  - `impl-llm-provider`（labeled step 与判题的 eino ChatModel 调用、vision 能力探测）
  - `impl-sidecar-contract`（MinerU `ParseDocument` 解析试卷 PDF）
  - `impl-api-unified-ws`（capability 经统一 WS 触发；三个 legacy WS 为本 change 自建端点，仅复用其鉴权中间件）
- 受影响面：前端 QuizViewer / 出题页 / 判题交互；`question_gen` task 目录布局；acceptance.md 差异 D-008（generate WS 弃 `AgentCoordinator` 改 QuestionPipeline，消息面以 fixtures 验证不变）。
