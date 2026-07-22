# impl-capability-visualize — Design

## Context

`visualize` 在 analyzing 阶段用 AnalysisAgent 判定 `render_type`：svg/chartjs/mermaid/html 走「generating → reviewing」文本产出路径（reviewing 是零成本本地确定性校验 + 失败时定向修复，**不做**开放式 LLM review）；manim_video/manim_image 转入与 `math_animator` 共享的 Manim 六阶段 pipeline。事实源：`docs/golang-req/openspec/specs/capability-visualize/spec.md`。有意差异（已写入 spec）：Python 基线本地 `python -m manim` subprocess 渲染，Go 版渲染经 Sidecar RenderManim，流事件与 envelope 字段保持对等；入口依赖检查改为 Sidecar 探活。两 capability 均 `tools_used=[]`——无工具 loop，是纯 pipeline 型 capability。

## Goals / Non-Goals

**Goals:**
- visualize 文本路径三阶段 + Manim 路径路由行为对等；`validate_visualization` 四类校验器确定性可单测。
- math_animator 六阶段、CodeRetryManager（max_retries=4）、visual review 开关、trace bridge 对等。
- RenderManim 调用与进度桥接（`trace_layer` raw/summary）落地；两路径共享同一 Manim pipeline 实现。

**Non-Goals:**
- 不实现 Manim 渲染器本体与其 Python 环境（impl-sidecar-contract）；不实现前端 viewer；不迁移基线本地 subprocess 渲染代码。

## Decisions

### D1. Go 包结构

```
internal/capabilities/visualize/
├── capability.go   # VisualizeCapability：manifest + Run()（analyzing 路由 + render_mode 覆盖）
├── analysis.go     # AnalysisAgent：{render_type, description, data_description}
├── generate.go     # CodeGenerationAgent：按 render_type 专属提示词 + 语言映射
├── review.go       # validate_visualization 四校验器 + html fallback + 定向修复 + 复检
├── envelope.go     # 文本路径 result（围栏 content + payload）
└── prompts/        # {en,zh}/（analysis / 各 render type / repair，基线原样拷贝）
internal/capabilities/mathanimator/
├── capability.go   # MathAnimatorCapability：manifest + Run()（Sidecar 探活、六阶段编排、timings）
├── pipeline.go     # 六阶段函数（concept_analysis/concept_design/code_generation/code_retry/summary/render_output）
├── renderer.go     # Sidecar RenderManim 适配：质量映射、scene 类名提取、分镜切分、进度桥接
├── retry.go        # CodeRetryManager：RetryAttempt/retry_history/不可重试判定/修复调用（180s 超时）
├── review.go       # 可选 visual review（帧抽取 + vision 判定 + skip 语义）
├── tracebridge.go  # llm_call 更新 → progress/thinking/error 流事件映射
└── prompts/        # {en,zh}/（五个 agent 提示词，基线原样拷贝）
```

visualize 的 Manim 路径直接构造 `mathanimator.Pipeline` 运行（共享实现，envelope 额外补 `render_type` 顶层字段）。

### D2. Capability interface 实现签名

```go
type VisualizeCapability struct { manim *mathanimator.Pipeline; /* llm deps */ }
func (c *VisualizeCapability) Manifest() core.CapabilityManifest
// stages=[analyzing, generating, reviewing, concept_analysis, concept_design,
//         code_generation, code_retry, summary, render_output]（单 turn 只流一个子集，名实分离）
// tools_used=[], cli_aliases=["visualize","viz"]
func (c *VisualizeCapability) Run(ctx context.Context, uc *core.UnifiedContext, bus *core.StreamBus) error

type MathAnimatorCapability struct { pipe *mathanimator.Pipeline; sidecar sidecar.Client }
// stages=[concept_analysis, concept_design, code_generation, code_retry, summary, render_output],
// cli_aliases=["animate"], config_defaults={output_mode:"video", quality:"medium", style_hint:""}

// renderer.go
type Renderer struct{ sidecar sidecar.Client }
func (r *Renderer) Render(ctx context.Context, req RenderRequest,
    onProgress func(line string, layer string)) (*RenderedArtifact, error)
// req: {Code, SceneName, QualityFlag(-ql/-qm/-qh), OutputMode} → sidecar.RenderManim
```

