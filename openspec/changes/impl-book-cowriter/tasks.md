# impl-book-cowriter — Tasks

## 1. 模型与存储

- [ ] 1.1 实现 `internal/book/models.go`：Book/BookStatus/BookProposal/Spine/Page/PageStatus/BlockType/ContentType/阅读进度（JSON 字段名与基线一致）
- [ ] 1.2 实现 `internal/book/storage.go`：per-user 目录、book/spine/pages/progress JSON 原子写（临时文件 + rename）
- [ ] 1.3 实现 `internal/cowriter/storage.go`：documents CRUD（`^[0-9a-f]{8,32}$` 守卫先于任何路径操作、Clean+前缀防御断言、不匹配 404 无 FS 调用）、历史 append、`<operation_id>_*.json` tool_calls、`operation_id` 生成（`YYYYMMDD_HHMMSS_<hex6>`）
- [ ] 1.4 从基线原样拷贝 book 与 co_writer prompts（{en,zh}），接入 PromptStore

## 2. BookEngine 三段式与编译队列

- [ ] 2.1 实现 create_book（Stage 1）：IdeationAgent 产出 BookProposal、book 落盘 DRAFT、`ideation`/`exploration` stage 事件与 book_event
- [ ] 2.2 实现 confirm_proposal（Stage 2）：SpineSynthesizer draft→critique→revise 多轮（`synthesis`/`critique` stage 逐轮可见）、Overview 章节注入、SPINE_READY
- [ ] 2.3 实现 confirm_spine（Stage 3）：PENDING page shell、Overview 页无 LLM 即时 READY（intro + 章节索引）、auto_compile 默认 true → COMPILING 入队
- [ ] 2.4 实现 per-book 运行时（`runtime.go`）：队列 + queued 去重（READY 页跳过）、串行 worker goroutine（空闲超时退出、页异常 GENERATING/PLANNING→ERROR 继续）、全 READY finalize、`-race` 竞态测试
- [ ] 2.5 实现 compile_page（READY 非 force 直返 / force 重置 PENDING 同步编译）与 rebuild_book（清队列+去重、非 Overview 全部 PENDING、SPINE_READY、auto_compile 重入队）

## 3. 块编辑与学习交互

- [ ] 3.1 实现块编辑五操作：regenerate_block（params_override）、insert_block（非法 BlockType 400、position、compile_now）、delete_block、move_block、change_block_type（重生成）；目标不存在 404；流事件落 `block` 阶段
- [ ] 3.2 实现学习交互：create_deep_dive_subpage（ContentType 校验默认 concept、PENDING 走编译）、record_quiz_attempt（更新并返回进度）、supplement_for_weakness（页内追加补救块）、set_page_chat_session；进度随 book 持久化

## 4. /api/v1/book REST 与 book WS

- [ ] 4.1 实现 22 REST 端点（gin group，HTTP 鉴权）：读取组 6、创作组 5（空 user_intent 400 / 非法 proposal 400 / 不存在 404）、块编辑组 5、学习组 4、维护组 2（kb_drift + log_health、refresh-fingerprints）；ValueError 类查找失败→404、引擎内部错误→500
- [ ] 4.2 实现 `WS /api/v1/book/ws`：WS 鉴权失败关闭；每条消息新建 StreamBus + 转发 goroutine（`{type, source, stage, content, metadata}` 序列化、单写 goroutine）；五种消息类型与 `*_result` 终态；缺 type/未知/解析失败/异常 → error 且连接保持；每消息 finally 关 bus + cancel 转发；`goleak` 断言无泄漏

## 5. Co-Writer

- [ ] 5.1 实现 EditAgent：/edit（text≤600k、instruction≤10k、action 三值、可选 source 检索）、/automark；按语言缓存单例 + 每请求刷新 LLM 配置；产出剥离 thinking 标签与外层围栏
- [ ] 5.2 实现 /edit_react：选区≤120k、mode 四值（none 且无 instruction → 400 i18n、空选区 400）、instruction 空时按 mode 注入默认指令、逐工具检索（rag 无 kb_name 跳过、失败降级不阻塞）、参考资料块拼装、`tools_used` 返回、历史记录追加
- [ ] 5.3 实现 /edit_react/stream SSE：先同校验（4xx）、`text/event-stream` + `Cache-Control: no-cache` + `X-Accel-Buffering: no`、内部 StreamBus 转 `event: stream` 帧（exploring 工具轨迹 + responding 逐 chunk 与 stage start/end）、终态 `event: result` / `event: error`
- [ ] 5.4 实现历史与文档端点：GET /history（`{history, total}`）、GET /history/{id}（404）、GET /tool_calls/{id}（文件名模式、404）、documents 五端点（摘要视图按更新排序 / 完整文档 / PUT 单字段 / DELETE `{deleted}`）

## 6. 验收（spec Scenario 落测试 + 协议对等）

- [ ] 6.1 spec 全部 12 个 Scenario 落为 Go 测试（replay provider）：完整创作链、跳读同步编译、critique 可见、非法块类型 400、补救块、整书视图、WS 内逐步创作（中途失败连接保持）、RAG 选区改写、SSE 流、tool_calls 回看、非法 doc_id 无 FS 操作、重启恢复路径
- [ ] 6.2 协议对等验收（acceptance.md M3 矩阵「Book / Co-Writer」行）：22 REST 端点 contract test 对 golden OpenAPI spec；`/api/v1/book/ws` 按前端消费消息子集录制基线 fixtures，Go 回放逐事件 diff（acceptance.md 3.1 legacy WS 条款）
- [ ] 6.3 B 类数据兼容验收：Go 直接打开基线 book/co_writer 数据快照读写；Go 写入后回切 Python 可用（acceptance.md 3.2 反向兼容）
