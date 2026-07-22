# impl-capability-visualize — Proposal

## Why

deeptutor-go 需要交付 `visualize` 与 `math_animator` 两个 capability 的 Go 实现：visualize 是统一可视化入口（analyzing 路由六种 render type——svg/chartjs/mermaid/html 走三阶段文本产出，manim_video/manim_image 接管为 Manim 子管线），math_animator 是独立 Manim 动画 capability，两者共享同一 Manim pipeline。行为事实源为 `docs/golang-req/openspec/specs/capability-visualize/spec.md`；按 `docs/golang-req/openspec/ROADMAP.md` 定位属 Wave 3（capability-visualize），spec 标注里程碑 M4，acceptance M3 矩阵已含 visualize/math_animator 验收行（编排在 Go、渲染经 Sidecar）。关键契约：reviewing 阶段是**本地确定性校验 + 定向修复**而非开放式 LLM review；Manim 渲染经 Sidecar RenderManim（有意差异，对流事件与 envelope 保持对等）。

## What Changes

- 新增 `internal/capabilities/visualize`：`visualize` capability——analyzing（AnalysisAgent 路由 + `render_mode` 覆盖）、generating（按 render type 生成代码）、reviewing（`validate_visualization` 本地校验：svg XML / chartjs strict JSON / mermaid lint / html sanity；失败分支：html 走 fallback 模板、其余一次定向修复 + 复检）；文本路径 result envelope（`render_type` 判别字段）。
- 新增 `internal/capabilities/mathanimator`：`math_animator` capability——六阶段（concept_analysis → concept_design → code_generation → code_retry → summary → render_output）、逐阶段 timings、CodeRetryManager（`max_retries=4`、修复超时 180s、不可重试环境错误立断）、可选 visual review（默认关闭）、trace bridge（子 agent `llm_call` 更新映射为流事件）。
- 渲染经 `impl-sidecar-contract` 的 RenderManim：video 单场景 / image 分镜 storyboard、质量映射 `-ql/-qm/-qh`、进度行桥接为 `render_output` progress（`trace_layer` raw/summary）；入口可用性检查由本地 manim 依赖改为 Sidecar RenderManim 探活（有意差异，已写入 spec）。
- 从基线原样拷贝 prompts：`agents/visualize/prompts/{en,zh}/`、`agents/math_animator/prompts/{en,zh}/`；状态文案经 StatusI18n 等价机制。

## Capabilities

### New Capabilities
- `capability-visualize`: visualize（render_type 路由 + 文本三阶段 + 本地校验修复）与 math_animator（Manim 六阶段 + Sidecar 渲染 + 重试回路），共享 Manim pipeline，行为对等基线 v1.5.2。

### Modified Capabilities
（无）

## Impact

- 依赖的其他 change（按 ROADMAP 依赖图）：
  - `impl-llm-provider`（各子 agent 的 eino ChatModel 调用、vision 能力探测）
  - `impl-runtime-registry`（capability 注册、StreamBus 生命周期）
  - `impl-agent-loop`、`impl-turn-runtime`（capability 触发与 UnifiedContext；本模块无工具 loop，依赖较浅）
  - `impl-sidecar-contract`（RenderManim 接口与进度事件流——硬依赖）
  - `impl-capability-chat-solve`（`EmitCapabilityResult` 收敛点、PromptStore）
  - `impl-api-unified-ws`（capability 经统一 WS 触发）
- 受影响面：前端 SVG/Chart.js/Mermaid/HTML viewer 与 MathAnimatorViewer（按 `render_type` 分发）；`deeptutor run math_animator` CLI 路径；M3 矩阵 math_animator e2e（渲染出真实视频文件）。
