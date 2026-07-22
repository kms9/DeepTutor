# Design: impl-foundation-stream

## Context

基线 `deeptutor/core/stream.py` 定义 `StreamEvent`（dataclass + `to_dict`）与 15 个事件类型，`deeptutor/core/stream_bus.py` 提供 asyncio 版单 turn 扇出总线（history 回放 + 实时队列 + wait_for_input + bus 注册表），seq 分配在 `deeptutor/services/session/turn_runtime.py` 的 `_publish_live_event`。前端逐字段消费该信封，Go 版序列化必须与基线逐字段对等（acceptance.md §3.1 WS fixtures 对比，忽略白名单仅 `timestamp`）。Go 侧的等价并发原语是 goroutine + channel + mutex，需要在语义上复刻 asyncio 队列的「原子快照回放」行为。

## Goals / Non-Goals

**Goals:**

- `StreamEvent` 9 字段信封、15 枚举、便捷发射器 metadata 形状与基线逐字段对等。
- `StreamBus` 扇出不重不漏、close 语义、wait_for_input/submit_input、turn_id 注册表。
- NDJSON 序列化（非 ASCII 不转义）供 CLI/日志消费。

**Non-Goals:**

- seq 的实际分配（落在 turn-runtime 模块；本模块只固化分配约定与消费端去重规则）。
- 事件持久化（`turn_events` 表归 session-store）与 WS 消息封装（归 api-unified-ws）。
- eino `StreamReader` → `StreamEvent` 的映射（归 llm-provider / agent-loop；本模块只提供目标信封）。

## Decisions

### D1. Go 包结构（`internal/core`，无子包）

```
deeptutor-go/internal/core/
├── event.go        # StreamEvent + EventType 枚举 + 构造器默认值
├── emit.go         # 便捷发射器（NewToolCallEvent 等 metadata 形状约定）
├── ndjson.go       # NDJSON 序列化
├── bus.go          # StreamBus（emit/subscribe/close/wait_for_input/submit_input）
├── busregistry.go  # turn_id → *StreamBus 注册表
└── *_test.go
```

与 project.md 目录约定 `internal/core/ # StreamEvent、StreamBus、UnifiedContext、protocol 定义` 一致（`UnifiedContext` 由 turn-runtime change 后续加入同包）。

### D2. StreamEvent 结构体与序列化

```go
package core

type EventType string

const (
    EventStageStart   EventType = "stage_start"
    EventStageEnd     EventType = "stage_end"
    EventThinking     EventType = "thinking"
    EventObservation  EventType = "observation"
    EventContent      EventType = "content"
    EventToolCall     EventType = "tool_call"
    EventToolResult   EventType = "tool_result"
    EventProgress     EventType = "progress"
    EventSources      EventType = "sources"
    EventResult       EventType = "result"
    EventError        EventType = "error"
    EventSession      EventType = "session"
    EventSessionMeta  EventType = "session_meta"
    EventDone         EventType = "done"
    EventWaitForInput EventType = "wait_for_input"
)

type StreamEvent struct {
    Type      EventType      `json:"type"`
    Source    string         `json:"source"`
    Stage     string         `json:"stage"`
    Content   string         `json:"content"`
    Metadata  map[string]any `json:"metadata"`
    SessionID string         `json:"session_id"`
    TurnID    string         `json:"turn_id"`
    Seq       int64          `json:"seq"`
    Timestamp float64        `json:"timestamp"` // Unix epoch 秒
}

func NewEvent(t EventType, opts ...EventOption) StreamEvent // Metadata 归一为非 nil 空 map、Timestamp 取创建时刻
func (e StreamEvent) ToDict() map[string]any                 // 对应基线 to_dict，恰好 9 键
func (e StreamEvent) NDJSON() ([]byte, error)                 // 单行；encoder.SetEscapeHTML(false)，encoding/json 天然不转义非 ASCII
```

关键点：`Metadata` 序列化前归一为 `{}`（Go 的 nil map 会序列化成 `null`，与基线不对等）；字段顺序不影响 JSON 对等（fixtures 对比按键取值）；无 `omitempty`，保证恰好 9 键。

### D3. StreamBus：mutex 保护下的原子「快照 + 注册」

