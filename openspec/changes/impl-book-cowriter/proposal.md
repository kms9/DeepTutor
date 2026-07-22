# impl-book-cowriter — Proposal

## Why

deeptutor-go 需要交付 BookEngine（交互式教材：三段式创作、后台编译、块级编辑、阅读/学习进度）与 Co-Writer（协同写作：整文/选区编辑、SSE 流式、自动标注、多文档管理）的 Go 实现。行为事实源为 `docs/golang-req/openspec/specs/book-cowriter/spec.md`；按 `docs/golang-req/openspec/ROADMAP.md` 定位属 Wave 3（book-cowriter），spec 标注里程碑 M5，acceptance M3 矩阵已含「Book / Co-Writer：book WS 事件流对等 + 前端页面可用」验收行。spec 已冻结的关键契约：**两者均不是 Level 2 capability**——不经 `emit_capability_result()`，以自有 REST/WS/SSE 面对外，内部流事件仍走 StreamBus（`source="book"` / `source="co_writer_react_edit"`）。

## What Changes

- 新增 `internal/book`：BookEngine——三段式创作（create_book / confirm_proposal / confirm_spine，`BookStatus` 状态机 DRAFT → SPINE_READY → COMPILING → READY）、per-book 编译队列与串行 worker（goroutine 实现，有意差异已写入 spec）、块级编辑五操作、学习交互（deep-dive / quiz-attempt / supplement / page-chat-session）、阅读进度持久化。
- 新增 `/api/v1/book` REST 面（22 端点）与 `WS /api/v1/book/ws`（长连接多任务：每条客户端消息新建一个 StreamBus，事件实时转发 + `*_result` 终态消息，错误不断连）。
- 新增 `internal/cowriter`：EditAgent（/edit 整文、/edit_react 选区 + 可选 RAG/Web 检索降级、/automark）、SSE 流式端点（/edit_react/stream：`event: stream` 帧 + `result`/`error` 终态）、操作历史与 tool_calls 查询、多文档 CRUD（`doc_id` 经 `^[0-9a-f]{8,32}$` 正则校验防路径遍历）。
- per-user 数据目录落盘（原子写），布局与基线兼容（前端零改动、老数据免迁移）。

## Capabilities

### New Capabilities
- `book-cowriter`: BookEngine（三段式创作 + 编译队列 + 块编辑 + 学习交互 + 22 REST + book WS）与 Co-Writer（编辑/SSE/历史/文档 CRUD），非 capability 形态，行为对等基线 v1.5.2。

### Modified Capabilities
（无）

## Impact

- 依赖的其他 change（按 ROADMAP 依赖图）：
  - `impl-llm-provider`（Ideation/SpineSynthesizer/页编译/EditAgent 的 eino ChatModel 调用）
  - `impl-foundation-stream`（StreamBus——book WS 每消息新建 bus、co_writer SSE 内部 bus）
  - `impl-api-unified-ws`（仅复用 HTTP/WS 鉴权中间件；book WS 与 SSE 为本 change 自建端点）
  - `impl-sidecar-contract`（SourceExplorer 取材与 edit_react 的 RAG 检索）
  - `impl-turn-runtime` / `impl-runtime-registry`（间接：page-chat-session 绑定 chat session；不经 orchestrator 路由）
  - `impl-foundation-config`（per-user 数据目录、storage layout）
- 受影响面：前端 Book 页与 Co-Writer 页全部功能；`data/` 下 book 与 co_writer 目录（与基线互操作，回滚可用）。
