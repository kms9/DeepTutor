## Context

`/api/v1/ws` 是前端 `web/lib/unified-ws.ts` 消费的唯一统一实时通道，Wave 1 / M1 的协议面出口。基线实现 `deeptutor/api/routers/unified_ws.py`（FastAPI WebSocket + asyncio task 订阅管理）。Go 版约束：gin-gonic/gin 组织 `/api/v1` route group，gorilla/websocket 在 gin handler 内 upgrade；对基线的 11 种消息类型做**字节级对等**（以录制的 WS golden fixtures 验收）。上游：`impl-turn-runtime`（全部业务动作）、`impl-foundation-stream`（StreamEvent 形状与全局 bus 注册表）；鉴权门对接 multi-user-auth 的接口（M1 期可注入单用户桩）。

## Goals / Non-Goals

**Goals:**

- 11 种消息类型逐字段对等（错误文案、错误帧形状、rejected metadata、`active_turn_info` 响应）。
- gorilla/websocket 的连接生命周期（accept 前鉴权、断开清理）与写并发安全。
- 订阅任务管理：同 turn 同连接单活跃订阅、session 订阅独立键、断开全量清理不影响 turn 执行。

**Non-Goals:**

- 不实现 legacy WS（`/api/v1/chat`、book/question/partners/knowledge progress WS——各自模块）。
- 不实现鉴权策略本体（multi-user-auth）；此处只定义 `WSAuthenticator` 注入点。
- 不实现事件生产与 seq 语义（turn-runtime / foundation-stream），WS 层只透传。

## Decisions

### D1. gin + gorilla/websocket 落位

```go
// internal/api/router.go
v1 := engine.Group("/api/v1")
v1.GET("/ws", ws.UnifiedHandler(deps)) // gin.HandlerFunc

// internal/api/ws/unified.go
var upgrader = websocket.Upgrader{ CheckOrigin: func(*http.Request) bool { return true } /* 同基线本机信任面 */ }

func UnifiedHandler(deps Deps) gin.HandlerFunc {
    return func(c *gin.Context) {
        user, err := deps.Auth.Authenticate(c.Request) // accept（upgrade）前鉴权，等价 ws_require_auth
        if err != nil { c.AbortWithStatus(http.StatusForbidden); return } // 不进入消息循环
        conn, err := upgrader.Upgrade(c.Writer, c.Request, nil)
        if err != nil { return }
        newWSConn(conn, user, deps).run(c.Request.Context()) // 阻塞至连接结束
    }
}
```

鉴权失败在 upgrade **之前**以 HTTP 状态拒绝（gorilla 场景下等价基线「拒绝连接不 accept」）。用户身份存入 `wsConn` 结构体字段，随连接生命周期释放——不使用 goroutine-local 隐式传播（呼应 acceptance.md **D-005** 的显式 context 原则，本模块无新差异）。

### D2. 连接结构与写并发控制

gorilla/websocket 不允许并发写。设计为**单写者 goroutine + 发送通道**：

```go
type wsConn struct {
    conn   *websocket.Conn
    user   auth.Identity
    sendCh chan map[string]any      // 全部下行帧唯一入口（buffered，如 256）
    closed atomic.Bool              // 发送失败后置位，停止后续推送
    subs   map[string]*subscription // 键：turn_id 或 "session:<session_id>"
    subsMu sync.Mutex
    deps   Deps
}
type subscription struct{ cancel context.CancelFunc; done chan struct{} }
```

- **读循环**（handler goroutine 本体）：顺序 `ReadMessage` → JSON 解码 → 按 `type` 分发。解码失败回 `Invalid JSON.`，未知 type 回 `Unknown type: <type>`，任何 handler 错误不退出循环（仅 socket 错误退出）。
- **写循环**（单 goroutine）：从 `sendCh` 取帧 → `safeSend`：`json.Marshal` 失败时对 metadata 值做 `fmt.Sprint` 降级重序列化（等价基线 `default=str`）；`WriteMessage` 失败置 `closed` 并丢弃后续帧，不向读循环抛错。
- **订阅转发 goroutine**（每订阅一个）：消费 `SubscribeTurn` 返回的事件通道 → `event.ToWire()`（保持 `{type, source, stage, content, metadata, session_id, turn_id, seq, timestamp}` 形状）→ `sendCh`。对应基线 `_forward` task。

### D3. 订阅生命周期