```go
type StreamBus struct {
    mu      sync.Mutex
    history []StreamEvent
    subs    []*subscriber   // 每订阅者一个无界队列（slice + sync.Cond）
    waiters []chan string   // wait_for_input 等待者
    closed  bool
}

func NewStreamBus() *StreamBus
func (b *StreamBus) Emit(ev StreamEvent)                       // closed 后静默丢弃
func (b *StreamBus) Subscribe(ctx context.Context) <-chan StreamEvent
func (b *StreamBus) Close()
func (b *StreamBus) WaitForInput(ctx context.Context, prompt string, timeout time.Duration) string
func (b *StreamBus) SubmitInput(content string)
```

- 不重不漏的关键：`Subscribe` 在同一次 `mu.Lock()` 内完成「history 快照拷贝 + subscriber 队列注册（含 closed 标记）」，与基线 asyncio 实现的原子步骤一一对应。出流 goroutine 先吐快照、再消费队列；`Emit` 只写队列，因此回放期间并发 emit 的事件只经队列投递一次。
- 订阅者用**无界队列**（slice + `sync.Cond`）而非固定容量 channel。理由：`Emit` 语义是非阻塞投递（基线 asyncio.Queue 无界），固定容量 channel 会让慢消费者反压生产者、改变基线行为。终止信号用队列内哨兵（closed 标记），出流 goroutine 读到即 `close(ch)`。
- `WaitForInput`：先 `Emit` 一条 `wait_for_input` 事件，再注册 `chan string` 等待者；`SubmitInput` 广播给全部等待者并清空列表；`timeout > 0` 时 `select` 超时返回 `""`（不返回 error），`timeout == 0` 无限等待，`ctx` 取消同样解除阻塞。

### D4. bus 注册表

```go
func RegisterBus(turnID string, bus *StreamBus)
func UnregisterBus(turnID string)
func GetBus(turnID string) *StreamBus // 不存在返回 nil，不报错
```

`sync.RWMutex` + map 的进程内单例（包级私有变量），与基线模块级注册表对应。WS 层 `user_input` 路由：`GetBus(turnID)` 为 nil 时安全忽略。

### D5. seq 约定的落位

本模块不分配 seq：`StreamEvent.Seq` 默认 0，`StreamBus` 原样透传。分配算法（`seq<=0` 分配 `next_seq` 并自增；`seq>0` 保留并推进 `next_seq = max(next_seq, seq+1)`）固化在 spec，由 turn-runtime 的发布点实现；本模块提供消费端去重辅助 `func DedupBySeq(lastSeq int64, ev StreamEvent) bool`（`seq <= lastSeq` 跳过），供 WS 回放与 CLI 使用。

### D6. 与 Python 基线的映射表

| Python 基线 | Go 落位 |
| --- | --- |
| `deeptutor/core/stream.py` `StreamEvent`/`EventType`/`to_dict` | `internal/core/event.go` |
| `stream.py` 便捷发射器（tool_call/tool_result/progress/sources/result） | `internal/core/emit.go` |
| `deeptutor/core/stream_bus.py` `StreamBus.emit/subscribe/close` | `internal/core/bus.go` |
| `stream_bus.py` `wait_for_input`/`submit_input` | `bus.go` `WaitForInput`/`SubmitInput` |
| `stream_bus.py` `register_bus`/`unregister_bus`/`get_bus` | `internal/core/busregistry.go` |
| `turn_runtime.py` `_publish_live_event` 的 seq 分配 | 约定固化于 spec，实现归 `impl-turn-runtime` |

## Risks / Trade-offs

- [无界队列在订阅者僵死时内存增长] → 单 turn 生命周期有限且事件量级为千级（acceptance.md D 类：1 万事件回放 < 1s）；`Subscribe` 绑定 `ctx`，连接断开即释放订阅者与队列。
- [`timestamp` 浮点精度与 Python `time.time()` 不完全一致] → acceptance.md §3.1 已把 `timestamp` 列入忽略白名单，无对等风险。
- [Go `encoding/json` 默认转义 `<`/`>`/`&`] → NDJSON 与 WS 出口统一使用 `json.Encoder` + `SetEscapeHTML(false)`，并在 fixtures 对比测试中覆盖含 HTML 字符与中文的样例。
- [wait_for_input 多等待者语义（广播 + 清空）易被误实现为单播] → 直接按基线语义落测试：两个并发 `WaitForInput` 都收到同一次 `SubmitInput` 的值。

本模块无登记的有意差异（acceptance.md §6 D-001~D-008 均不涉及 foundation-stream），行为以与基线完全对齐为准。
