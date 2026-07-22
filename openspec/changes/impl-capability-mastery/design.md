# impl-capability-mastery — Design

## Context

`mastery_path` 是 chat loop 的又一 LoopCapability 消费者：capability 本身不做编排创新，只追加五个 `mastery_*` 工具与 tutor system prompt；真正的行为核心在 `learning` 引擎——grading / mastery / scheduler / policy 四个**确定性**组件按 topic type 门控学习者能否前进（MEMORY/PROCEDURE 定量门 ≥0.9，CONCEPT/DESIGN 定性门）。事实源：`docs/golang-req/openspec/specs/capability-mastery/spec.md`。硬约束：三个算法对相同输入与 Python 逐位一致（acceptance.md M3「golden case 对比」）；存储每 path 一个 JSON 文件；`/api/v1/learning` REST 与 dashboard 共用策略层。

## Goals / Non-Goals

**Goals:**
- `mastery_path` capability 与五工具行为对等基线；result envelope 经 `EmitCapabilityResult` 与 chat 相同。
- learning 引擎（grading/mastery/scheduler/policy）Go 移植，golden case 与 Python 输出完全一致；其中 `difflib.SequenceMatcher.ratio()` 以 Ratcliff-Obershelp 原算法复刻。
- `/api/v1/learning` 8 个端点 REST 对等；LearningStore 原子写 + 路径守卫。

**Non-Goals:**
- 不实现 chat loop 与 mounting（消费 impl-capability-chat-solve / impl-agent-loop）；不实现 `ask_user`/`rag` 工具本体；不做 progress 数据 schema 迁移（JSON 布局与基线兼容）。

## Decisions

### D1. Go 包结构

```
internal/capabilities/mastery/
├── capability.go   # MasteryPathCapability：manifest + Run()（path id 解析、mastery_mode、委托 chat loop）
├── loopcap.go      # MasteryLoopCapability：tutor prompt 注入 + 五工具追加 + id kwargs 注入
├── tools.go        # mastery_status / mastery_quiz / mastery_grade / mastery_assess / mastery_build
├── choices.go      # choice 选项正文校验 / 标签解析 / legacy ask_user 恢复
└── prompts/        # {en,zh}/system.md（基线原样拷贝）
internal/learning/
├── grading.go      # grade_answer / classify_error / ratcliff.go（SequenceMatcher.ratio 移植）
├── mastery.go      # compute_mastery（近因权重 + 置信上限）
├── scheduler.go    # 间隔序列 / schedule_next / review queue 重建
├── policy.go       # is_mastered / objective_status / next_objective / map_summary（纯函数）
├── service.go      # grade_and_record 全流程 / replace_modules / redo
├── storage.go      # LearningStore：JSON per path、进程锁、原子写、路径守卫
├── models.go       # Progress / PendingQuestion / RepetitionState / NextStep ...
└── prompts/        # {en,zh}.yaml（generate-from-notebook 提示词，基线原样拷贝）
internal/api/routers/learning.go  # /api/v1/learning gin route group
```

### D2. Capability interface 实现签名

```go
type MasteryPathCapability struct { chat *chat.ChatCapability }
func (c *MasteryPathCapability) Manifest() core.CapabilityManifest
// stages=["responding"], tools_used=[mastery_status, mastery_quiz, mastery_grade,
//   mastery_assess, mastery_build, rag, read_source, ask_user], cli_aliases=["mastery"]
func (c *MasteryPathCapability) Run(ctx context.Context, uc *core.UnifiedContext, bus *core.StreamBus) error
// Run: uc.Metadata["mastery_mode"]=true；ResolvePathID(uc)（显式 → book_references 首项(string|{book_id|id}) → session_id，
//      [A-Za-z0-9_-] 白名单清洗）；委托 chat pipeline

// loopcap.go 实现 chat.LoopCapability：
func (m *MasteryLoopCapability) InjectKwargs(tool string, uc *core.UnifiedContext) map[string]any
// 对五个 mastery_* 注入 _mastery_path_id / _session_id / _turn_id（模型永不提供）
```

每次工具调用新建 `LearningStore` + `LearningService` 实例（与 REST 路由一致，规避共享对象竞态——基线同构）。

### D3. 阶段编排落位（eino Graph vs 自定义）

**结论：无编排层**——mastery 没有自己的 pipeline，`Run()` 直接委托 chat 的自定义 loop 函数（见 impl-capability-chat-solve design D3 的取舍结论），不引入 eino Graph。learning 引擎是纯同步函数库，与 eino 无关；唯一的 LLM 调用是 REST 端 `generate-from-notebook`，为一次性 `ChatModel.Generate`（非流式，容错 JSON 解析），同样无需编排。

### D4. stage 事件发射点

与 chat 完全一致：整个 turn 仅 `bus.Stage("responding", source="chat")` 一对 stage 事件（loop 由 chat pipeline 持有，source 沿用基线取值，以 M3 fixtures 为准逐字段对齐）；五个工具的 tool_call/tool_result 事件在该 stage 内发出；result 经 `EmitCapabilityResult`（response payload + `metadata.cost_summary`）。mastery 不新增任何专属 stage。

