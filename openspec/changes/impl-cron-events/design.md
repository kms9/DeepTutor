# Design: impl-cron-events

## Context

基线实现：`deeptutor/services/cron/service.py`（store + scheduler 循环）、`executor.py`（chat/partner job 执行）、`deeptutor/events/event_bus.py`（asyncio 队列版 EventBus）。Go 版为单机单进程：一个 in-process scheduler 独占 `cron/jobs.json`，不引入外部队列。执行 chat job 需经 turn-runtime/orchestrator 跑完整轮次并回写 session。

## Goals / Non-Goals

**Goals:**

- `internal/cron/`：job store（损坏取证）、三种 schedule 校验与 next-run 计算、单协程调度循环（补跑/过期清理/即时唤醒）、chat/partner executor。
- `internal/events/`：全局 EventBus（非阻塞发布、handler 异常隔离、flush/优雅停止）。
- `cron` 工具的服务侧接口（owner key 过滤、`_cron_in_context` 拒绝）。

**Non-Goals:**

- `cron` 工具的 LLM schema 与参数归一（tools-builtin）；partner 消息总线本体（partners，M4 前 executor 对 partner job 统一记 `skipped`）；StreamBus（foundation-stream）。

## Decisions

### D1. Go 包结构

```
internal/cron/
├── model.go       # Job / Schedule / Owner（chat|partner）/ State / RunRecord
├── store.go       # cron/jobs.json 原子读写 + 损坏改名 .corrupt-<ts>
├── schedule.go    # at/every/cron 校验与 NextRun 计算（robfig/cron parser）
├── scheduler.go   # 单协程 tick 循环 + 唤醒 channel
├── executor.go    # chat job → orchestrator 轮次 + session 回写；partner job → 总线注入（M4 前 stub skipped）
├── service.go     # Schedule/List/Cancel（owner key 过滤、in-context 拒绝）—— cron 工具消费
└── notify.go      # 桌面通知（可选增强，best-effort）

internal/events/
└── bus.go         # 全局 EventBus 单例
```

### D2. cron 表达式：robfig/cron parser + 自研调度循环（关键取舍）

- **表达式解析/next-run 计算用 `robfig/cron/v3` 的 `cron.ParseStandard`**（5 段标准表达式，`SpecSchedule.Next(t)` 在给定 `time.Location` 求下一触发点）：时区/DST 边界处理成熟，避免自研解析器的正确性风险。IANA `tz` 经 `time.LoadLocation` 校验（未知时区拒绝）；「永不触发」检测：`Next` 结果超过 now+5 年视为永不触发并拒绝（与基线语义一致）。
- **调度循环不用 robfig 的 `cron.Cron` runner，自研单协程 timer 循环**：robfig runner 的 Entry 生命周期与本 spec 的持久化语义（补跑、过期 `at` 清理、run_history、`delete_after_run`、外部编辑 store 60 秒拾取、增删即时唤醒）不匹配，硬套需要大量绕行；自研循环 ~100 行且完全对齐基线 `service.py` 语义。
- 结论：**robfig 只当"表达式计算库"用，调度器自研**。

```go
// internal/cron/schedule.go
type Schedule struct {
    Kind         string // "at" | "every" | "cron"
    AtMs         int64  // at
    EverySeconds int64  // every，最小 30
    CronExpr     string // cron，5 段
    TZ           string // IANA，可选
}
func ValidateSchedule(s Schedule, now time.Time) error
func NextRun(s Schedule, now time.Time) (int64, bool) // epoch ms；false = 不再触发

// internal/cron/scheduler.go —— 单协程
type Scheduler struct {
    store  *Store
    exec   Executor
    wake   chan struct{} // 增删 job 时非阻塞写入，立即唤醒
}
func (s *Scheduler) Run(ctx context.Context) // tick: 执行到期 → 更新 state → 睡 min(最近到期, 60s)
```

### D3. 启动补跑/过期清理语义（scheduler.go 启动路径）

服务启动加载 store 后、进入首个 tick 前：

