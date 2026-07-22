# Design: impl-sidecar-contract

## Context

基线把 RAG（`deeptutor/services/rag/service.py`、`factory.py`）、文档解析（`deeptutor/services/parsing/`、附件轻量抽取 `deeptutor/utils/document_extractor.py`）、Manim 渲染（`deeptutor/agents/math_animator/renderer.py`）、沙箱执行（`deeptutor/services/sandbox/`）都跑在同一 Python 进程内，以 Python 对象直调。Go 重写后这些域保留在 Python（生态绑定：llama-index/MinerU/manim/沙箱运行时），进程边界上升为网络契约。核心风险是「无类型 map 直传」造成的隐性契约漂移，因此 spec 把 schema 强校验、未知字段拒绝、稳定错误码定为总则；同时 Sidecar 属可降级依赖——不可用时 chat 主链路必须无感（acceptance.md §3.4「Sidecar 降级」）。契约冻结于 `contracts/sidecar/`（acceptance.md §5），变更需评审（ROADMAP §5 并行约束）。

## Goals / Non-Goals

**Goals:**

- 五个业务端点（`RAGSearch`/`RAGIndex`/`ParseDocument` 两档/`RenderManim`/`ExecCode`）+ 健康检查 + 进度回报的 typed 契约与双边校验。
- Go typed client（按端点超时、受理+进度模式、降级快速失败）与 progress WS 转发驱动。
- Python 侧 FastAPI 薄壳：复用基线四个域的代码、不修改基线仓库。
- 错误归一为六类稳定错误码。

**Non-Goals:**

- RAG pipeline、解析引擎、Manim、沙箱 backend 的内部行为变更（薄壳原样复用基线实现）。
- KB CRUD/文件管理与 `/api/v1/knowledge` 完整 REST 面（归 knowledge 模块；本 change 只交付 progress WS 的转发驱动与契约）。
- `rag`/`exec`/`code_execution` 工具封装（归 tools-builtin）；math_animator 编排（归 capability-visualize）。

## Decisions

### D1. 契约形式：OpenAPI 3.1 + 代码生成

- 传输选 HTTP/JSON + OpenAPI（而非 protobuf/gRPC）。理由：Sidecar 端 FastAPI + pydantic 原生产出 OpenAPI、校验免费获得；单机 localhost 通信性能无 gRPC 必要；acceptance.md §5 亦以 OpenAPI 为建议形式。
- 契约冻结件：`deeptutor-go/contracts/sidecar/openapi.json`（由 sidecar FastAPI 导出后评审入库）+ `contracts/sidecar/fixtures/*.json`（每端点请求/响应 golden fixtures，字段级对等依据）。
- Go client 用 `oapi-codegen` 从冻结的 `openapi.json` 生成 types + client（`internal/sidecar/gen/`），生成物入库、CI 校验「重新生成无 diff」防契约漂移；手写薄封装在 `internal/sidecar/client.go`。

### D2. 端点与长任务模式

| 端点 | 方法/路径 | 模式 | Go 超时 |
| --- | --- | --- | --- |
| 健康检查 | `GET /health` | 同步 | 短（~2s） |
| RAGSearch | `POST /rag/search` | 同步 | 短（默认 ~30s，可配） |
| RAGIndex | `POST /rag/index` | 受理返回 `{task_id}` | 受理短超时 |
| 进度流 | `GET /progress/stream?kb_name&task_id`（SSE） | 服务端推送 | 空闲超时 + 终态关闭 |
| ParseDocument(full) | `POST /parse/document` | 受理 + 进度/轮询 | 受理短超时 |
| ParseExtract(light) | `POST /parse/extract` | 同步（bytes-in/text-out） | 短 |
| RenderManim | `POST /render/manim` | 受理 + 进度（stdout/stderr 逐行帧） | 受理短超时 |
| ExecCode | `POST /exec` | 同步（自身带 `timeout_s`） | `timeout_s` + 余量 |

