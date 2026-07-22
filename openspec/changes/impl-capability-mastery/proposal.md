# impl-capability-mastery — Proposal

## Why

deeptutor-go 需要交付 `mastery_path` capability（Guided Learning）与其底层 learning 引擎的 Go 实现。行为事实源为 `docs/golang-req/openspec/specs/capability-mastery/spec.md`；按 `docs/golang-req/openspec/ROADMAP.md` 定位属 Wave 3 / 里程碑 M3。核心公理：智能在 loop 出口（模型决定教什么），而学习者能否前进的门（gate）由确定性引擎计算——grading / mastery / scheduler 三个算法必须对相同输入产出与 Python 完全一致（acceptance.md M3 矩阵「learning 引擎：golden case 对比」），其中 grading 的 short 题模糊匹配须在 Go 复刻 Python `difflib.SequenceMatcher.ratio()`（Ratcliff-Obershelp）。

## What Changes

- 新增 `internal/capabilities/mastery`：`mastery_path` capability——复用 chat agent loop，追加五个 `mastery_*` 工具（status / quiz / grade / assess / build）与 tutor system prompt；`mastery_path_id` 解析链（显式 metadata → book_references 首项 → session_id 回退，白名单清洗）。
- 新增 `internal/learning`：确定性引擎——`grading`（含 Ratcliff-Obershelp 移植）、`mastery`（近因加权 + 置信上限）、`scheduler`（间隔重复 + review queue）、`policy`（纯函数门控与 `next_objective`）、`service`（grade_and_record 全流程）、`storage`（每 path 一个 JSON、原子写、路径守卫）、`models`。
- 新增 `/api/v1/learning` REST 面（gin route group）：progress 列表/详情/map、init-modules、import-from-book、generate-from-notebook、delete、redo。
- 从基线原样拷贝 prompts：`capabilities/mastery/prompts/{en,zh}/system.md`、`learning/prompts/{en,zh}.yaml`。

## Capabilities

### New Capabilities
- `capability-mastery`: mastery_path capability（五工具 + tutor prompt + chat loop 复用）、learning 确定性引擎（grading/mastery/scheduler/policy 逐位一致）、learning 存储与 `/api/v1/learning` REST 面。

### Modified Capabilities
（无）

## Impact

- 依赖的其他 change（按 ROADMAP 依赖图）：
  - `impl-capability-chat-solve`（chat loop 与 `LoopCapability` 挂钩点、`EmitCapabilityResult` 收敛点）
  - `impl-agent-loop`、`impl-turn-runtime`（loop 原语、UnifiedContext、活跃 turn cancel）、`impl-runtime-registry`（capability/tool 注册）
  - `impl-tools-builtin`（挂载面中的 `rag`/`read_source`/`ask_user`）
  - `impl-session-store`（question bank 同步到 notebook entries）
  - 间接：`impl-llm-provider`（generate-from-notebook 的 LLM 调用）、`impl-foundation-config`。
- 受影响面：前端 Learning Space 页面（dashboard 经 `/api/v1/learning`，与 tutor 的 `mastery_status` 共用同一策略层）；统一 WS 的 mastery turn 事件序列（M3 fixtures）。
- golden case 对比测试要求维护一份 Python/Go 共享输入输出用例集（grading/mastery/scheduler）。