### D3. 阶段编排落位（eino Graph vs 自定义）——取舍结论

**结论：自定义阶段函数，不用 eino Graph；这是六个 capability change 中最接近 Graph 适用场景的一个，仍判定不用。** 论证：math_animator 的六阶段确实是静态线性拓扑，Graph 可表达；但 (1) `code_retry` 内部是「渲染 → 失败 → 修复 → 重渲」的**外部 IO 驱动循环**（Sidecar 调用 + 180s 超时 + 不可重试短路 + 可选 visual review 反馈重渲），Graph 的分支/循环表达远不如显式 `for attempt := 0; attempt <= maxRetries; attempt++` 可读且可精确对齐基线重试语义；(2) 逐阶段 timings（3 位小数）与 stage 事件包裹要求编排层对每阶段有统一的进入/退出钩子，手写 `runStage(name, fn)` helper 十行解决；(3) 保持与 chat/question/research 相同结论（各自 design D3），全项目"eino 管 LLM 调用、手写 Go 管编排"心智统一。各子 agent（analysis/design/codegen/summary/review）为独立的一次性 eino `ChatModel.Stream` 调用，经 trace bridge 映射流事件。

### D4. stage 事件发射点

- visualize 文本路径：`analyzing`（thinking "Analyzing…" → progress `render_type_detected`）→ `generating`（thinking "Generating…" → progress `code_generated`）→ `reviewing`（校验分支 progress：`validation_passed` / `html_invalid_fallback` / thinking `repairing` + `code_repaired` / `repair_skipped_error` / `repair_incomplete`；末尾围栏代码块 content）。
- Manim 路径（两 capability 共享）：六对 stage 事件按序；`code_generation` 完成发 `code_prepared` progress；`code_retry` 入口发 `call_state:"running"` progress（"Rendering <mode> with quality=<q>"）、每次失败发 Retry progress（`trace_layer:"raw"`）+ 修复状态 progress（`trace_layer:"summary"`）；`summary` 阶段总结文本以 content 流出；`render_output` 发 `call_state:"complete"` progress（artifact 单/复数 i18n）且 `timings.render_output=0.0`；Sidecar 进度行经 `on_render_progress` 桥接为 `render_output` 阶段 progress。
- trace bridge：`state=running`→progress（`trace_kind:"call_status"`）、`streaming` 非空 chunk→thinking（`llm_chunk`）、`complete` 未流式→thinking（`llm_output`）+ complete progress、`error`→error 事件；非 `llm_call` 更新忽略。
- result：两路径均经 `shared.EmitCapabilityResult`。文本路径 payload `{response, render_type, code{language, content}, analysis, review{optimized_code, changed, review_notes}}`；Manim 路径 `{response, summary, code{language:"python", content}, output_mode, artifacts, timings, render{quality, retry_attempts, retry_history, source_code_path, visual_review}, analysis, design}`，visualize 路由时额外顶层 `render_type`。

### D5. Sidecar RenderManim 调用与 max_retries=4 重试语义

- **调用**：`sidecar.RenderManim(ctx, {code, scene, quality_flag, output_mode})`，流式响应（进度行 + 终态 artifact 路径/stderr）。质量映射 `low→-ql / medium→-qm / high→-qh`（未知回退 `-qm`）；scene 类名以 `class X(...Scene...)` 正则提取；image 模式在 Go 侧把代码切分为分镜 block 后逐块调用（每块一个图片 artifact）。渲染失败返回含 stderr 摘要的错误进入重试。
- **重试语义（CodeRetryManager）**：`max_retries` 默认 **4**（即至多 5 次渲染尝试）；每次失败：`RetryAttempt{attempt, error}` 入 `retry_history` → `on_retry` 发 "Retry N: <error>" progress（raw）→ `on_status` 发修复状态 progress（summary）→ 以（当前代码, 错误文本, 尝试序号）调修复 agent（单次 **180s** 超时，超时视为该次尝试失败）→ 重渲。识别为不可重试的环境类错误（依赖缺失等模式匹配基线清单）立即终止**不消耗重试**。预算耗尽仍失败抛错终止 turn。`retry_attempts`/`retry_history` 进 envelope 的 `render`。
- **探活**：capability 入口检查 Sidecar RenderManim 可用性（health/capability 探测），不可用时以与基线依赖缺失等价的错误信息终止（有意差异，spec 已登记）。

