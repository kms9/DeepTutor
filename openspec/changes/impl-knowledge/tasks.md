# Tasks: impl-knowledge

## 1. 注册表与类型基座

- [ ] 1.1 `internal/knowledge/types.go` + `naming.go`：KB 条目/状态/embedding signature/index version 模型、KB 名校验、默认 KB 别名归一（`default`/`current`/`selected`/`默认`）
- [ ] 1.2 `internal/knowledge/registry.go`：`kb_config.json` flock 文件锁 + 原子写、磁盘未注册 KB 自动注册
- [ ] 1.3 数据兼容冒烟：指向基线 `data/knowledge_bases/` 加载注册表与目录布局不改写

## 2. 生命周期与 Sidecar facade

- [ ] 2.1 `internal/knowledge/manager.go`：`Create`（initializing 登记 → `raw/` 落盘 → 异步索引）、`Upload`（增量）、`Delete`（整目录 + 解注册；连接型只解注册）、`DELETE .../files/{filename}`（删文档 + 同步索引）
- [ ] 2.2 支持文件类型校验与 `GET /supported-file-types`；上传路径越界拒绝
- [ ] 2.3 索引任务编排：goroutine + `sidecar.RAGIndex` 转发、task id、per-task 日志 ring buffer + `GET /tasks/{task_id}/stream` SSE；Sidecar 不可用时任务明确失败
- [ ] 2.4 版本化存储：按激活 embedding signature 选择/创建 `version-N/`、历史保留、`POST /{kb_name}/reindex`（按 `rag_provider` 路由）、`POST /{kb_name}/retry`

## 3. reconcile 与进度

- [ ] 3.1 `internal/knowledge/reconcile.go`：磁盘版本（含 legacy 布局）对照当前指纹，`needs_reindex`/`embedding_mismatch` 置位/清除、全新 KB 不误标、连接型跳过、`index_versions` 快照刷新
- [ ] 3.2 `internal/knowledge/progress.go`：进度帧持久化（stage/message/percent/current/total/timestamp/task_id）、`GET /{kb_name}/progress`、`POST /{kb_name}/progress/clear`、per-KB broadcaster
- [ ] 3.3 `internal/api/knowledgeapi/progress_ws.go`：gorilla/websocket upgrade、鉴权、无活动任务快速完成路径（发终态帧即关）、活动任务实时推送

## 4. 连接型 KB 与 linked folders

- [ ] 4.1 `connected.go`：`POST /connect-obsidian`、`POST /probe-folder` + `POST /connect-folder`（允许列表校验）、`POST /probe-lightrag-server` + `POST /connect-lightrag-server`；删除只解注册不删外部数据
- [ ] 4.2 `linked.go`：`POST /{kb_name}/link-folder`、`GET /{kb_name}/linked-folders`、`DELETE .../linked-folders/{folder_id}`、`POST /{kb_name}/sync-folder/{folder_id}`（增删改检测 → 仅变更文件增量索引 → sync state 更新）

## 5. 配置面与文件管理 REST

- [ ] 5.1 provider 配置组：`GET /health`、`GET /rag-providers`、`PUT /rag-providers/{provider}/mode`、`GET/PUT /rag-pipelines/{provider}/config`、`GET /rag-pipelines/{provider}/preflight`、`GET /rag-pipelines/model-options`、`PUT /rag-pipelines/active-model`（持久化后即时生效）
- [ ] 5.2 KB 配置组：`GET /configs`、`GET/PUT /{kb_name}/config`、`POST /configs/sync`、`GET /default`、`PUT /default/{kb_name}`
- [ ] 5.3 文件管理组：`GET /list`、`GET /{kb_name}`、`GET /{kb_name}/files`（文件树）、`POST /{kb_name}/folders`、`POST /{kb_name}/files/move`、`GET /{kb_name}/files/{filename}`（下载）、`GET /{kb_name}/file-preview-text/{filename}`（截断标记）；全路径参数防穿越
- [ ] 5.4 `internal/knowledge/access.go`：归属解析（可见/可写、`assigned`/`read_only` 标记；M2 先单用户实现留接口），`rag` 工具复用 `ResolveKBName`

## 6. 测试与验收（spec Scenario 落测试 + 协议对等）

- [ ] 6.1 将本 spec 全部 11 个 Scenario 落为 Go 测试（mock Sidecar）：基线 KB 免迁移、创建全流程、embedding 切回复用、mismatch 置标、WS 快速关闭、连接文件夹、增量同步、preflight 拦截、默认别名解析、预览截断、被分配 KB 只读
- [ ] 6.2 `/api/v1/knowledge` 全路由 contract test 对 OpenAPI golden spec 逐字段 diff（acceptance.md 3.1 A 类 + M2 矩阵「Knowledge 接入面」项）
- [ ] 6.3 progress WS 帧序列与基线 fixtures 对比（acceptance.md 3.1 legacy WS `knowledge/{kb}/progress/ws` 子集）
- [ ] 6.4 e2e：真实 Sidecar 建库→检索→reindex 闭环 + kill Sidecar 后 KB 请求返回明确错误（acceptance.md 3.4 Sidecar 降级、M2 矩阵 e2e 项）
