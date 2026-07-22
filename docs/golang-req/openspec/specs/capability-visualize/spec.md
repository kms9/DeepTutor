# capability-visualize Specification

## Purpose
本模块定义 Go 版 `visualize` 与 `math_animator` 两个 capability 的目标行为。`visualize` 是统一可视化入口：analyzing 阶段路由到六种 render type 之一——svg / chartjs / mermaid / html 走三阶段文本产出管线（generating → reviewing，本地确定性校验 + 定向修复），manim_video / manim_image 则接管为 Manim 多智能体子管线。`math_animator` 是独立的 Manim 动画 capability，与 visualize 的 Manim 路径共享同一 pipeline。Python 基线用本地 `manim` subprocess 渲染；Go 版渲染 SHALL 经 Sidecar RenderManim 完成（有意差异），对流事件与 envelope 字段保持对等。

- 参考实现（基线）：`deeptutor/agents/visualize/capability.py`、`deeptutor/agents/visualize/pipeline.py`、`deeptutor/agents/visualize/utils.py`、`deeptutor/agents/visualize/prompts/{en,zh}/`、`deeptutor/agents/math_animator/capability.py`、`deeptutor/agents/math_animator/pipeline.py`、`deeptutor/agents/math_animator/renderer.py`、`deeptutor/agents/math_animator/retry_manager.py`、`deeptutor/agents/math_animator/visual_review.py`、`deeptutor/agents/math_animator/prompts/{en,zh}/`
- 依赖 spec / 里程碑：依赖 llm-provider、runtime-registry、sidecar-contract（RenderManim）、api-unified-ws；里程碑：M4

## Requirements

### Requirement: visualize manifest 与 render_type 路由
系统 SHALL 注册名为 `visualize` 的 capability：manifest 声明 `stages=[analyzing, generating, reviewing, concept_analysis, concept_design, code_generation, code_retry, summary, render_output]`（前三个覆盖文本产出路径，其余覆盖 Manim 路径；单次 turn 只流其中一个子集）、`tools_used=[]`、`cli_aliases=["visualize", "viz"]`。`run()` SHALL 校验 request config（`render_mode`、`quality`、`style_hint`），在 `analyzing` 阶段（stage 上下文自动发 `stage_start`/`stage_end`）执行 AnalysisAgent：先发 thinking（"Analyzing…" i18n 文案），得到 `{render_type, description, data_description}` 后发一条 progress（`render_type_detected` 文案）。`render_mode` 指定时 SHALL 约束/覆盖路由结论。`render_type ∈ {manim_video, manim_image}` 时 SHALL 转入 Manim 子管线，否则继续文本路径。会话历史文本（`metadata.conversation_context_text`）SHALL 注入 analysis 与代码生成提示词。

#### Scenario: 路由到 Mermaid
- **WHEN** 用户输入"画一个 TCP 握手流程图"
- **THEN** `analyzing` 阶段产出 `render_type="mermaid"` 并发 progress，随后进入 `generating`/`reviewing`，不出现 Manim 阶段

### Requirement: 文本路径 — generating 阶段
`generating` 阶段 SHALL 先发 thinking（"Generating…"），再执行 CodeGenerationAgent 按 render_type 专属提示词产出代码文本，完成后发 progress（`code_generated`）。代码语言映射：svg→`svg`、mermaid→`mermaid`、html→`html`、chartjs→`javascript`。

#### Scenario: 生成 Chart.js 代码
- **WHEN** analysis 判定 `render_type="chartjs"`
- **THEN** `generating` 阶段产出严格 JSON 的 Chart.js 配置，语言标签为 `javascript`

### Requirement: 文本路径 — reviewing 阶段（本地校验 + 定向修复）
`reviewing` 阶段 SHALL 首先执行零成本的确定性本地校验（`validate_visualization`：svg 为 well-formed XML、chartjs 为 strict JSON、mermaid 为 lint、html 为 sanity check），SHALL NOT 对通过校验的草稿追加开放式 LLM review。分支行为：
- 校验通过 → 直接采用草稿，`review.changed=false`、notes 为 "Passed local validation."，发 progress（`validation_passed`）；
- html 校验失败 → SHALL NOT 走修复调用（文档过大），改用最小可渲染 fallback HTML 模板（注入 title/summary/note），`review.changed=true`，发 progress（`html_invalid_fallback`）；
- 其余类型校验失败 → 发 thinking（`repairing`）后执行一次由具体错误驱动的定向修复 LLM 调用；修复调用自身异常时 SHALL 回退未校验草稿并发 progress（`repair_skipped_error`）；修复产物 SHALL 复检——通过发 `code_repaired`，仍失败发 `repair_incomplete`（携带残余错误）但继续交付。