### D6. 本地校验器与 prompts

`review.go` 四校验器为纯函数：svg 用 `encoding/xml` well-formed 解析；chartjs 用 `encoding/json` strict 解析；mermaid 移植基线 lint 规则（行级语法检查）；html sanity check（标签配平等基线同款启发式）。html 失败不修复——渲染最小 fallback 模板（注入 title/summary/note）。prompts 经 `go:embed` + shared.PromptStore（`{var}` 注入，如 visual review 的 `user_input`/`output_mode`/`reviewed_frames`/`render_json`/`current_code`）；状态文案经 StatusI18n 等价机制按 `context.language` 取 `visualize`/`math_animator` 模块键，缺键回退英文默认。

### D7. 与 Python 基线文件映射表

| Python 基线 | Go 落位 |
| --- | --- |
| `deeptutor/agents/visualize/capability.py` | `internal/capabilities/visualize/capability.go` |
| `deeptutor/agents/visualize/pipeline.py` | `internal/capabilities/visualize/{analysis,generate,review,envelope}.go` |
| `deeptutor/agents/visualize/utils.py`（validate_visualization） | `internal/capabilities/visualize/review.go` |
| `deeptutor/agents/visualize/prompts/{en,zh}/` | `internal/capabilities/visualize/prompts/{en,zh}/`（原样拷贝） |
| `deeptutor/agents/math_animator/capability.py` | `internal/capabilities/mathanimator/capability.go` |
| `deeptutor/agents/math_animator/pipeline.py` | `internal/capabilities/mathanimator/pipeline.go` |
| `deeptutor/agents/math_animator/renderer.py`（本地 subprocess） | `internal/capabilities/mathanimator/renderer.go`（Sidecar RenderManim，有意差异）+ sidecar 侧复用基线渲染代码 |
| `deeptutor/agents/math_animator/retry_manager.py` | `internal/capabilities/mathanimator/retry.go` |
| `deeptutor/agents/math_animator/visual_review.py` | `internal/capabilities/mathanimator/review.go` |
| `deeptutor/agents/math_animator/prompts/{en,zh}/` | `internal/capabilities/mathanimator/prompts/{en,zh}/`（原样拷贝） |

## Risks / Trade-offs

- [Sidecar 渲染进度行的粒度/内容与基线 stdout 逐行读取不完全一致] → 桥接层保持 `trace_layer` raw/summary 语义与事件类型不变；fixtures 验收锚定事件结构而非日志逐行文本（豁免按 acceptance.md 3.1 登记）。
- [分镜切分规则（image 模式）在 Go 重实现可能与基线偏差] → 从基线提取切分用例做表驱动 golden 测试。
- [180s 修复超时 + 5 次尝试的最坏 turn 时长 >15 分钟] → 与基线同构不改预算；`code_retry` 阶段 progress 持续喂前端 idle 看门狗（trace bridge 保证不静默）。
- [mermaid lint 无权威 Go 库] → 只移植基线自带 lint 规则（行级启发式），不引入第三方 mermaid 解析器，避免校验严格度漂移。
- [visual review 依赖 vision 能力探测] → 无帧或无 vision 一律 `passed=true` + skip 原因（spec 规定 fail-open），不阻塞主链路。

## Migration Plan

实现顺序：本地校验器 + 文本路径（无 Sidecar 依赖，可先行）→ Manim pipeline + retry（mock sidecar）→ 真实 Sidecar e2e（M3 矩阵 math_animator 行：渲染出真实视频文件）→ visualize 路由整合。回滚 = 不注册两 capability。

## Open Questions

- Sidecar RenderManim 的流式进度契约（行粒度、终态字段）以 impl-sidecar-contract 冻结版为准，若与本 design 假设不符需回改 renderer.go 适配层。
