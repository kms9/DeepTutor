# Design: impl-knowledge

## Context

基线实现位于 `deeptutor/knowledge/`（`manager.py`/`initializer.py`/`add_documents.py`/`kb_types.py`/`naming.py`/`progress_tracker.py`）、REST 在 `api/routers/knowledge.py`（约 46 条路由）、归属解析在 `multi_user/knowledge_access.py`。Go 版是 facade：目录/注册表/进度/访问控制在 Go，索引与检索 pipeline 本体（llamaindex/pageindex/graphrag/lightrag）经 typed 契约调用 Python Sidecar。技术约束：REST 用 gin route group，progress WS 用 gorilla/websocket 在 gin handler 内 upgrade。

## Goals / Non-Goals

**Goals:**

- `internal/knowledge/`：注册表（`kb_config.json` 文件锁 + 原子写）、生命周期、embedding reconcile、进度追踪 + WS broadcaster、连接型 KB、linked folders、provider 配置面。
- `/api/v1/knowledge` 约 46 条路由 + `WS /{kb_name}/progress/ws` 与基线对等。
- 老 KB（`data/knowledge_bases/`）免迁移可用。

**Non-Goals:**

- 索引/检索/解析 pipeline 本体与 embedding 计算（sidecar-contract）；`rag` 工具的 LLM 参数面（tools-builtin）；多用户 grant 管理面（multi-user-auth，M2 可先以单用户路径实现，归属解析留接口）。

## Decisions

### D1. Go 包结构

```
internal/knowledge/
├── types.go         # KBEntry / KBStatus / EmbeddingSignature / IndexVersion（对应 kb_types.py）
├── naming.go        # KB 名校验 + 默认 KB 别名归一（对应 naming.py）
├── registry.go      # kb_config.json 读写：flock 文件锁 + 临时文件 rename 原子写；自动注册磁盘 KB
├── manager.go       # 生命周期门面：create/upload/delete/reindex/retry（对应 manager.py/initializer.py/add_documents.py）
├── reconcile.go     # embedding 指纹对照 version-N（含 legacy 布局）
├── progress.go      # progress tracker 持久化 + broadcaster（对应 progress_tracker.py）
├── connected.go     # Obsidian / folder / LightRAG server 连接型 KB
├── linked.go        # linked folders 增删查 + 增量同步 sync state
├── access.go        # 归属解析：可见/可写（对应 multi_user/knowledge_access.py；M2 先单用户实现）
└── providers.go     # rag provider / pipeline 配置面与 preflight

internal/api/knowledgeapi/
├── router.go        # gin route group /api/v1/knowledge（约 46 条）
└── progress_ws.go   # gorilla/websocket upgrade + 快速完成路径
```

### D2. 关键接口签名

```go
// internal/knowledge/registry.go
type Registry struct{ path string; ... } // data/knowledge_bases/kb_config.json

func (r *Registry) Load(ctx context.Context) (*KBConfig, error)        // flock 共享锁读
func (r *Registry) Mutate(ctx context.Context, fn func(*KBConfig) error) error // flock 排他锁 + 原子写
func (r *Registry) AutoRegisterFromDisk(ctx context.Context) error

// internal/knowledge/manager.go
type Manager struct {
    reg     *Registry
    sidecar sidecar.Client   // typed 契约：RAGIndex / RAGSearch / ParseDocument
    prog    *ProgressTracker
}
func (m *Manager) Create(ctx context.Context, req CreateReq) (taskID string, err error)   // 登记 initializing → raw/ 落盘 → 异步索引
func (m *Manager) Upload(ctx context.Context, kb string, files []Upload) (taskID string, err error)
func (m *Manager) Reindex(ctx context.Context, kb string) (taskID string, err error)      // 按 rag_provider 路由，不拉回默认 pipeline
func (m *Manager) Delete(ctx context.Context, kb string) error                            // 连接型只解注册
func (m *Manager) ResolveKBName(ctx context.Context, name string, u auth.UserContext) (string, error) // 别名 + 归属（rag 工具复用）

// internal/knowledge/reconcile.go
func Reconcile(cfg *KBConfig, kbRoot string, active EmbeddingSignature) // 加载注册表时调用；连接型跳过

// internal/knowledge/progress.go
type ProgressTracker struct{ ... }
func (p *ProgressTracker) Update(kb string, frame ProgressFrame)          // 持久化 + 广播
func (p *ProgressTracker) Subscribe(kb string) (<-chan ProgressFrame, func())
func (p *ProgressTracker) Snapshot(kb string) (*ProgressFrame, bool)
```

