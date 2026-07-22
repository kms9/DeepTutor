# impl-book-cowriter — Design

## Context

BookEngine 与 Co-Writer 是两个**非 capability** 域：不进 CapabilityRegistry、不经 `emit_capability_result()`，以自有 REST/WS/SSE 面对外；内部流事件仍走 StreamBus（`source="book"` / `source="co_writer_react_edit"`），由各自端点把事件序列化转发给前端。事实源：`docs/golang-req/openspec/specs/book-cowriter/spec.md`。有意差异（已写入 spec）：基线 book worker 依赖进程内 asyncio 单例，Go 版以 goroutine + 每书串行队列实现等价语义（同书不并发、异书不阻塞、重启不恢复队列）。安全关键点：co_writer `doc_id` 删除为递归删除，必须先过 `^[0-9a-f]{8,32}$` 正则（不匹配一律 404）。

## Goals / Non-Goals

**Goals:**
- 三段式创作、编译队列、块编辑、学习交互、22 REST + book WS 对等基线；Co-Writer 三编辑端点 + SSE + 历史 + 文档 CRUD 对等。
- book WS「长连接多任务：每条消息新建 StreamBus」语义精确复刻；SSE 帧格式（stream/result/error）对等。
- per-user 目录布局与基线互操作（B 类数据兼容 + 回滚）。

**Non-Goals:**
- 不做 capability 化改造；不实现 RAG 检索本体（Sidecar）；不改前端；不实现队列持久化/重启恢复（与基线一致靠 compile_page/rebuild 补偿）。

## Decisions

### D1. Go 包结构

```
internal/book/
├── engine.go       # BookEngine：create_book / confirm_proposal / confirm_spine / compile_page /
│                   #   rebuild_book / 块编辑五操作 / 学习交互四操作 / finalize
├── runtime.go      # per-book 运行时：编译队列 + queued 去重集合 + 串行 worker goroutine
├── agents.go       # IdeationAgent / SourceExplorer / SpineSynthesizer(draft-critique-revise) / 页编译 agents
├── streaming.go    # stage 事件 helper + book_event 定制事件
├── models.go       # Book/BookStatus/Proposal/Spine/Page/PageStatus/BlockType/ContentType/进度
├── storage.go      # per-user book 目录 JSON 原子写
└── prompts/        # {en,zh}/（基线原样拷贝）
internal/cowriter/
├── edit_agent.go   # EditAgent：整文/选区编辑、automark、检索融合、thinking/围栏剥离
├── storage.go      # documents/ CRUD（doc_id 守卫）、历史记录、tool_calls 落盘
└── prompts/        # {en,zh}/（基线原样拷贝）
internal/api/routers/
├── book.go         # /api/v1/book 22 REST 端点（gin group）
├── book_ws.go      # WS /api/v1/book/ws（gorilla upgrade，多任务长连接）
└── co_writer.go    # /api/v1/co_writer REST + /edit_react/stream SSE
```

### D2. 关键接口签名

```go
type BookEngine struct { store *Storage; runtimes sync.Map /* bookID → *bookRuntime */; llm ...; sidecar ... }
func (e *BookEngine) CreateBook(ctx context.Context, bus *core.StreamBus, req CreateBookRequest) (*Book, *BookProposal, error)
func (e *BookEngine) ConfirmProposal(ctx context.Context, bus *core.StreamBus, bookID string, proposal *BookProposal) (*Book, *Spine, error)
func (e *BookEngine) ConfirmSpine(ctx context.Context, bus *core.StreamBus, bookID string, spine *Spine, autoCompile bool) ([]*Page, error)
func (e *BookEngine) CompilePage(ctx context.Context, bus *core.StreamBus, bookID, pageID string, force bool) (*Page, error)
func (e *BookEngine) RebuildBook(ctx context.Context, bookID string, autoCompile bool) error
// 块编辑：RegenerateBlock / InsertBlock / DeleteBlock / MoveBlock / ChangeBlockType
// 学习：CreateDeepDiveSubpage / RecordQuizAttempt / SupplementForWeakness / SetPageChatSession

type bookRuntime struct {
    queue  chan string          // pageID，串行消费
    queued map[string]struct{}  // 去重集合（互斥保护）
    // worker goroutine：逐页出队编译，空闲超时退出；页异常 → GENERATING/PLANNING 置 ERROR 继续
}

type EditAgent struct{ lang string; llm ... } // 按语言缓存单例，每请求刷新 LLM 配置
func (a *EditAgent) Edit(ctx, req EditRequest) (edited string, opID string, err error)
func (a *EditAgent) EditReact(ctx, bus *core.StreamBus, req EditReactRequest) (edited string, opID string, toolsUsed []string, err error)
func (a *EditAgent) Automark(ctx, text string) (marked string, opID string, err error)
```

### D3. 编排落位（eino Graph vs 自定义）——取舍结论

**结论：自定义函数编排，不用 eino Graph。** 论证：(1) BookEngine 的「编排」核心不是 LLM pipeline 而是**持久状态机 + 后台队列**（BookStatus/PageStatus 跨请求、跨进程生命周期推进），Graph 是进程内单次执行模型，与此正交；(2) SpineSynthesizer 的 draft → critique → revise 多轮是唯一的多步 LLM 流程，轮数动态（评审通过即止），一个 `for` 循环 + stage 事件即可；(3) 与其余五个 capability change 的 D3 结论一致，全项目统一「eino 只管 ChatModel 调用」。Co-Writer 的 edit_react 是「可选检索 → 单次编辑调用」两步，同样无编排框架必要。

