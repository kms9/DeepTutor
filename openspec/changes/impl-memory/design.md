# Design: impl-memory

## Context

基线实现位于 `deeptutor/services/memory/`（`store.py`/`trace.py`/`paths.py`/`ids.py`/`ops.py`/`document.py`/`settings.py`/`consolidator/`/`snapshot/`）与 `deeptutor/api/routers/memory.py`。全部数据是纯文件（markdown + JSONL + meta JSON），Go 版必须对基线目录逐字节兼容续写。约束：consolidator 的 LLM 调用走 eino ChatModel（llm-provider 提供）；REST 用 gin route group；无 SQLite 参与。

## Goals / Non-Goals

**Goals:**

- `internal/memory/` 完整实现 L1/L2/L3 存储、consolidator 四操作 + run manager、snapshot、`/api/v1/memory` 约 27 条路由。
- 与基线 memory 目录免迁移互写（Go 写入后回切 Python 可继续用）。
- `read_memory`/`write_memory` 的存储接口供 tools-builtin 消费。

**Non-Goals:**

- 工具的 LLM schema 与参数面（tools-builtin spec）；trace 事件的产生时机（各 capability / turn-runtime）；多用户鉴权本体（multi-user-auth，本模块只经路径服务解析 per-user 根）。

## Decisions

### D1. Go 包结构

```
internal/memory/
├── paths.go           # per-user 根解析 + partner scope 覆盖（对应 paths.py）
├── ids.go             # ULID（Crockford base32，26 位）生成/解析（对应 ids.py）
├── trace.go           # L1 append/iterate/lookup（对应 trace.py）
├── document.go        # L2/L3 markdown 解析与渲染：entry、footnote 引用（对应 document.py）
├── ops.go             # Add/Edit ops 模型与原子应用（对应 ops.py）
├── store.go           # Store 门面：读写/删条/reset/锁（对应 store.py）
├── settings.go        # memory: 设置子树 defaults 合并（对应 settings.py）
├── migrate.go         # v1 布局归档 backup/<ts>/（幂等）
├── consolidator/
│   ├── consolidator.go  # 四模式入口（对应 modes/）
│   ├── chunker.go       # L1 分 chunk（对应 chunker.py）
│   ├── meta.go          # .meta.json seen-ids（对应 meta.py）
│   ├── parse.go         # LLM 输出解析为 ops（对应 parse.py）
│   ├── references.go    # 引用池校验（对应 references.py/guards.py）
│   └── runs.go          # run manager：seq 缓冲、SSE 重放、cancel/undo（对应 runs.py）
└── snapshot/
    ├── snapshot.go      # 实体派生 + diff + changes.jsonl（对应 adapters/diff/entity/store.py）

internal/api/memoryapi/
└── router.go          # gin route group /api/v1/memory（对应 api/routers/memory.py）
```

### D2. 关键接口签名

```go
// internal/memory/store.go
type Store struct{ ... } // 依赖 PathResolver；内部 per-path 锁表

func (s *Store) AppendTrace(ctx context.Context, ev TraceEvent)              // 永不返回 error（失败仅记日志）
func (s *Store) IterateTrace(ctx context.Context, surface string, opt IterOpt) iter.Seq2[TraceEvent, error]
func (s *Store) LookupTraceByIDs(ctx context.Context, ids []string) (map[string]TraceEvent, error)
func (s *Store) ReadDoc(ctx context.Context, layer Layer, key string) (*Document, error)
func (s *Store) OverwriteDoc(ctx context.Context, layer Layer, key, text string) error   // 临时文件+rename
func (s *Store) DeleteEntry(ctx context.Context, layer Layer, key, entryID string) error // 未命中 ErrNotFound
func (s *Store) ApplyOps(ctx context.Context, layer Layer, key string, ops []Op) (*OpsReport, error)
func (s *Store) ResetDoc(ctx context.Context, layer Layer, key string) error

// 工具存储接口（tools-builtin 消费）
func (s *Store) ReadL3Concatenated(ctx context.Context) (string, error)  // recent/profile/scope/preferences 拼接
func (s *Store) WritePreference(ctx context.Context, req PreferenceWrite) (*OpsReport, error) // 仅 L3/preferences

// internal/memory/consolidator/runs.go
type RunManager struct{ ... }
func (m *RunManager) Start(ctx context.Context, req RunRequest) (*RunHandle, error) // 同文档活动 run 冲突 → ErrConflict(409)
func (m *RunManager) Events(runID string, since int64) (<-chan RunEvent, func(), error) // seq 重放 + 阻塞跟新
func (m *RunManager) Cancel(runID string) error // 协作式；非活动 ErrConflict
func (m *RunManager) Undo(runID string) error   // undo 栈；无可回滚 ErrConflict
```

