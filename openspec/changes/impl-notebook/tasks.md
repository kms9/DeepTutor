# Tasks: impl-notebook

## 1. 学习笔记本文件存储

- [ ] 1.1 `internal/notebook/model.go` + `filestore.go`：`notebooks_index.json` + `<id>.json` 读写、字段与基线一致（`created_at` epoch 秒整型）、损坏文件降级空数据、原子写与索引同步
- [ ] 1.2 `internal/notebook/manager.go`：笔记本 CRUD、per-notebook 互斥锁、per-user 路径解析、thinking tags 清洗统一入口
- [ ] 1.3 基线文件 round-trip 兼容测试（读→写→字段比对不改写格式）

## 2. 记录管理与 LLM 摘要

- [ ] 2.1 `internal/notebook/summary.go`：摘要 agent（eino ChatModel 流式、按 `metadata.ui_language` 选语言）
- [ ] 2.2 `Manager.AddRecord`：summary 非空清洗直用/为空生成、多本同记录加入、返回实际加入清单；`UpdateRecord`/`DeleteRecord` 局部更新与 404

## 3. notebook REST（gin group /api/v1/notebook）

- [ ] 3.1 `GET /list`、`GET /statistics`、`POST /create`（默认值）、`GET /{notebook_id}`（404）、`PUT /{notebook_id}`、`DELETE /{notebook_id}`、`GET /health`
- [ ] 3.2 `POST /add_record`；`POST /add_record_with_summary` SSE（`summary_chunk`* → `result`/`error`）
- [ ] 3.3 `PUT /{notebook_id}/records/{record_id}`、`DELETE /{notebook_id}/records/{record_id}`（未命中 404）

## 4. question-notebook（SQLite + gin group /api/v1/question-notebook）

- [ ] 4.1 `internal/notebook/questionstore.go`：基线三表读写（复用 session-store `*sql.DB`）、`ON CONFLICT(session_id, turn_id, question_id) DO UPDATE` upsert（行 id 不变）
- [ ] 4.2 `POST /entries/upsert`：base64 图片物化 AttachmentStore（引用存储、null 保留/空数组清空、单图失败不整体失败、不回传 base64、session 缺失 404）
- [ ] 4.3 `GET /entries`（过滤 + 分页 items/total）、`GET /entries/lookup/by-question`（`missing_ok=true` → 204）、`GET /entries/{entry_id}`、`PATCH /entries/{entry_id}`（空更新 400）、`DELETE /entries/{entry_id}`
- [ ] 4.4 分类组：`GET /categories`（计数）、`POST /categories`（201/409）、`PATCH /categories/{category_id}`、`DELETE /categories/{category_id}`（级联清关联保条目）、条目关联增删两路由

## 5. 工具存储接口（供 impl-tools-builtin）

- [ ] 5.1 `list_notebook` 数据面：索引模式（50 本上限）与钻取模式（80 条、倒序、标题 100/摘要 240 截断）复用 `Manager` 查询
- [ ] 5.2 `write_note` 数据面：append→`AddRecord`（chat 域）、edit→`UpdateRecord`（`note` 作 summary 替换）

## 6. 测试与验收（spec Scenario 落测试 + 协议对等）

- [ ] 6.1 将本 spec 全部 8 个 Scenario 落为 Go 测试：基线文件直接可用、删除笔记本/404、SSE 流式摘要与清洗、upsert 幂等（id 不变）、missing_ok 204、删分类保条目、工具与 REST 数据一致、工具写入 UI 可见
- [ ] 6.2 `/api/v1/notebook` 与 `/api/v1/question-notebook` 全路由 contract test 对 OpenAPI golden spec 逐字段 diff（acceptance.md 3.1 A 类 + M2 矩阵「Notebook / Question Notebook」项）
- [ ] 6.3 数据兼容验收：基线 `data/` 快照上三表 CRUD 与 notebook JSON 读写、回切 Python 可打开（acceptance.md 3.2 B 类双向兼容）
