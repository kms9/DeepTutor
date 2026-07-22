# book-cowriter Specification

## Purpose
本模块定义 Go 版 BookEngine（交互式教材：三段式创作、后台编译、块级编辑、阅读/学习进度）与 Co-Writer（协同写作：文本编辑、自动标注、文档管理）的目标行为。两者均不是 Level 2 capability：不经 `emit_capability_result()` 输出 capability envelope，而是以自有 REST/WS/SSE 面对外，内部流事件仍走 StreamBus（`source="book"` / `source="co_writer_react_edit"`）。REST 前缀分别为 `/api/v1/book` 与 `/api/v1/co_writer`。

- 参考实现（基线）：`deeptutor/book/`（`engine.py`、`streaming.py`、`models.py`、`storage.py`、agents）、`deeptutor/co_writer/`（`edit_agent.py`、`storage.py`）、`deeptutor/api/routers/book.py`、`deeptutor/api/routers/co_writer.py`
- 依赖 spec / 里程碑：依赖 llm-provider、stream-bus、api-unified-ws（鉴权）、storage-layout、sidecar-contract（RAG 检索）；里程碑：M5

## Requirements

### Requirement: BookEngine 三段式创作流程
系统 SHALL 实现三段式教材创作，状态机沿 `BookStatus` 推进：
1. `create_book`（Stage 1，Ideation）：输入 `user_intent`（必填）+ 可选 `chat_session_id`/`chat_selections`/`notebook_refs`/`knowledge_bases`/`question_categories`/`question_entries`/`language`，运行 IdeationAgent 产出 `BookProposal`，book 落盘为 `DRAFT`；
2. `confirm_proposal`（Stage 2，Spine）：接受用户编辑后的完整 proposal（可选），运行 SpineSynthesizer（draft / critique / revise 多轮，逐轮经 stage 事件可见）产出 `Spine`，并注入引擎生成的 Overview 章节，book 置 `SPINE_READY`；
3. `confirm_spine`（Stage 3）：接受编辑后的 spine（可选），为每章创建 `PENDING` page shell；Overview 页 SHALL 无 LLM、不进队列地立即物化为 `READY`（intro 文本 + 章节索引）；`auto_compile=true`（默认）时 book 置 `COMPILING` 并把其余 pending 页入编译队列，否则停在 `SPINE_READY`。

#### Scenario: 完整创作一本书
- **WHEN** 用户依次调用 create → confirm-proposal → confirm-spine（auto_compile 默认开）
- **THEN** 返回 proposal、spine、page shell 列表，Overview 页即时 READY，其余页进入后台编译，book 状态依次为 DRAFT → SPINE_READY → COMPILING

### Requirement: 后台编译队列与 per-book worker
系统 SHALL 为每本书维护运行时（编译队列 + 去重集合 + 单 worker 协程等价物）：入队 SHALL 跳过 `READY` 页与已在 `queued` 集合中的页；worker 逐页出队编译，队列空闲超时后自行退出；页编译异常 SHALL 把 `GENERATING`/`PLANNING` 状态的页标记为 `ERROR` 并继续后续页。全部页 `READY` 后 SHALL 把 book 置为 `READY`（finalize）。`compile_page`（用户翻到某页时调用）SHALL 提供当前页优先语义：页已 `READY` 且非 `force` 直接返回；`force=true` 重置为 `PENDING` 后同步编译。`rebuild_book` SHALL 清空该书队列与去重集合、重置所有非 Overview 页为 `PENDING`、book 置 `SPINE_READY`，`auto_compile=true` 时重新入队。

#### Scenario: 读者跳读未编译页
- **WHEN** 某页仍在队列中而用户直接打开它并调用 `POST /books/compile-page`
- **THEN** 该页被同步编译并返回 READY 页对象；后台 worker 再次遇到该页时因已 READY 而跳过

### Requirement: Book stage 事件与 book_event 语义
BookEngine 的流事件 SHALL 以 `source="book"` 发出，stage 名固定为：`ideation`、`exploration`（SourceExplorer 取材）、`synthesis`（spine 起草/修订）、`critique`（spine 评审轮）、`overview`、`spine`（spine 阶段外层包裹）、`page_plan`、`compilation`、`block`、`interaction`。每个阶段 SHALL 经 StreamBus stage 上下文自动发出 `stage_start`/`stage_end`，过程状态经 progress/thinking 事件、错误经 error 事件呈现；书籍特有的结构化通知 SHALL 经 `book_event`（携带 `kind` 与实体 payload 的定制事件）发出，供前端做增量 UI 更新。

#### Scenario: spine 评审轮可见
- **WHEN** confirm_proposal 触发 SpineSynthesizer 的 critique 轮
- **THEN** 流上出现 `critique` 阶段的 stage_start/stage_end 与逐轮 progress，前端可展示评审过程