### D3. 并发写保护：per-path 互斥锁表

所有文档写路径（OverwriteDoc/DeleteEntry/ApplyOps/consolidator 落盘/WritePreference）统一经 `Store` 内部 `map[cleanAbsPath]*sync.Mutex`（`sync.Map` + LazyInit）串行化；trace 追加锁按 surface 独立（`map[surface]*sync.Mutex`）。对应基线的 per-file asyncio 锁。原子性由「写临时文件 + `os.Rename`」保证，锁只解决同进程内交错。

### D4. consolidator LLM ops 流程（eino ChatModel）

1. `update`：`chunker` 按 `.meta.json` 的 seen-ids 过滤出新增 L1/L2 源 → 分 chunk → 逐 chunk 以 `{en,zh}` prompt（`prompts/memory/`，随基线拷贝）调 `model.BaseChatModel.Generate` → `parse.go` 把 LLM 输出解析为 ops → `references.go` 用本 chunk 引用池校验（trace id / L2 entry id / snapshot ref / L3 裸 surface 白名单），不通过的 op 拒绝 → `Store.ApplyOps` 原子应用 → 更新 seen-ids。
2. `audit`：读全文 + 按引用回查原始证据 → 行级核对 prompt → ops 应用。
3. `dedup`：迭代式行级去重（`iterations` 可配），无引用池扩展。
4. `merge`：合并整理，同 dedup 走全文改写路径。
5. `llm_selection`（profile_id + model_id）覆盖默认模型：经 llm-provider 的 client 工厂按选择构造 ChatModel。
6. preferences 门禁：`RunManager.Start` 对 `layer=L3,key=preferences` 且 `mode∈{update,audit}` 直接返回 405 语义错误。

### D5. run 事件流与 SSE

- run 事件在内存 ring buffer（保留全量直至 run 终态后 TTL 清理），每事件带单调 `seq`。
- `GET /runs/{id}/events?since=N`：gin handler 以 `http.Flusher` 写 SSE；先重放 `seq>N` 缓冲，再阻塞消费 channel 直至终态。客户端断开只关闭订阅，run goroutine 继续执行（`context.WithoutCancel` 派生 run ctx）。
- legacy `POST /doc/{layer}/{key}/{update|audit|dedup}`：内部 `RunManager.Start` + 立即内联 SSE 该 run 的事件流。

### D6. 与 Python 基线文件映射

| Python 基线 | Go 落位 |
| --- | --- |
| `services/memory/paths.py` / `ids.py` / `trace.py` | `internal/memory/{paths,ids,trace}.go` |
| `services/memory/document.py` / `ops.py` / `store.py` / `settings.py` | `internal/memory/{document,ops,store,settings}.go` |
| `services/memory/consolidator/{chunker,meta,parse,references,guards,runs,modes/}.py` | `internal/memory/consolidator/` |
| `services/memory/snapshot/{adapters,diff,entity,store}.py` | `internal/memory/snapshot/` |
| `api/routers/memory.py`（约 27 路由） | `internal/api/memoryapi/router.go`（gin route group） |
| `services/memory/prompts/` | `prompts/memory/{en,zh}/`（随基线拷贝） |

### D7. 备选方案取舍

- **文档条目存储**：备选「解析进 SQLite 建索引」——被否，破坏文件事实源与基线互写兼容；保持 markdown 为唯一事实源，解析在内存进行。
- **run 事件持久化**：备选「落盘 JSONL 支持进程重启后重放」——被否，基线亦为进程内缓冲，重启后 run 本就终止；内存 ring buffer 足够。
- **锁粒度**：备选全局单锁——被否，consolidator 长写会阻塞全部 surface；采用 per-path 锁。

## Risks / Trade-offs

- [Go 与 Python 的 markdown 渲染细节差异（空行、footnote 排版）导致互写后 Python 解析漂移] → document.go 的渲染以基线输出为 golden：用基线生成的样例文档做 round-trip 测试（解析→渲染→字节对比）。
- [ULID 时间戳前缀语义不一致] → 采用 Crockford base32 26 位、前 10 字符编码毫秒，与基线 `ids.py` 单测逐例对齐。
- [LLM 输出不可控导致 ops 解析失败率高] → 与基线相同的 parse 容错 + 引用池校验拒绝，失败 op 计入 run 报告而非中断 run。
- [SSE 长连接经反向代理被断] → 事件带 `seq`，前端以 `since` 重连即可续传（spec 已约定）。
