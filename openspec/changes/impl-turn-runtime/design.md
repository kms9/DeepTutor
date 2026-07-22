## Context

turn-runtime 是 Wave 1 / M1 的中枢模块：把 api-unified-ws 的协议动作（start/subscribe/cancel/reply/regenerate）落到「后台跑一整轮 + 事件全序持久化 + 可回放扇出」上。基线实现 `deeptutor/services/session/turn_runtime.py`（约 2k 行，`TurnRuntimeManager`）+ `context_builder.py` + `source_inventory.py`。本模块承载两处已登记有意差异：**D-002**（`waiting_input` 状态拆分）与 **D-003**（append-before-publish 事件持久化）；并依赖 session-store 的 **D-001**（`(turn_id, seq)` 冲突报错而非 `INSERT OR REPLACE`——正常路径由本模块的唯一 seq 分配保证不冲突）。技术约束：goroutine/channel 实现扇出与 per-turn 回复队列；modernc.org/sqlite 经 session-store 的 store API 访问；附件解析走 Sidecar 契约。

## Goals / Non-Goals

**Goals:**

- `TurnRuntimeManager` 的完整生命周期：准入 → 后台执行 → seq 全序 → append-before-publish → 扇出 → 收尾持久化 → 终态。
- `subscribe_turn(after_seq)` 回放 + 直播无缝衔接（无重复、无乱序），崩溃/重启（孤儿轮）语义。
- cancel / submit_user_reply / regenerate（含 `parent_message_id` 消息分支）。
- `UnifiedContext` 全量组装与 ContextBuilder 历史压缩。

**Non-Goals:**

- 不实现 WS 帧编解码与连接管理（api-unified-ws）。
- 不实现 loop 内部语义（agent-loop）、store SQL 细节（session-store）、Sidecar 解析实现（sidecar-contract）。
- source inventory 的全文抽取质量优化、多用户授权门的策略管理（multi-user-auth 提供接口，本模块只调用）。

## Decisions

### D1. Go 包结构

```
internal/
├── runtime/turn/
│   ├── manager.go        # TurnRuntimeManager：StartTurn/Subscribe*/Cancel/SubmitReply/Regenerate
│   ├── execution.go      # turnExecution：per-turn goroutine、seq 计数器、订阅者集合、回复队列
│   ├── publish.go        # publishEvent：唯一取号 + append-before-publish + 扇出 + events.jsonl 镜像
│   ├── replay.go         # subscribe 回放/直播衔接、终态合成 ERROR/DONE、孤儿轮处理
│   ├── finalize.go       # 收尾：assistant 消息持久化、narration 剔除、工件附件、失败/取消路径
│   ├── title.go          # 会话标题生成（20s 限时 + session_meta 事件）
│   └── attachments.go    # 附件规范化、存储、Sidecar 解析
├── session/contextbuilder/
│   ├── builder.go        # ContextBuilder：历史压缩、摘要水位、分支祖先链
│   └── source_inventory.go
└── core/context.go       # UnifiedContext 全量字段（扩展 runtime-registry change 的骨架）
```

### D2. 关键接口签名（api-unified-ws 的编译期契约）

```go
type TurnRuntimeManager struct { /* store session.Store; orch *runtime.ChatOrchestrator; execs sync.Map[string]*turnExecution; sidecar sidecar.Client; ... */ }

// StartTurn：准入 + 建 turn 行 + 启动后台 goroutine；返回前首条 SESSION 事件已发布。
// 拒绝以 *RejectedError{Reason, Message} 返回（WS 层转 status:"rejected" 错误帧）。
func (m *TurnRuntimeManager) StartTurn(ctx context.Context, payload map[string]any) (turn TurnInfo, session SessionInfo, err error)

// Subscribe：回放（store seq>afterSeq）→ 原子挂接直播 → 按 seq 去重；返回只读通道，
// 消费完毕或 ctx 取消后由实现关闭。终态且未见 DONE 时合成 ERROR/DONE。
func (m *TurnRuntimeManager) SubscribeTurn(ctx context.Context, turnID string, afterSeq int64) (<-chan core.StreamEvent, error)
func (m *TurnRuntimeManager) SubscribeSession(ctx context.Context, sessionID string, afterSeq int64) (<-chan core.StreamEvent, error) // 解析活跃 turn；无则返回已关闭通道

func (m *TurnRuntimeManager) CancelTurn(ctx context.Context, turnID string) bool
func (m *TurnRuntimeManager) SubmitUserReply(turnID string, text string, answers []QuestionAnswer) bool
func (m *TurnRuntimeManager) RegenerateLastTurn(ctx context.Context, sessionID string, overrides map[string]any) (TurnInfo, SessionInfo, error) // 错误 Reason: regenerate_busy / nothing_to_regenerate
func (m *TurnRuntimeManager) HasLiveExecution(turnID string) bool // check_active_turn 用
```

### D3. per-turn 执行体与扇出的 channel 设计

```go
type turnExecution struct {
    turnID, sessionID string
    seq       atomic.Int64                 // seq 唯一分配权威（事件自带 seq>0 时 CAS 推进到其后）
    mu        sync.Mutex                   // 保护 subscribers + buffer 的原子快照（订阅挂接点）
    subscribers map[*liveSubscriber]struct{}
    buffer    []core.StreamEvent           // 内存缓冲（供挂接瞬间补发）
    replyQ    chan UserReply               // ask_user 回复队列（buffered=1；收尾时 close）
    cancel    context.CancelFunc           // cancel_turn 触发
    doneHeld  *core.StreamEvent            // 暂扣的 DONE
}
type liveSubscriber struct{ ch chan core.StreamEvent } // buffered（如 256）；满则丢弃该订阅者本条，不阻塞发布
```