进度回报统一走 SSE（每帧即基线 ProgressTracker 形状：`stage/message/percent/current/total/timestamp/task_id`；RenderManim 复用同帧型并区分 raw/summary 层）。选 SSE 而非 WS：单向推送、FastAPI `StreamingResponse` 即可、Go 端 `bufio.Scanner` 消费简单。

### D3. Go 客户端签名与降级

```go
package sidecar

type Client struct {
    base   string
    http   *http.Client
    health atomic.Pointer[HealthStatus] // 最近一次探测缓存
}

func New(baseURL string, opts ...Option) *Client
func (c *Client) Health(ctx context.Context) (*HealthStatus, error)          // 短超时；失败只标记不可用
func (c *Client) Available(capability string) bool                            // "rag"/"parse"/"manim"/"sandbox"
func (c *Client) RAGSearch(ctx context.Context, req RAGSearchRequest) (*RAGSearchResponse, error)
func (c *Client) RAGIndex(ctx context.Context, req RAGIndexRequest) (*TaskAccepted, error)
func (c *Client) StreamProgress(ctx context.Context, kbName, taskID string) (<-chan Progress, error)
func (c *Client) ParseDocument(ctx context.Context, req ParseDocumentRequest) (*TaskAccepted, error)
func (c *Client) ParseExtract(ctx context.Context, req ParseExtractRequest) (*ParseExtractResponse, error)
func (c *Client) RenderManim(ctx context.Context, req RenderManimRequest) (*TaskAccepted, error)
func (c *Client) ExecCode(ctx context.Context, req ExecRequest) (*ExecResult, error)

type APIError struct { Code string; Message string; Details map[string]any } // 六类稳定错误码
var ErrSidecarUnavailable = errors.New("sidecar unavailable")
```

- 降级：连接失败/健康检查失败/超时 → 立即返回 `ErrSidecarUnavailable`（不重试、不排队），由调用方（工具层）转成 StreamEvent `error` 或工具结果的用户可读文案；恢复探测只随健康检查节奏（启动时 + 调用失败后按需 + 周期性短探测），保证 chat 主链路零依赖。
- 错误映射：HTTP 4xx/5xx 响应体统一为 `{"error":{"code","message","details"}}`，client 反序列化为 `APIError`；Go 侧按 `Code` 分支（spec「SHALL NOT 依赖解析 message 文本」）。

### D4. progress WS 转发（Go 侧，gin + gorilla/websocket）

`internal/api/knowledgeprogress.go` 提供 `/api/v1/knowledge/{kb_name}/progress/ws`：gin handler 内 `websocket.Upgrader.Upgrade`，随后 `client.StreamProgress` 的 SSE 帧包成 `{"type":"progress","data":{...}}` 转发。快速路径：连接时先取当前进度快照，若无活跃任务（终态或时间戳老于活跃窗口）且 KB 已就绪 → 立即发一帧 `stage=completed, percent=100` 后关闭；`task_id` 查询参数只透传该任务帧；终态帧发出后结束推送并关闭连接。该 handler 的完整路由注册随 knowledge 模块合入，本 change 交付可独立测试的转发驱动。

### D5. Python 侧薄壳与基线剥离方式

新建 `deeptutor-sidecar` 包（独立 pyproject，依赖 `fastapi` + 基线包）：

```
deeptutor-sidecar/
├── pyproject.toml            # 依赖 deeptutor（基线 v1.5.2，只读引用）+ fastapi/uvicorn
└── sidecar/
    ├── main.py               # FastAPI app、错误归一 exception handlers
    ├── schemas.py            # 全部请求/响应 pydantic 模型（model_config: extra="forbid"）
    ├── routers/
    │   ├── health.py         # 聚合 rag/parsing/manim/sandbox 就绪探测
    │   ├── rag.py            # 包装 services/rag/service.py + factory.py（provider 解析、版本模型）
    │   ├── parse.py          # full 档包装 services/parsing/service.py；light 档包装 utils/document_extractor.py
    │   ├── render.py         # 包装 agents/math_animator/renderer.py
    │   └── exec.py           # 包装 services/sandbox/service.py（ExecRequest/ExecResult 对应 spec.py）
    └── progress.py           # ProgressTracker 桥接 → SSE
```