### Requirement: 块级编辑操作
系统 SHALL 支持对 READY 页的块级操作，均持久化并返回受影响实体：`regenerate_block`（可带 `params_override`，重新生成单块内容）、`insert_block`（`block_type` 必须是合法 `BlockType` 枚举值否则 400；可指定 `position`；`compile_now=true` 时立即生成内容）、`delete_block`、`move_block`（`new_position` 重排）、`change_block_type`（换类型可带 `params_override` 并重生成）。目标不存在 SHALL 返回 404。块生成过程的流事件 SHALL 落在 `block` 阶段。

#### Scenario: 插入未知类型块
- **WHEN** `POST /books/insert-block` 携带 `block_type="hologram"`
- **THEN** 返回 400 `Unknown block type: hologram`，页面不变

### Requirement: 学习交互与阅读进度
系统 SHALL 支持：`create_deep_dive_subpage`——从某页的 topic（可关联 block）派生 `content_type`（合法 `ContentType` 枚举，默认 `concept`）的子页（初始 `PENDING`，走编译流程）；`record_quiz_attempt`——记录页内测验块的作答（`question_id`/`user_answer`/`is_correct`）并更新返回阅读进度对象；`supplement_for_weakness`——针对薄弱 topic 在页内追加补救块；`set_page_chat_session`——把页与 chat session 绑定（页内追问跳转 chat 时复用）。阅读进度 SHALL 随 book 持久化并在 `GET /books/{book_id}` 中随 book/spine/pages 一并返回。

#### Scenario: 答错后请求补救
- **WHEN** 学生答错 quiz 块并前端随后调用 `POST /books/supplement`
- **THEN** quiz-attempt 更新进度中的作答记录，supplement 在该页追加一个补救块并返回

### Requirement: /api/v1/book REST 面
系统 SHALL 提供以下 22 个 REST 端点（HTTP 鉴权，路径相对 `/api/v1/book`），响应 JSON 字段与基线一致（实体以 JSON 序列化全量返回）：
- 读取组：`GET /health`、`GET /books`、`GET /books/{book_id}`（book+spine+pages+progress）、`GET /books/{book_id}/spine`、`GET /books/{book_id}/pages/{page_id}`、`DELETE /books/{book_id}`；
- 创作组：`POST /books`（空 `user_intent` → 400）、`POST /books/confirm-proposal`（非法 proposal → 400；book 不存在 → 404）、`POST /books/confirm-spine`、`POST /books/compile-page`、`POST /books/rebuild`；
- 块编辑组：`POST /books/regenerate-block`、`POST /books/insert-block`、`POST /books/delete-block`、`POST /books/move-block`、`POST /books/change-block-type`；
- 学习组：`POST /books/deep-dive`、`POST /books/quiz-attempt`、`POST /books/supplement`、`POST /books/page-chat-session`；
- 维护组：`GET /books/{book_id}/health`（返回 `kb_drift` 报告 + `log_health`）、`POST /books/{book_id}/refresh-fingerprints`（刷新 KB 指纹，book 不存在 → 404）。
引擎内部错误 SHALL 统一映射 500（detail 为错误消息），`ValueError` 类查找失败映射 404。

#### Scenario: 获取整书视图
- **WHEN** `GET /api/v1/book/books/{book_id}` 且书存在
- **THEN** 返回 `{book, spine, pages, progress}` 四键；书不存在时 404

### Requirement: /api/v1/book/ws 流式协议
系统 SHALL 提供 `WS /api/v1/book/ws`（WS 层鉴权，失败即关闭）：长连接内按消息驱动，每条客户端消息 SHALL 新建一个 StreamBus 并把 `source="book"` 的事件序列化为 `{type, source, stage, content, metadata}` 实时转发。支持的消息类型与响应：`create` → 流事件 + `{type: "create_result", book, proposal}`；`confirm_proposal` → `confirm_proposal_result`（book+spine）；`confirm_spine` → `confirm_spine_result`（pages）；`compile_page`（可带 `force`）→ `compile_page_result`（page）；`regenerate_block` → `regenerate_block_result`（block 或 null）。缺失 `type`/未知 type/JSON 解析失败/执行异常 SHALL 回 `{type: "error", content}` 且连接保持；每条消息处理完 SHALL 关闭其 bus 并取消转发任务。

#### Scenario: WS 内创建并逐步编译
- **WHEN** 客户端在同一连接上依次发送 create、confirm_proposal、confirm_spine
- **THEN** 每步先收到对应阶段的流事件（ideation/synthesis/critique/spine 等），再收到对应 `*_result` 消息；中途某步失败只收到 error，连接可继续下一条消息