#### Scenario: SVG 修复一次后通过
- **WHEN** 生成的 SVG XML 不闭合
- **THEN** 执行一次定向修复调用，复检通过后发 `code_repaired` progress，最终交付修复后代码

### Requirement: 文本路径结果 envelope（收敛点）
`reviewing` 阶段末尾 SHALL 发一条 content 事件：`` ```<lang_tag>\n<final_code>\n``` `` 围栏代码块（stage=`reviewing`）。随后经 `emit_capability_result()` 发唯一 result：payload 为 `{response: <content_md>, render_type, code: {language, content}, analysis: <AnalysisResult 全量>, review: {optimized_code, changed, review_notes}}`，`metadata.cost_summary` 由 UsageTracker 合并；`render_type` 为前端 viewer 分发的判别字段。

#### Scenario: 前端按 render_type 分发
- **WHEN** result 事件 `render_type="svg"`
- **THEN** 前端从 `code.content` 取原始 SVG 交给 SVG viewer 渲染，聊天区展示围栏代码块

### Requirement: math_animator manifest 与阶段编排
系统 SHALL 注册名为 `math_animator` 的 capability：manifest 声明 `stages=[concept_analysis, concept_design, code_generation, code_retry, summary, render_output]`、`tools_used=[]`、`cli_aliases=["animate"]`、`config_defaults={output_mode: "video", quality: "medium", style_hint: ""}`。（有意差异）Python 基线在入口检查本地 `manim` 依赖缺失即抛错；Go 版 SHALL 改为检查 Sidecar RenderManim 可用性，不可用时以等价错误信息终止。六个阶段 SHALL 顺序执行且每阶段記录耗时（秒，3 位小数）进 `timings`：
1. `concept_analysis`：分析概念（注入 history_context 与附件）；
2. `concept_design`：产出场景设计；
3. `code_generation`：产出 Manim Python 代码，完成发 progress（`code_prepared`）；
4. `code_retry`：渲染 + 重试循环（见下），入口发一条含 `call_state: "running"` 的 progress（"Rendering <mode> with quality=<q>"）；
5. `summary`：产出总结文本并以 content 事件流出（stage=`summary`）；
6. `render_output`：发一条 `call_state: "complete"` 的 progress 报告 artifact 数量（单/复数 i18n 键），`timings.render_output=0.0`。

#### Scenario: 一次完整视频渲染
- **WHEN** 用户运行 `deeptutor run math_animator "泰勒展开动画"`
- **THEN** 六对 stage 事件按序出现，`summary` 阶段流出总结文本，result 携带 artifacts 与逐阶段 timings

### Requirement: 渲染执行与产物（Sidecar RenderManim）
渲染 SHALL 按 `output_mode` 分派：`video` 渲染完整场景为单个视频 artifact；`image` 将代码切分为分镜 block 逐块渲染静帧（storyboard），产出多个图片 artifact。质量映射 SHALL 与基线一致：`low→-ql`、`medium→-qm`、`high→-qh`，未知回退 `-qm`；scene 类名从代码 `class X(...Scene...)` 模式提取。（有意差异）Python 基线以 `python -m manim` subprocess 执行并逐行读取 stdout/stderr；Go 版 SHALL 调用 Sidecar 的 RenderManim 接口，sidecar 回传的渲染进度行 SHALL 经 `on_render_progress` 桥接为 `render_output` 阶段的 progress 事件（metadata `trace_layer` 区分 `raw`（原始日志行）/`summary`（阶段性摘要）），产物路径/类型语义与基线 `RenderedArtifact` 一致。渲染失败 SHALL 以含 stderr 摘要的错误进入重试流程。

#### Scenario: image 模式分镜渲染
- **WHEN** `output_mode="image"` 且代码含 3 个分镜 block
- **THEN** 逐块渲染出 3 个图片 artifact，`render_output` 阶段可见各块的 raw 进度行

### Requirement: code_retry 重试预算与修复回路
渲染重试 SHALL 由 CodeRetryManager 等价逻辑管理：`max_retries` 默认 4（即最多 5 次渲染尝试），单次代码修复调用超时 180s；识别为不可重试的环境类错误（如依赖缺失）SHALL 立即终止不消耗重试。每次失败 SHALL：构造 `RetryAttempt{attempt, error}` 记入 `retry_history` → 经 `on_retry` 发 `code_retry` 阶段 progress（"Retry N: <error>"，metadata `trace_layer: "raw"`）→ 经 `on_status` 发修复状态 progress（`trace_layer: "summary"`）→ 以（当前代码, 错误文本, 尝试序号）调用修复 agent 生成新代码后重渲。修复调用超时 SHALL 视为该次尝试失败。预算耗尽仍失败 SHALL 抛错终止 turn。最终 `render.retry_attempts`/`retry_history` SHALL 进入 result envelope。

