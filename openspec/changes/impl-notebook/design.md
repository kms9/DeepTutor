# Design: impl-notebook

## Context

基线实现：学习笔记本在 `deeptutor/services/notebook/service.py`（JSON 文件存储于 `data/user/workspace/notebook/`）+ `api/routers/notebook.py`；错题本在 `api/routers/question_notebook.py`，表 DDL 在 `services/session/sqlite_store.py`（与 sessions 同一 SQLite 库）。技术约束：SQLite 用 modernc.org/sqlite + database/sql（免 CGO），REST 用 gin route group，摘要生成经 eino ChatModel 流式。

## Goals / Non-Goals

**Goals:**

- `internal/notebook/`：学习笔记本文件 manager + question-notebook SQLite store。
- `/api/v1/notebook` 与 `/api/v1/question-notebook` 全路由对等（含 SSE 摘要流、base64 图片物化）。
- `list_notebook`/`write_note` 工具的存储接口（与 REST 同源）。

**Non-Goals:**

- 两工具的 LLM schema 与参数校验（tools-builtin）；`notebook_entries` 等表的 DDL 创建与 AttachmentStore 本体（session-store change，本模块复用其 `*sql.DB` 与 store 接口）；quiz 判题产生条目的时机（capability-question）。

## Decisions

### D1. Go 包结构

```
internal/notebook/
├── model.go        # Notebook / Record（type 六域、created_at epoch 秒）
├── filestore.go    # notebooks_index.json + <id>.json 读写（对应 services/notebook/service.py）
├── manager.go      # CRUD/记录管理门面 + thinking tags 清洗（REST 与工具共用）
├── summary.go      # 摘要 agent：eino ChatModel 流式，按 metadata.ui_language 选语言
└── questionstore.go # notebook_entries/categories/entry_categories 查询与 upsert

internal/api/notebookapi/router.go          # gin group /api/v1/notebook
internal/api/questionnotebookapi/router.go  # gin group /api/v1/question-notebook
```

### D2. 关键接口签名

```go
// internal/notebook/manager.go —— REST 与 list_notebook/write_note 工具共用
type Manager struct{ paths config.PathResolver; sum *Summarizer }

func (m *Manager) ListNotebooks(ctx context.Context, u auth.UserContext) ([]Notebook, error)
func (m *Manager) GetNotebook(ctx context.Context, u auth.UserContext, id string) (*NotebookDetail, error) // 未知 ErrNotFound
func (m *Manager) CreateNotebook(ctx context.Context, u auth.UserContext, req CreateReq) (*Notebook, error)
func (m *Manager) UpdateNotebook(ctx context.Context, u auth.UserContext, id string, patch Patch) error
func (m *Manager) DeleteNotebook(ctx context.Context, u auth.UserContext, id string) error

// summary 为空时内部调 Summarizer（onChunk 供 SSE；非流式路径传 nil）
func (m *Manager) AddRecord(ctx context.Context, u auth.UserContext, req AddRecordReq, onChunk func(string)) (*AddRecordResult, error)
func (m *Manager) UpdateRecord(ctx context.Context, u auth.UserContext, nbID, recID string, patch RecordPatch) error
func (m *Manager) DeleteRecord(ctx context.Context, u auth.UserContext, nbID, recID string) error

// internal/notebook/questionstore.go —— 复用 session-store 的 *sql.DB（modernc.org/sqlite）
type QuestionStore struct{ db *sql.DB; attachments session.AttachmentStore }

func (q *QuestionStore) UpsertEntry(ctx context.Context, e EntryUpsert) (*Entry, error)
// INSERT ... ON CONFLICT(session_id, turn_id, question_id) DO UPDATE —— 行 id 不变
func (q *QuestionStore) ListEntries(ctx context.Context, f EntryFilter) (items []Entry, total int, err error)
func (q *QuestionStore) LookupByQuestion(ctx context.Context, sessionID, questionID string, missingOK bool) (*Entry, error)
func (q *QuestionStore) PatchEntry(ctx context.Context, id int64, patch EntryPatch) error // 空更新 ErrBadRequest
func (q *QuestionStore) DeleteEntry(ctx context.Context, id int64) error
// categories：List(含计数)/Create(409)/Rename/Delete(级联清关联)；entry↔category 关联增删
```

### D3. 文件存储兼容策略（filestore.go）

- JSON 序列化保持基线字段名与类型（`created_at` epoch 秒为数值）；写入经临时文件 + rename 原子落盘；读损坏文件降级空数据并告警（spec 要求）。
- 记录数冗余在 `notebooks_index.json`，任何记录增删后同步索引（与基线一致）。
- per-user 根经 `config.PathResolver` 解析，工具与 REST 均不得绕过（多用户安全）。

### D4. base64 图片物化（upsert）

`POST /entries/upsert` 的 `user_answer_images`：

- 元素为 base64 → 解码写入 session-store 的 AttachmentStore，DB 只存 `{id,url,filename,mime_type}` 引用 JSON；
- `null` → 不动既有图片；空数组 → 显式清空；
- 单图解码/写入失败 → 丢弃该图记 warning，不整体失败；
- 响应绝不回传 base64。session 不存在（外键）→ 404。

### D5. SSE 摘要流（router）

`POST /add_record_with_summary`：gin handler 以 `http.Flusher` 写 SSE；`Manager.AddRecord` 的 `onChunk` 回调逐块发 `summary_chunk` 事件，完成后发含记录的 `result`（失败发 `error`）。落库 summary 为清洗 thinking tags 后全文（清洗逻辑与基线正则对齐，进 `manager.go` 统一入口，`add_record` 直传 summary 同样清洗）。

### D6. 与 Python 基线文件映射

| Python 基线 | Go 落位 |
| --- | --- |
| `services/notebook/service.py` | `internal/notebook/{filestore,manager,model}.go` |
| `api/routers/notebook.py` | `internal/api/notebookapi/router.go` + `internal/notebook/summary.go` |
| `api/routers/question_notebook.py` | `internal/api/questionnotebookapi/router.go` + `internal/notebook/questionstore.go` |
| `services/session/sqlite_store.py`（notebook 表 DDL） | session-store change 的 schema（本模块只消费） |
| `tools/list_notebook.py` / `write_note.py`（存储侧） | `Manager` 接口（工具面在 tools-builtin） |

### D7. 备选方案取舍

- **学习笔记本迁 SQLite**：被否——破坏基线文件格式兼容与回滚前提（acceptance.md 3.2 反向兼容），保持 JSON 文件。
- **upsert 实现**：备选「SELECT 后分支 INSERT/UPDATE」——被否，存在并发竞态；用 SQLite 原生 `ON CONFLICT ... DO UPDATE`（modernc.org/sqlite 支持），行 id 天然不变。
- **摘要 agent 独立服务化**：被否——单次流式补全，进程内 Summarizer 足够。

## Risks / Trade-offs

- [JSON 数值类型漂移（Go 写 float 导致 Python 读出 1.0 时间戳）] → `created_at` 用整型序列化；round-trip 测试对基线样例逐字节比对关键字段。
- [同一 notebook 文件的并发写（REST 与工具同时 add_record）] → manager 内 per-notebook 互斥锁 + 原子写。
- [thinking tags 清洗正则与基线不一致导致 summary 差异] → 从基线移植正则并以基线样例做 golden 测试。
- [AttachmentStore 接口未冻结（session-store 并行开发）] → 先以接口 + 内存 stub 联调，合入需 session-store 已验收（ROADMAP 并行约束）。