- `subscribe(key, turnID, afterSeq)`：持 `subsMu` 先 `stopSubscription(key)`（cancel + 等 `done`，对应基线「重复订阅先停旧」）再建新 `subscription`。turn 订阅键 = `turn_id`，session 订阅键 = `session:<session_id>`，互不挤占。
- `message/start_turn` 与 `regenerate` 成功后自动 `subscribe(turnID, turnID, 0)`；runtime 的 `*RejectedError` 转完整事件形状错误帧（`source: "unified_ws"`、`metadata.turn_terminal/status/reason`、`session_id` 回显、`seq: 0`）。
- `unsubscribe`：按携带的 `turn_id`/`session_id` 分别停键，匹配不到 no-op；不触碰 runtime（turn 照常执行）。
- 连接结束（读循环退出）：`defer` 依次 cancel 全部订阅并等退出 → close `sendCh` 停写循环 → 释放身份绑定。连接级未捕获 panic 经 `recover` 尽力回一条 `{type:"error"}` 再走同样清理。

### D4. 消息路由表（对基线逐条映射）

| type | runtime 调用 | 错误语义 |
| --- | --- | --- |
| `message`/`start_turn` | `StartTurn(payload)` + 自动订阅 | rejected 事件形状错误帧 |
| `ping` | — 直接回 `{type:"pong"}` | — |
| `subscribe_turn`/`resume_from` | `SubscribeTurn(turnID, afterSeq)`（`resume_from` 的水位参数名为 `seq`） | 缺 `turn_id` → `Missing turn_id.` |
| `subscribe_session` | `SubscribeSession(sessionID, afterSeq)`；无活跃 turn 时通道即关（静默无帧） | 缺 `session_id` → `Missing session_id.` |
| `check_active_turn` | store 活跃行 + `HasLiveExecution`；执行体死则置 `cancelled`（`Stale turn after restart`）回 `status:"none"` | 缺参回错误；`status` 可为 `running`/`waiting_input`（**D-002**，前端同等视为进行中） |
| `unsubscribe` | 本地订阅表操作 | no-op 容忍 |
| `cancel_turn` | `CancelTurn(turnID)`；false → `Turn not found: <id>` | 成功无响应帧 |
| `submit_user_reply` | `SubmitUserReply(turnID, text, answers)`（answers 清洗：questionId|id 归一、无效剔除、空视为无） | false → `Turn <id> is not awaiting a user reply.` |
| `regenerate` | `RegenerateLastTurn(sessionID, overrides)` + 自动订阅 | `reason: regenerate_busy | nothing_to_regenerate` 的 rejected 帧 |
| `user_input` | `core.LookupTurnBus(turnID)` → `bus.SubmitInput(content)`（唤醒全部 `WAIT_FOR_INPUT` 等待者并清空队列） | 缺参 → `Missing turn_id for user_input.`；无 bus → `No active bus for turn: <id>` |

### D5. 心跳

不启用 gorilla 的协议级 ping/pong 与 read deadline——基线服务端不主动要求心跳、45s 判死由**前端**负责；服务端只需对应用层 `{"type":"ping"}` 帧立即回 `{"type":"pong"}`。保持协议面最小与 fixtures 对等。

### D6. 与 Python 基线映射

| Go | Python 基线 |
| --- | --- |
| `internal/api/ws/unified.go` `UnifiedHandler` / 读循环 | `unified_ws.py` `unified_websocket` 主循环 |
| `wsConn.safeSend` + 写循环 | `safe_send`（`default=str` 序列化 + 失败标记关闭） |
| `subscription` + `stopSubscription` | `subscribe_turn`/`subscribe_session`/`stop_subscription` 的 asyncio task 管理 |
| 事件 `ToWire()` | `StreamEvent.to_dict()` 序列化形状 |
| `user_input` 路由 | `stream_bus.py` 全局注册表 + `submit_input` |

## Risks / Trade-offs

- [写通道满阻塞读循环] `sendCh` 满会背压订阅转发 goroutine → 转发侧带 `closed` 检查与 ctx select；容量按 fixtures 峰值压测调整（回放 1 万事件 < 1s 指标同样约束本层）。
- [字节级对等 vs Go JSON 序列化差异] 字段序/空值渲染（`null` vs 缺席）与 Python `json.dumps` 不同会挂 fixtures diff → `ToWire()` 用固定字段序的 struct 序列化，golden fixtures 对比器按字段语义比较（忽略 timestamp，白名单见 acceptance.md §3.1）。
- [订阅泄漏] 停旧订阅等待 `done` 若转发 goroutine 卡在发送会死锁 → cancel 先行、发送侧 select ctx；goleak 全覆盖 WS 测试。
- [鉴权依赖未就绪] multi-user-auth 在 Wave 4 → `WSAuthenticator` 接口 + 单用户默认实现（admin 身份），行为开关与基线单用户模式一致，接口冻结避免返工。