1. 遍历 `at` 型一次性 job：`at < now` → 直接删除（**不补跑**，停机期间过期的提醒已失去时效）；
2. 重复型（`every`/`cron`）job：`next_run_at_ms < now` 保持"到期"状态 → **首个 tick 补跑一次**，随后按 `NextRun` 重排；
3. tick 内单 job 执行 panic/error → `recover` 后记该 job `state.last_status="error"` 与 `last_error`，循环继续。

### D4. chat job 执行流程（executor.go）

1. 以 `owner.user_id`/`is_admin` 建用户 scope（显式 context 传递，呼应 D-005 原则）；
2. `owner.session_id` 查 session store：不存在 → run 记 `status="error", error="session no longer exists"`；
3. message 包装为提醒式 prompt（用户语言自然转达、不暴露 job id）；
4. 组装 UnifiedContext：既有对话史、`active_capability="chat"`、metadata `{source:"cron", cron_job_id}`、`turn_id = "cron-<job_id>-<rand>"`；
5. 经 `orchestrator.RunTurn` 同步跑完整 chat 轮次（agent loop、工具可用，`_cron_in_context=true` 注入以拒绝再 schedule）；
6. 最终回复非空 → user prompt + assistant 回复两条消息（metadata 带 `cron_job_id`）追加 session；无产出 → 以最后 error 或 `"turn produced no answer"` 记 `error`；
7. run 结果（status/时长/error）入 `run_history`（截断至 10 条）。

### D5. EventBus（bus.go）

```go
type EventType string // SOLVE_COMPLETE | QUESTION_COMPLETE | CAPABILITY_COMPLETE
type Event struct {
    Type      EventType
    TaskID    string
    UserInput string
    AgentOutput string
    ToolsUsed []string
    Success   bool
    Metadata  map[string]any
    EventID   string
    Timestamp time.Time
}
type Handler func(context.Context, Event)

type Bus struct{ queue chan Event; ... } // 有界队列 + 单处理协程

func (b *Bus) Publish(ev Event)                      // 非阻塞入队
func (b *Bus) Subscribe(t EventType, h Handler) (unsubscribe func()) // 幂等
func (b *Bus) Flush(ctx context.Context) error       // 限时等待排空
func (b *Bus) Stop(ctx context.Context) error        // 限时排空后停协程
```

处理协程按订阅顺序逐个调用 handler，每次调用包 `recover`（异常记日志不影响后续）；无订阅者静默丢弃。与 StreamBus 边界：StreamBus 是 per-turn StreamEvent 扇出（foundation-stream），EventBus 是跨模块能力完成粗粒度通知。

### D6. 与 Python 基线文件映射

| Python 基线 | Go 落位 |
| --- | --- |
| `services/cron/service.py`（store + 调度循环 + 工具侧 API） | `internal/cron/{model,store,schedule,scheduler,service}.go` |
| `services/cron/executor.py` | `internal/cron/executor.go` |
| `events/event_bus.py` | `internal/events/bus.go` |
| 桌面通知（`osascript` 分支） | `internal/cron/notify.go`（可选增强，登记于 spec「桌面通知」条目） |

### D7. 备选方案取舍汇总

- **全用 robfig `cron.Cron` runner**：被否（见 D2，持久化/补跑/取证语义不匹配）。
- **全自研 cron 表达式解析**：被否，时区与 DST 边界易错；robfig parser 是社区标准。
- **EventBus 用多队列 per-type**：被否，基线为单队列顺序派发，保持一致最简。

## Risks / Trade-offs

- [robfig `ParseStandard` 与基线 croniter 的边缘表达式行为差异（如 `L`/`W` 扩展、星期 7）] → 只接受 5 段标准语法（与基线一致），差异用例做双实现对照测试（golden case）。
- [chat job 执行占用调度协程导致后续 job 延迟] → 执行在独立 goroutine、以 owner key 去重防同 job 重入；调度协程只做分发（与基线异步执行等价）。
- [store 被外部（人工/回滚期 Python）编辑] → 60 秒 sleep 上限保证一分钟内拾取；写路径统一原子写。
- [进程崩溃丢失 in-flight run 状态] → run_history 只在执行结束落盘，崩溃后 job 保持"到期"由补跑语义兜底（重复型）或按 at 清理，不额外引入 WAL。
