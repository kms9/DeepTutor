# Design: impl-session-store

## Context

基线持久层在 `deeptutor/services/session/sqlite_store.py`（schema + 增量迁移 + store API）、`deeptutor/services/session/protocol.py`（store 协议）、`deeptutor/api/routers/sessions.py`（REST），孤儿 turn 收敛与 seq 分配的调用方在 `deeptutor/services/session/turn_runtime.py`。硬约束：Python v1.5.2 写出的 `data/user/chat_history.db` 免迁移直接打开，Go 写入后回切 Python 仍可用（acceptance.md §3.2 双向兼容）；REST 面与前端逐字段对等（§3.1 golden spec）。技术选型固定：`modernc.org/sqlite`（免 CGO）+ `database/sql`，REST 用 `gin-gonic/gin`。

## Goals / Non-Goals

**Goals:**

- schema 幂等初始化 + 老库增量列迁移 + notebook UNIQUE 重建 + legacy 库搬迁，双向兼容。
- store API 全量语义：会话生命周期、消息三态 parent、turn 状态机、事件追加/回放、列表/导入判别、错题本。
- `/api/v1/sessions` 路由族逐字段对等（含 1 MiB UI 截断）。
- D-001 落地：`(turn_id, seq)` 冲突报错而非覆盖。

**Non-Goals:**

- turn 的执行与 seq 分配（归 turn-runtime；本模块只提供 `append_turn_event` 的存储语义与孤儿判定原语）。
- 附件存储本体（附件清理走接口回调，落在 attachments 相关模块）；quiz 判题逻辑（归 capability-question，本模块只做成绩落库）。
- PocketBase session store（acceptance.md §1 明确不在范围）。

## Decisions

### D1. Go 包结构

```
deeptutor-go/internal/session/
├── schema.go      # DDL 常量 + EnsureSchema（CREATE TABLE IF NOT EXISTS）
├── migrate.go     # 增量列迁移、parent 链回填、notebook UNIQUE 重建、legacy 库搬迁
├── store.go       # Store：连接管理、单写者互斥、事务辅助
├── sessions.go    # 会话生命周期 + 列表/导入判别
├── messages.go    # add_message / get_messages_for_context
├── turns.go       # turn 状态机 / append_turn_event / get_turn_events / 孤儿收敛
├── notebook.go    # 错题本 upsert / 分类多对多
├── types.go       # Session/Message/Turn/NotebookEntry 等（对应基线 protocol.py）
└── *_test.go
deeptutor-go/internal/api/
└── sessions.go    # gin route group /api/v1/sessions（含 1 MiB UI 截断）
```

### D2. modernc.org/sqlite 打开方式与并发模型

```go
dsn := "file:" + paths.ChatHistoryDB() + "?_pragma=foreign_keys(1)&_pragma=busy_timeout(5000)&_pragma=journal_mode(WAL)"
db, _ := sql.Open("sqlite", dsn)
db.SetMaxOpenConns(1) // 单连接即单写者
```

- 驱动名 `"sqlite"`（modernc 注册名），`_pragma=foreign_keys(1)` 保证每个连接 `PRAGMA foreign_keys = ON`（对应 spec 要求）。
- 并发模型：`SetMaxOpenConns(1)` + Store 级 `sync.Mutex`，全部读写串行化到单写者；每个 store 方法 `BEGIN IMMEDIATE` 事务内完成、`defer tx.Rollback()` 异常回滚——满足 spec「存储并发模型」的可观察行为（并发 `append_turn_event` 无重复无空洞）。基线是「互斥 + 每调用独立连接」，Go 用等价机制（spec 明示实现细节不作约束）。
- WAL 仅为并发读优化；回切 Python 前无需处理（SQLite WAL 库 Python `sqlite3` 直接可开）。

### D3. 打开流程与增量迁移（migrate.go）

`Open(paths *config.PathService)` 顺序：① legacy `data/chat_history.db` 存在且目标不存在 → `os.Rename` 搬迁；② `EnsureSchema`；③ 逐项探测 `PRAGMA table_info`：缺 `sessions.preferences_json`/`messages.metadata_json`/`messages.parent_message_id`/`notebook_entries.turn_id`/`user_answer_images_json`/`ai_judgment` → `ALTER TABLE ADD COLUMN`；④ 补 `parent_message_id` 时按 session 内 `id` 升序回填线性父链（单事务批量 UPDATE）；⑤ 检查 `notebook_entries` 的 UNIQUE 索引仍为 `(session_id, question_id)` → 建新表-拷贝-改名重建为 `(session_id, turn_id, question_id)`。全部迁移与基线 `sqlite_store.py` 同款、幂等。

### D4. 关键接口签名