### D3. progress WS 快速完成路径（progress_ws.go）

gin handler 内 `websocket.Upgrader.Upgrade` 后：

1. 鉴权失败 → close（policy violation）。
2. 读 `ProgressTracker.Snapshot(kb)`：progress 缺失 / 终态 / KB 已 ready → **快速路径**：写一帧当前状态（终态帧），随即发送 close frame 主动关闭——避免前端无限轮询挂空连接。
3. 否则 `Subscribe(kb)` 进入推送循环：广播帧逐条下发，任务到终态后发终态帧并关闭；客户端断开时取消订阅。
广播器为进程内 per-KB fanout（无外部队列），与基线 broadcaster 语义一致。

### D4. 索引任务编排与 SSE 日志

长任务（create/upload/reindex）由 manager 启动后台 goroutine：`raw/` 受支持文件集合 → `sidecar.RAGIndex`（带 provider 与 version 目标）→ Sidecar 回报的阶段进度经 `ProgressTracker.Update` 持久化并广播 → 完成后 `Mutate` 更新状态/`index_versions`。任务日志入 per-task ring buffer，`GET /tasks/{task_id}/stream` 以 SSE 输出。Sidecar 不可用 → 任务立即失败并把明确错误写入 progress（acceptance.md 3.4 Sidecar 降级）。

### D5. embedding reconcile 算法（reconcile.go）

对每个自管 KB：枚举磁盘 `version-N/`（含 legacy 布局适配）读取各版本 embedding signature →

- 有 ready 版本匹配当前 signature → 清 `needs_reindex`/`embedding_mismatch`，激活该版本；
- 有 ready 版本但均不匹配，或存储 `embedding_model`/`embedding_dim` 与当前不符 → 置两标记（UI Re-index CTA）；
- 全新 KB（无 ready 版本且无历史 embedding 记录）→ 不误标；
- 连接型 KB（Obsidian/linked/LightRAG server/subagent 标记）→ 跳过。
  随后刷新 `index_versions` 快照写回注册表。

### D6. 与 Python 基线文件映射

| Python 基线 | Go 落位 |
| --- | --- |
| `knowledge/kb_types.py` / `naming.py` | `internal/knowledge/{types,naming}.go` |
| `knowledge/manager.py` / `initializer.py` / `add_documents.py` | `internal/knowledge/manager.go` |
| `knowledge/progress_tracker.py` | `internal/knowledge/progress.go` |
| `multi_user/knowledge_access.py` | `internal/knowledge/access.go` |
| `api/routers/knowledge.py`（约 46 路由 + progress WS） | `internal/api/knowledgeapi/{router,progress_ws}.go` |
| `services/rag/service.py`（pipeline 本体） | 不迁移——Sidecar（`internal/sidecar` typed client 调用） |

### D7. 备选方案取舍

- **注册表并发控制**：备选「进程内锁即可」——被否：基线约定文件锁以兼容多进程（CLI 与 server 并存、回滚期 Python 并存）；采用 `flock`（`golang.org/x/sys/unix.Flock`）+ 原子写双保险。
- **progress 推送**：备选「前端轮询 GET /progress」——保留查询端点，但 WS 推送是基线前端已消费的协议面（legacy WS 对等，acceptance.md 3.1），必须保留；快速完成路径按 spec 强制。
- **任务模型**：备选引入持久化任务队列——被否（禁止消息队列）；进程内 goroutine + progress 持久化足以恢复 UI 状态，进程重启后残留 `initializing` 由 `progress/clear` 与 retry 兜底。

## Risks / Trade-offs

- [46 条路由字段对等工作量大] → 以 OpenAPI golden spec 驱动 contract test 逐条锁定，pipeline 参数 schema 不在 spec 重复枚举（以 golden spec 为准）。
- [Go/Python 同时写 `kb_config.json`（回滚演练期）] → flock + 原子写与基线一致；reconcile 只在加载时执行，避免写放大。
- [Sidecar 长任务中途崩溃导致 KB 卡 `initializing`] → progress 带 task_id 与时间戳，`POST /{kb_name}/progress/clear` + `POST /{kb_name}/retry` 组合恢复（spec 已约定）。
- [连接型 KB 的允许列表校验不足导致任意目录挂载] → `probe-folder`/`connect-folder` 双阶段 + 允许列表校验；路径 EvalSymlinks 后再校验。