### D5. SequenceMatcher 移植方案与 golden case 对比测试

- **算法**：在 `internal/learning/ratcliff.go` 手写 Ratcliff-Obershelp：`ratio = 2*M / (len(a)+len(b))`，M 为递归最长公共子串匹配的总长。逐条复刻 CPython `difflib.SequenceMatcher` 语义：`find_longest_match` 的 b2j 索引表、**autojunk 启发式**（len(b) ≥ 200 时把出现率 >1% 的字符列入 junk——判分输入 expected ≤ 30 字符走模糊匹配，实际不会触发，但仍实现以保逐位一致）、匹配块选取的「最早最长」平局规则、`get_matching_blocks` 的队列遍历顺序。不使用任何现成 Go 编辑距离库（Levenshtein 与 Ratcliff-Obershelp 结果不同，spec 明令禁止替代）。
- **golden case 对比测试**：写一个 Python 脚本（基线仓库内只读运行）批量产出 `(user, expected, question_type) → bool` 与 `(a, b) → ratio` 的 JSONL golden 文件（覆盖：精确相等、大小写/空格、0.85 阈值两侧、多字节中文、空串、>30 字符不模糊、open 关键词切分 `[,;，；。\n]+` 边界、choice 去空格），checked in 到 `internal/learning/testdata/`；Go 测试逐行断言一致。`compute_mastery` 与 `scheduler` 同法产出 golden（浮点按同一计算顺序，比较到 1e-12）。

### D6. 存储与 REST 落位

- `LearningStore`：`<workspace>/learning/<book_id>.json`；`book_id` 含 `/ \ .. :` 拒绝（与工具侧白名单双保险）；保存在进程级 `sync.Mutex` 内，`version++`、刷新 `updated_at`，临时文件 + `os.Rename` 原子写；`get_or_create` 对新 path 立即落盘。
- REST：gin route group `/api/v1/learning`，8 端点按 spec；参数校验顺序 book_id → body 结构（字段级错误 422、语义错误 400）；`init-modules` 先经 turn-runtime 取消该 path 活跃 turn 再 replace；`generate-from-notebook` 截 20 条记录、HTML 转义、LLM 输出容错解析（无效结构 502）。

### D7. 与 Python 基线文件映射表

| Python 基线 | Go 落位 |
| --- | --- |
| `deeptutor/capabilities/mastery/capability.py` | `internal/capabilities/mastery/capability.go` |
| `deeptutor/capabilities/mastery/loop.py` | `internal/capabilities/mastery/loopcap.go` |
| `deeptutor/capabilities/mastery/tools.py` | `internal/capabilities/mastery/tools.go` |
| `deeptutor/capabilities/mastery/choices.py` | `internal/capabilities/mastery/choices.go` |
| `deeptutor/capabilities/mastery/prompts/{en,zh}/system.md` | `internal/capabilities/mastery/prompts/{en,zh}/system.md`（原样拷贝） |
| `deeptutor/learning/grading.py`（含 `difflib` 依赖） | `internal/learning/grading.go` + `ratcliff.go` |
| `deeptutor/learning/mastery.py` | `internal/learning/mastery.go` |
| `deeptutor/learning/scheduler.py` | `internal/learning/scheduler.go` |
| `deeptutor/learning/policy.py` | `internal/learning/policy.go` |
| `deeptutor/learning/service.py` | `internal/learning/service.go` |
| `deeptutor/learning/storage.py` | `internal/learning/storage.go` |
| `deeptutor/learning/models.py` | `internal/learning/models.go` |
| `deeptutor/learning/prompts.py` + `learning/prompts/{en,zh}.yaml` | `internal/learning/prompts/`（shared.PromptStore 寻址） |
| `deeptutor/api/routers/mastery_path.py` | `internal/api/routers/learning.go` |

## Risks / Trade-offs

- [Ratcliff-Obershelp 移植细节（平局规则/autojunk）出错导致边界样本判分漂移] → golden case 覆盖阈值两侧 + 随机 fuzz 对比（Python 脚本离线生成 1 万随机对的 ratio golden），CI 逐行断言。
- [浮点顺序差异使 `compute_mastery` 尾数不同] → Go 按与 Python 相同的累加顺序实现（先加权和再除权重和），golden 比较容差 1e-12；展示层 3 位小数四舍五入另测。
- [`LEARNING_DEBUG` 环境变量改变时间单位，测试易误置] → 调度器把 `unit` 作为注入参数，env 解析只在构造点做一次。
- [每工具调用新建 store/service 带来重复文件 IO] → 与基线同构（正确性优先），progress 文件为 KB 级，单机可忽略。

## Migration Plan

无数据迁移（JSON 布局与基线一致，老 progress 直接可读）。实现顺序：learning 引擎（可独立单测）→ 存储/REST → capability/工具接线 → fixtures。回滚 = 不注册 capability + 不挂 REST group。

## Open Questions

- session question bank 同步（notebook entries）的失败告警日志级别与基线是否需一致——仅日志差异，实现时确认即可。