- **publishEvent**（唯一发布路径）：取号 → **store append（D-003：先持久化再扇出；同步 `INSERT` 进 `turn_events`，依赖 store 在 D-001 下对冲突报错暴露编程错误）** → 追加 buffer → 镜像 `events.jsonl`（失败仅 debug 日志）→ 对每个订阅者非阻塞 `select send/default`。turn 行已被删除（session 删除竞态）时跳过持久化不级联失败。
- **订阅衔接**：回放 store 事件记录 `maxSeq` → 持 `mu` 把订阅者加入集合并快照 buffer 中 `seq > maxSeq` 的事件 → 释放锁后先发快照再转发直播通道，全程按 seq 单调过滤去重。
- **DONE 暂扣**：orchestrator 的 SESSION 吞掉、DONE 存入 `doneHeld`；收尾完成（assistant 消息 + flush + 状态 `completed`）后经 publishEvent 发布（未产出则合成 `status: completed`）。
- **ask_user waiter**：`context.Metadata["wait_for_user_reply"]` 注入闭包 `func(ctx) (*UserReply, bool)`——进入等待前把 turn 置 `waiting_input`（**D-002**），`select` replyQ / turn ctx；收到回复置回 `running` 返回 `(reply, true)`；队列 close 或 ctx 取消返回 `(nil, false)`。收尾顺序：先从 execs 摘除 replyQ 并 close，再清理执行体，保证迟到 `SubmitUserReply` 确定性 false。

### D4. 状态机与孤儿轮

状态列 `running`/`waiting_input`/`completed`/`failed`/`cancelled` 写入基线 `turns` 表既有 `status` 字段（schema 不变；D-002 已登记，`check_active_turn` 面由 api-unified-ws 映射）。「活跃」判定 = `status IN (running, waiting_input)`。孤儿判定 = store 非终态 ∧ `execs` 无执行体（内存态，不依赖 store）；StartTurn 前对该 session 孤儿轮置 `failed`，SubscribeTurn 遇孤儿先终态化再走合成收尾。

### D5. ContextBuilder 与 UnifiedContext

`ContextBuilder.Build(ctx, session, leafMsgID, llm)` 产出 `conversation_history`（35% 窗口预算、摘要 40%、水位 `summary_up_to_msg_id` 不在祖先链则重建；摘要 trace 事件经注入的 `publishEvent` 回调进本 turn 事件流——满足「旁路写入统一取号」）。UnifiedContext 组装按 spec 全字段：chat 轮建 source inventory（manifest + `source_index`），非 chat 轮 legacy 拼接；`metadata` 携带 `turn_id`/waiter/`source_index`/摘要统计/llm_selection/references。

### D6. 与 Python 基线映射

| Go | Python 基线 |
| --- | --- |
| `internal/runtime/turn/manager.go` | `turn_runtime.py` `TurnRuntimeManager.start_turn/regenerate_last_turn/cancel_turn/submit_user_reply/subscribe_turn/subscribe_session` |
| `internal/runtime/turn/execution.go` | `_TurnExecution` / `_LiveSubscriber` / `_run_turn` |
| `internal/runtime/turn/publish.go` | `_publish_live_event` + `_flush_buffered_events`（改为 append-before-publish，D-003）+ `_mirror_event_to_workspace` |
| `internal/runtime/turn/replay.go` | `subscribe_turn` 的 `_track` 去重 + `_synthesize_done_event`/`_synthesize_error_event` + `_fail_orphan_running_turn` |
| `internal/runtime/turn/finalize.go` | `_persisted_answer`（narration 剔除）+ `_artifact_attachments` + 失败/取消收尾 |
| `internal/runtime/turn/title.go` | `_maybe_generate_session_title` / `_sanitize_session_title` |
| `internal/session/contextbuilder/` | `context_builder.py`、`source_inventory.py` |
| `internal/core/context.go` | `deeptutor/core/context.py`（`UnifiedContext`） |

有意差异引用：**D-002**（`waiting_input`，WS 事件面不变）、**D-003**（append-before-publish，崩溃不丢已提交事件）、依赖 **D-001**（store seq 冲突报错）。除此之外逐行为对齐，任何新差异先登记 acceptance.md §6 再实现。

## Risks / Trade-offs

- [每事件同步 INSERT 的写放大（D-003 的代价）] 高频 content chunk 逐条落库拖慢流式 → SQLite WAL 模式 + 单 writer goroutine 批量 group-commit（在「publish 前已提交」的语义内合并事务）；acceptance.md §3.4「subscribe_turn 回放 1 万事件 < 1s」做基准。
- [订阅者慢消费丢帧] 非阻塞投递丢弃后客户端出现 seq 空洞 → 与基线一致（前端靠 `resume_from` 补齐）；队列容量取基线等价值并在丢弃时记日志。
- [收尾竞态] cancel 与自然完成并发、DONE 暂扣与订阅替换交错 → 收尾入口用 `sync.Once` 收敛为单次执行，全部收尾步骤独立 recover（spec：互相独立容错，状态更新必须执行）。
- [regenerate 载荷复原不全] request snapshot 字段缺漏导致 overrides 优先级错 → snapshot 字段集以基线 `_request_snapshot_metadata` 为准逐字段对拷，golden 用例回归。
- [Sidecar 不可用] 附件解析失败阻塞起轮 → 解析失败非致命（`extracted_text` 缺席、warning 日志），主链路可用（对应 acceptance.md §3.4 Sidecar 降级项）。