```go
package session

type Store struct { db *sql.DB; mu sync.Mutex }
func Open(paths *config.PathService) (*Store, error)

// 会话
func (s *Store) CreateSession(id, title string) (Session, error)   // id 为空时生成 unified_<epoch_ms>_<8hex>
func (s *Store) EnsureSession(id, title string) (Session, error)
func (s *Store) GetSession(id string) (*SessionDetail, error)      // 含 status/active_turn_id/capability/preferences 派生
func (s *Store) ListSessions(limit, offset int) ([]SessionSummary, error)
func (s *Store) ListImportedSessions() ([]SessionSummary, error)
func (s *Store) RenameSession(id, title string) (Session, error)
func (s *Store) DeleteSession(id string) error

// 消息（三态 parent：Auto / Explicit(id) / Root）
type ParentRef struct{ Explicit bool; ID *int64 } // 零值=Auto；Explicit&&ID==nil=挂根
func (s *Store) AddMessage(sessionID string, m NewMessage, parent ParentRef) (Message, error)
func (s *Store) GetMessagesForContext(sessionID string, leaf *int64) ([]Message, error) // 防环上限 10000

// turn
func (s *Store) CreateTurn(sessionID, capability string) (Turn, error) // turn_<epoch_ms>_<10hex>；已有 running 报 ErrTurnActive{TurnID}
func (s *Store) FinishTurn(turnID, status, errMsg string) error        // 终态写 finished_at
func (s *Store) MarkOrphanIfRunning(turnID string) (bool, error)       // 孤儿收敛为 failed
func (s *Store) AppendTurnEvent(turnID string, ev core.StreamEvent) (core.StreamEvent, error)
func (s *Store) GetTurnEvents(turnID string, afterSeq int64) ([]core.StreamEvent, error)
func (s *Store) GetTurn(turnID string) (*TurnDetail, error)            // 含 last_seq 派生

// 错题本
func (s *Store) UpsertNotebookEntries(sessionID, turnID string, entries []NotebookEntry) (int, error)

var ErrSeqConflict = errors.New("turn event seq conflict") // D-001
```

### D5. AppendTurnEvent 与 D-001

单事务内：`SELECT 1 FROM turns WHERE id=?`（不存在报错）→ `seq<=0` 时 `SELECT COALESCE(MAX(seq),0)+1` → 回填 `seq/turn_id/session_id` → **普通 `INSERT`**（非基线的 `INSERT OR REPLACE`）→ 刷新 `turns.updated_at`。捕获 sqlite 约束错误（modernc 错误码 `SQLITE_CONSTRAINT_UNIQUE`，errors.As 判定）映射为 `ErrSeqConflict`。这是 acceptance.md §6 登记的有意差异 **D-001**：覆盖会掩盖生产者分叉，正常路径由 turn runtime 的唯一 seq 分配保证不冲突，前端影响为无。

### D6. sessions REST（gin）与 UI 截断

`internal/api/sessions.go` 用 `router.Group("/api/v1/sessions")` 注册 7 条路由，请求/响应形状照 spec 与 golden spec（acceptance.md §3.1）。1 MiB 截断在 handler 层实现 `truncateEventsForUI([]map[string]any)`：仅处理 `tool_result`/`observation` 的 `content` 与 `metadata.tool_metadata.{content,answer}`，超 1024*1024 字符截断 + 追加 `"\n\n[... content truncated]"` + 置 `_truncated: true`；操作在反序列化出的副本上进行，不回写库。错误对等：404 `"Session not found"`、running turn 配对删除 409、空 answers 400。

### D7. 与 Python 基线的映射表

| Python 基线 | Go 落位 |
| --- | --- |
| `deeptutor/services/session/sqlite_store.py` schema/迁移 | `internal/session/schema.go` + `migrate.go` |
| `sqlite_store.py` 会话/消息/turn/notebook API | `internal/session/sessions.go`/`messages.go`/`turns.go`/`notebook.go` |
| `deeptutor/services/session/protocol.py` 类型 | `internal/session/types.go` |
| `deeptutor/api/routers/sessions.py` | `internal/api/sessions.go`（gin） |
| `turn_runtime.py` 孤儿 turn 判定的存储侧 | `turns.go` `MarkOrphanIfRunning` |

## Risks / Trade-offs

- [`SetMaxOpenConns(1)` 让长查询阻塞所有操作] → 本层查询全部为索引点查/小范围扫描；1 MiB 截断在 API 层内存中做，不占连接；如实测成瓶颈可放开只读连接（WAL 下读写不互斥），行为不变。
- [modernc 与 CPython sqlite3 的方言差异（如 REAL 精度、AUTOINCREMENT 行为）] → 双向兼容测试用真实 Python 快照库跑全量 CRUD + 回切 Python 打开校验（acceptance.md §3.2）。
- [parent 三态在 JSON API 边界易失真（缺字段 vs 显式 null）] → REST/内部 API 统一用 `ParentRef` 显式建模，JSON 解码用 `json.RawMessage` 区分「未出现」与「null」。
- [D-001 使潜在的重复 seq 由静默变报错，可能暴露上游 bug] → 这正是差异目的；错误在 turn runtime 侧记日志并终止该事件写入，不影响已持久化数据（acceptance.md §6 D-001，前端影响无）。
- [quiz-results 与附件清理依赖尚未实现的模块] → 以接口注入（`AttachmentCleaner`、notebook upsert 自足），缺省实现为 no-op，不阻塞本 change 验收。
