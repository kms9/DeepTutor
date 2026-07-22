# impl-capability-visualize — Tasks

## 1. 骨架、prompts 与本地校验器（无 Sidecar 依赖，可先行）

- [ ] 1.1 建立 `internal/capabilities/{visualize,mathanimator}` 包骨架与模型（AnalysisResult / RenderedArtifact / RetryAttempt / envelope 结构）
- [ ] 1.2 从基线原样拷贝两组 prompts（`agents/visualize/prompts/{en,zh}/`、`agents/math_animator/prompts/{en,zh}/`），`go:embed` + PromptStore 接入（visual review 五变量注入等）；StatusI18n 等价状态文案（`render_type_detected`、`manim_retry`、`artifacts_one/many`，缺键回退英文）
- [ ] 1.3 实现 `validate_visualization` 四校验器（svg XML / chartjs strict JSON / mermaid lint 移植 / html sanity）+ html fallback 模板，表驱动单测（含基线提取的通过/失败样例）

## 2. visualize 文本路径

- [ ] 2.1 实现 analyzing：request config 校验（render_mode/quality/style_hint）、AnalysisAgent（thinking "Analyzing…" → `{render_type, description, data_description}` → `render_type_detected` progress）、`render_mode` 约束覆盖、`conversation_context_text` 注入
- [ ] 2.2 实现 generating：CodeGenerationAgent（render type 专属提示词）、语言映射（svg/mermaid/html/javascript）、`code_generated` progress
- [ ] 2.3 实现 reviewing 分支：通过→直接采用（`changed=false`、`validation_passed`）；html 失败→fallback 模板（`html_invalid_fallback`）；其余失败→`repairing` thinking + 一次定向修复 + 复检（`code_repaired`/`repair_incomplete`）+ 修复自身异常回退（`repair_skipped_error`）
- [ ] 2.4 实现文本路径 envelope：围栏代码块 content（stage=reviewing）+ `EmitCapabilityResult`（`{response, render_type, code, analysis, review}` + cost_summary）

## 3. Manim pipeline（math_animator + visualize 共享）

- [ ] 3.1 实现 `MathAnimatorCapability` manifest（六 stages、`cli_aliases=["animate"]`、config_defaults）与入口 Sidecar RenderManim 探活（不可用等价报错——有意差异）
- [ ] 3.2 实现六阶段编排：`runStage` helper（stage 事件包裹 + 逐阶段 timings 3 位小数）、concept_analysis（history/附件注入）→ concept_design → code_generation（`code_prepared`）→ code_retry → summary（content 流出）→ render_output（`call_state:"complete"` progress、`timings.render_output=0.0`）
- [ ] 3.3 实现 `renderer.go`：Sidecar RenderManim 调用、质量映射（-ql/-qm/-qh、未知回退 -qm）、scene 类名正则提取、image 模式分镜切分逐块渲染（golden 测试）、进度行 `on_render_progress` 桥接（`trace_layer` raw/summary）、失败含 stderr 摘要
- [ ] 3.4 实现 `retry.go` CodeRetryManager：`max_retries=4`（≤5 次尝试）、修复调用 180s 超时（超时算失败）、不可重试环境错误立断不消耗预算、RetryAttempt/retry_history、`on_retry`（"Retry N: <error>" raw）+ `on_status`（summary）progress、预算耗尽抛错
- [ ] 3.5 实现可选 visual review（默认关）：帧抽取 → vision review 调用（超时约束）、无帧/无 vision 时 `passed=true` + skip 原因、fail 未耗尽预算按反馈修复重渲、结果挂 `render.visual_review`
- [ ] 3.6 实现 trace bridge：`llm_call` 的 running/streaming/complete/error 四态映射（progress/thinking/error），非 `llm_call` 忽略
- [ ] 3.7 实现 Manim envelope：`EmitCapabilityResult` payload（response/summary/code/output_mode/artifacts/timings/render/analysis/design）；visualize 路由额外顶层 `render_type` 且 `output_mode` 由 render_type 推导、quality/style_hint 取 visualize config

## 4. visualize 路由整合

- [ ] 4.1 `render_type ∈ {manim_video, manim_image}` 时转入共享 Manim pipeline；确认单 turn 只流一个 stage 子集（文本三阶段或 Manim 六阶段）

## 5. 验收（spec Scenario 落测试 + 协议对等）

- [ ] 5.1 spec 全部 11 个 Scenario 落为 Go 测试（replay provider + mock sidecar）：mermaid 路由、chartjs 生成、SVG 修复、render_type 分发、完整视频渲染、image 分镜、两次失败后成功、vision 跳过、visualize→Manim envelope、design 流式 thinking、zh 状态文案
- [ ] 5.2 stage 事件 fixtures 对等验收（acceptance.md M3 矩阵「visualize」行）：录制基线文本路径（svg 含修复）与 Manim 路径（含 retry）统一 WS fixtures，Go 回放逐事件 diff
- [ ] 5.3 math_animator e2e（acceptance.md M3 矩阵「math_animator」行）：真实 Sidecar 渲染产出视频文件；Sidecar 不可用时的降级错误验证（acceptance.md 3.4「Sidecar 降级」行）