### Requirement: Co-Writer 编辑操作（/edit、/edit_react、/automark）
系统 SHALL 提供（HTTP 鉴权，路径相对 `/api/v1/co_writer`）：
- `POST /edit`：`{text(≤600k 字符), instruction(≤10k), action ∈ {rewrite, shorten, expand}, source ∈ {rag, web}|null, kb_name}` → EditAgent 整文编辑，返回 `{edited_text, operation_id}`；
- `POST /edit_react`：`{selected_text(≤120k), instruction, mode ∈ {rewrite, shorten, expand, none}, tools ⊆ {rag, web}, kb_name}` → 选区编辑，返回 `{edited_text, operation_id, tools_used}`。校验：`mode="none"` 且无 instruction → 400（i18n 文案）；选区为空 → 400。instruction 为空时 SHALL 按 mode 注入语言相关默认指令；`operation_id` 格式为 `YYYYMMDD_HHMMSS_<hex6>`；
- `POST /automark`：`{text}` → 自动标注，返回 `{marked_text, operation_id}`。
edit_react SHALL 先做可选检索（逐工具：`rag` 无 `kb_name` 时跳过；检索失败 SHALL 降级为纯编辑绝不阻塞）再拼装编辑 prompt（选区包 markdown 围栏 + 参考资料块），产出 SHALL 剥离 thinking 标签与外层代码围栏。每次操作 SHALL 追加历史记录（含 action/mode/tools/输入输出/model）。语言 SHALL 取 UI 设置回退 main 配置；EditAgent SHALL 为按语言缓存的单例并在每次请求刷新 LLM 配置。

#### Scenario: 带 RAG 的选区改写
- **WHEN** `POST /edit_react` 携带 `tools=["rag"]`、`kb_name="my-kb"`、`mode="rewrite"`
- **THEN** 先对 KB 检索（以 instruction 或选区前 400 字符为 query），检索结果融入改写，响应 `tools_used=["rag"]`

### Requirement: Co-Writer 流式编辑 SSE（/edit_react/stream）
`POST /edit_react/stream` SHALL 先做与 `/edit_react` 相同的请求校验（失败即 4xx），然后返回 `text/event-stream`（`Cache-Control: no-cache`、`X-Accel-Buffering: no`）：编辑过程经内部 StreamBus 转发为 `event: stream` 帧（data 为事件 JSON，含检索的 tool_call/tool_result（stage=`exploring`）与逐 chunk 的 content（stage=`responding`，前后有 `responding` 的 stage_start/stage_end））；结束后 SHALL 追加一帧终态——成功为 `event: result`（data 为 `{edited_text, operation_id, tools_used}`），失败为 `event: error`（data 为 `{detail}`）。

#### Scenario: SSE 流式选区编辑
- **WHEN** 前端以 stream 端点发起 expand 编辑
- **THEN** 依次收到 stream 帧（工具轨迹 + 增量文本），最后一帧为 result；LLM 中途失败时最后一帧为 error

### Requirement: Co-Writer 历史与工具调用查询
系统 SHALL 提供：`GET /history` → `{history: [...], total}`（全部操作记录）；`GET /history/{operation_id}` → 单条记录，未找到 404；`GET /tool_calls/{operation_id}` → 按 `<operation_id>_*.json` 文件名模式查找该操作落盘的工具调用详情，未找到 404。

#### Scenario: 回看某次编辑的检索详情
- **WHEN** `GET /tool_calls/{operation_id}` 且该操作曾执行 RAG 检索
- **THEN** 返回落盘的工具调用 JSON（query、结果全文）

### Requirement: Co-Writer 文档 CRUD 与 doc_id 安全校验
系统 SHALL 提供多文档管理：`GET /documents` → `{documents: [{id, title, created_at, updated_at, preview}]}`（按最近更新排序的摘要视图）；`POST /documents`（`{title?, content}`）→ 完整文档 `{id, title, content, created_at, updated_at}`；`GET /documents/{doc_id}`；`PUT /documents/{doc_id}`（title/content 可单独更新）；`DELETE /documents/{doc_id}` → `{deleted: bool}`。`doc_id` SHALL 先经 `^[0-9a-f]{8,32}$` 正则校验，不匹配一律按 404 处理（防止路径遍历触达 documents 根目录外，删除为递归删除必须严格校验）；文档不存在返回 404。

#### Scenario: 非法 doc_id 被拒
- **WHEN** `DELETE /documents/a%2F..%2F..%2Fx`
- **THEN** 返回 404 `Document not found`，不发生任何文件系统操作

### Requirement: 持久化与多用户隔离
Book 与 Co-Writer 数据 SHALL 落在 per-user 数据目录下（book：book/spine/pages/progress JSON；co_writer：documents/、历史与 tool_calls 文件），写入 SHALL 原子化（临时文件 + rename 等价），Go 版目录布局遵循 storage-layout spec 与基线兼容（前端零改动）。（有意差异）Python 基线的 book worker 依赖进程内 asyncio 单例，Go 版 SHALL 以 goroutine + 每书串行队列实现等价语义：同一本书的页编译不并发、不同书互不阻塞、进程重启后队列不恢复（与基线一致，靠 compile_page/rebuild 补偿）。

#### Scenario: 进程重启后的恢复路径
- **WHEN** 服务在 book 处于 COMPILING 时重启
- **THEN** 队列不自动恢复；用户打开某页触发 `compile-page` 或调用 `rebuild` 后继续编译，已 READY 页不受影响