#### Scenario: 两次失败后成功
- **WHEN** 首次与第二次渲染均因代码错误失败、第三次成功
- **THEN** 流上出现 2 条 Retry progress，result 的 `render.retry_attempts=2`、`retry_history` 含两条 `{attempt, error}`

### Requirement: 可选 visual review（默认关闭）
pipeline SHALL 支持 `enable_visual_review` 开关（默认 false）。开启时每次渲染成功后 SHALL 抽取产物帧构建图片附件并执行一次视觉质量 review LLM 调用（有超时约束）：无可用帧或当前模型不支持视觉时 SHALL 直接判定 `passed=true` 并注明 skip 原因；`passed=false` 且预算未耗尽时按视觉反馈进入修复重渲。review 结果 SHALL 挂到 `render_result.visual_review` 并透传进 envelope 的 `render.visual_review`（关闭或未执行时为 null）。

#### Scenario: 模型不支持视觉时跳过
- **WHEN** 开启 visual review 但当前模型无 vision 能力
- **THEN** review 返回 `passed=true` 且 summary 注明跳过原因，不触发额外重渲

### Requirement: Manim 路径结果 envelope（收敛点）
`math_animator` SHALL 经 `emit_capability_result()` 发唯一 result：payload 为 `{response: <summary_text>, summary, code: {language: "python", content: <final_code>}, output_mode, artifacts: [...], timings, render: {quality, retry_attempts, retry_history, source_code_path, visual_review}, analysis, design}`。`visualize` 的 Manim 路径 SHALL 发出结构完全相同的 payload 且额外携带顶层 `render_type`（`manim_video` 或 `manim_image`）作为前端分发判别字段，其 `output_mode` 由 render_type 推导（`manim_image→image`，否则 `video`），`quality`/`style_hint` 取自 visualize request config。两条路径 SHALL 共享同一 pipeline 实现。

#### Scenario: visualize 路由 Manim 后的 envelope
- **WHEN** visualize 的 analysis 判定 `render_type="manim_video"`
- **THEN** 后续阶段与 math_animator 完全一致，result 额外含 `render_type: "manim_video"`，前端据此路由到 MathAnimatorViewer

### Requirement: trace bridge — 子 agent LLM 调用映射为流事件
两个 capability SHALL 为 pipeline 注入 trace callback，把子 agent 的 `llm_call` 更新映射为流事件（source 为 capability 名，stage 取 update 的 phase/stage）：`state=running` → progress（metadata `trace_kind: "call_status"`、`call_state: "running"`，message 为 label 或阶段名 Title Case）；`state=streaming` 的非空 chunk → thinking（`trace_kind: "llm_chunk"`）；`state=complete` → 未流式时先以完整 response 发 thinking（`trace_kind: "llm_output"`），再发 complete 状态 progress；`state=error` → error 事件（`call_state: "error"`，空响应时用 i18n 兜底文案）。非 `llm_call` 更新 SHALL 忽略。

#### Scenario: 设计阶段流式思考可见
- **WHEN** `concept_design` 的 agent 以流式返回
- **THEN** 每个 chunk 作为 `trace_kind="llm_chunk"` 的 thinking 事件实时出现在 `concept_design` 阶段卡片中

### Requirement: prompts 资源与状态文案 i18n
两个 capability 的 agent 提示词 SHALL 来自各自目录 `prompts/{en,zh}/`（visualize：analysis / 各 render type 代码生成 / repair；math_animator：concept_analysis_agent、concept_design_agent、code_generator_agent、summary_agent、visual_review_agent），Go 版 SHALL 原样拷贝并实现相同 `{var}` 注入（如 visual review 注入 `user_input`/`output_mode`/`reviewed_frames`/`render_json`/`current_code`）。状态文案 SHALL 经 StatusI18n 等价机制按 `context.language` 取 `visualize`/`math_animator` 模块键（如 `render_type_detected`、`manim_retry`、`artifacts_one/many`），缺键回退调用点默认英文文案。

#### Scenario: 中文状态文案
- **WHEN** `language="zh"` 下运行 visualize
- **THEN** analyzing/generating/reviewing 的 thinking 与 progress 文案取 zh 资源，代码产物本身不受语言影响