- 剥离原则：**引用不修改**——sidecar 以 import 方式复用基线 `deeptutor.services.rag`、`deeptutor.services.parsing`、`deeptutor.services.sandbox`、`deeptutor.agents.math_animator.renderer`、`deeptutor.utils.document_extractor`；路由层只做 schema 化（pydantic `extra="forbid"` 落实「未知字段拒绝」）、错误归一（捕获基线 `ParserError`/`ManimRenderError` 等映射到六类错误码）与进度桥接。
- 路径与配置：sidecar 与 Go 主进程共享同一 `data/`（PathService 布局，foundation-config spec）；附件限额每次调用时重读 `system.json`（含 env override），满足「限额随设置即时生效」。
- 禁止 `dict[str, Any]`：schemas.py 全字段显式类型；RAG 结果 `metadata` 等开放结构声明为 `Json`/`dict` 且在 schema 注释标注 opaque + 消费方（前端 sources 面板）。

### D6. 与 Python 基线的映射表

| Python 基线 | 新落位 |
| --- | --- |
| `deeptutor/services/rag/service.py`、`factory.py` | `sidecar/routers/rag.py`（包装）+ Go `internal/sidecar` `RAGSearch`/`RAGIndex` |
| `deeptutor/services/parsing/service.py`、`types.py`（`ParsedDocument` IR、parse_cache） | `sidecar/routers/parse.py` full 档 |
| `deeptutor/utils/document_extractor.py`（附件抽取 + 限额） | `sidecar/routers/parse.py` light 档 |
| `deeptutor/agents/math_animator/renderer.py`（含 YON_IMAGE 锚块协议） | `sidecar/routers/render.py` |
| `deeptutor/services/sandbox/service.py`、`spec.py`（`ExecRequest`/`ExecResult`、隔离级别、配额） | `sidecar/routers/exec.py` |
| `deeptutor/api/routers/knowledge.py` 的 progress WS | Go `internal/api/knowledgeprogress.go`（gin + gorilla/websocket）+ `sidecar/progress.py` SSE |

## Risks / Trade-offs

- [受理 + SSE 进度相比基线进程内回调引入时序缝隙（受理成功但订阅前的帧丢失）] → Sidecar 按 KB/task 保留最近进度快照，`StreamProgress` 连接时先发快照帧再增量推送。
- [oapi-codegen 生成物与手写封装漂移] → 生成物入库 + CI「重新生成零 diff」+ golden fixtures 双向校验（Go 序列化请求 ↔ pydantic 校验；pydantic 序列化响应 ↔ Go 反序列化）。
- [pydantic `extra="forbid"` 使契约演进（新增字段）变为破坏性] → 这是有意收紧（spec 总则）；演进走契约评审 + 版本号（health 返回 version），Go client 与 sidecar 版本不匹配时在健康检查中可见。
- [两进程共享 `data/` 的路径漂移] → sidecar 复用基线 PathService 逻辑、Go 侧路径由 foundation-config 提供，双方以同一 `data/` 根启动参数对齐；契约中传路径的字段（`source_path`、`workdir`）一律为绝对路径。
- [沙箱/解析的重活拖垮 sidecar 事件循环] → FastAPI 侧对阻塞调用走线程池/子进程（沿基线既有实现），Go 侧长任务全部受理制，不做单请求长阻塞。

本模块无 acceptance.md §6 登记编号的有意差异（D-001~D-008 不涉及 sidecar-contract）；「未知字段拒绝、禁止无类型直传」等相对基线的收紧均已固化为本模块 spec 的总则要求。