### D4. stage 事件与 book_event 发射点

- 固定 stage 名：`ideation`、`exploration`、`synthesis`、`critique`、`overview`、`spine`、`page_plan`、`compilation`、`block`、`interaction`——均经 `bus.Stage(name, source="book")` 自动 stage_start/stage_end；过程用 progress/thinking，错误用 error。
- 结构化通知经 `book_event` 定制事件（`kind` + 实体 payload），供前端增量 UI 更新。
- 块生成流事件落 `block` 阶段；co_writer SSE 内部事件：检索 tool_call/tool_result 在 `exploring`，逐 chunk content 在 `responding`（前后有该 stage 的 start/end），source=`co_writer_react_edit`。

### D5. 「每消息新建 StreamBus 的多任务长连接」（book WS）

`WS /api/v1/book/ws`：WS 层鉴权失败即关闭。连接内循环读消息，**每条消息**：
1. 解析 JSON 取 `type`（缺失/未知/解析失败 → `{type:"error", content}`，**连接保持**）；
2. `bus := core.NewStreamBus()`；启动转发 goroutine 把 `source="book"` 事件序列化为 `{type, source, stage, content, metadata}` 实时写 WS（单写 goroutine 汇聚，规避 gorilla 并发写）；
3. 调 engine 对应方法（create/confirm_proposal/confirm_spine/compile_page/regenerate_block）；
4. 成功写 `{type:"<msg>_result", ...实体}`；异常写 error；
5. **finally：关闭该 bus 并 cancel 转发 goroutine**——bus 生命周期严格 = 单条消息处理期，杜绝跨消息事件串扰。
后台编译（confirm_spine 入队部分）不绑定该 bus——worker 用独立 bus/无 bus 静默编译，与基线一致（前端靠 compile-page/GET 轮询取页）。

### D6. doc_id 正则安全边界

`storage.go` 所有 documents 路径操作前置守卫：`var docIDRe = regexp.MustCompile(`^[0-9a-f]{8,32}$`)`；不匹配一律返回 404 "Document not found"（不泄露校验存在），**不触任何文件系统调用**。删除实现为 `os.RemoveAll(filepath.Join(docRoot, docID))`——因是递归删除，守卫必须在 join 之前；另加防御性断言：join 结果经 `filepath.Clean` 后必须仍以 docRoot 为前缀。URL 解码由 gin 完成，正则作用于解码后值（`a%2F..%2F..%2Fx` 解码含 `/` 必不匹配）。

### D7. 存储与并发

- book：book/spine/pages/progress JSON per-user 目录，临时文件 + rename 原子写；worker 串行保证同书页编译不并发，跨书 `sync.Map` 各自 runtime 互不阻塞；重启不恢复队列（有意同构）。
- co_writer：documents/、历史（append）、`<operation_id>_*.json` tool_calls；`operation_id` 格式 `YYYYMMDD_HHMMSS_<hex6>`。
- edit_react 检索：逐工具（rag 无 kb_name 跳过），检索失败降级纯编辑绝不阻塞；产物剥离 thinking 标签与外层代码围栏。

### D8. 与 Python 基线文件映射表

| Python 基线 | Go 落位 |
| --- | --- |
| `deeptutor/book/engine.py` | `internal/book/engine.go` + `runtime.go` |
| `deeptutor/book/streaming.py` | `internal/book/streaming.go` |
| `deeptutor/book/models.py` | `internal/book/models.go` |
| `deeptutor/book/storage.py` | `internal/book/storage.go` |
| `deeptutor/book/agents/*` | `internal/book/agents.go`（+ prompts 原样拷贝） |
| `deeptutor/api/routers/book.py` | `internal/api/routers/book.go` + `book_ws.go` |
| `deeptutor/co_writer/edit_agent.py` | `internal/cowriter/edit_agent.go` |
| `deeptutor/co_writer/storage.py` | `internal/cowriter/storage.go` |
| `deeptutor/api/routers/co_writer.py` | `internal/api/routers/co_writer.go` |

## Risks / Trade-offs

- [goroutine worker 与基线 asyncio 单例的空闲退出/重入时序差异] → 语义锚定 spec 三条不变量（同书串行、异书并行、重启不恢复），用竞态测试（`-race` + 并发 compile/rebuild）验证，不逐行对齐实现。
- [book WS 转发 goroutine 泄漏（客户端断连时消息处理中）] → context 取消贯穿 engine 调用；连接关闭统一 cancel 全部在处理消息；`goleak` 断言。
- [递归删除误伤] → D6 双重守卫 + 单测覆盖遍历攻击样例（spec Scenario）；评审清单专项。
- [SSE 在代理后被缓冲] → 按 spec 设 `Cache-Control: no-cache`、`X-Accel-Buffering: no`，gin flush 每帧。
- [22 端点字段对等工作量大] → 实体 JSON 序列化字段名以 golden OpenAPI spec contract test 兜底，不手工核对。

## Migration Plan

无数据迁移（目录布局兼容，基线数据直接可读；Go 写入后可回滚 Python——B 类验收）。实现顺序：models/storage → engine 同步路径（create/confirm×2）→ 队列 worker → 块编辑/学习交互 → REST → book WS → co_writer（可与 book 并行）。回滚 = 不挂两组路由。

## Open Questions

- `GET /books/{book_id}/health` 的 `kb_drift` 报告字段结构以基线响应录制为准（contract test 时冻结）。
