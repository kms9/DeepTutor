# notebook Specification

## Purpose

本模块定义两套笔记本子系统在 Go 侧的目标行为：学习笔记本（用户组织的记录集合，JSON 文件存储于 `data/user/workspace/notebook/`，REST 面 `/api/v1/notebook`）与错题/刷题笔记本（quiz 场景的题目条目与分类，SQLite `notebook_entries` / `notebook_categories` / `notebook_entry_categories` 表，REST 面 `/api/v1/question-notebook`），以及 `list_notebook` / `write_note` 两个 chat 工具与学习笔记本存储的对应关系。存储格式与表 schema 必须与基线兼容，Web 与 CLI 操作同一份数据。

- 参考实现（基线）：`deeptutor/services/notebook/service.py`、`deeptutor/api/routers/notebook.py`、`question_notebook.py`、`deeptutor/services/session/sqlite_store.py`（notebook 表 DDL）、`deeptutor/tools/list_notebook.py`、`write_note.py`
- 依赖 spec / 里程碑：依赖 foundation-config（路径）、session-store（SQLite 与 AttachmentStore）、llm-provider（记录摘要生成）；被 tools-builtin（list_notebook/write_note）依赖；里程碑：M2

## Requirements

### Requirement: 学习笔记本存储模型
学习笔记本 SHALL 以文件存储：`notebooks_index.json`（笔记本索引：id/名称/描述/颜色/图标/时间戳/记录数）+ 每本一个 `<notebook_id>.json`（含全部记录）。记录字段 SHALL 与基线一致：`id`、`type`（`solve`/`question`/`research`/`chat`/`co_writer`/`tutorbot`）、`title`、`summary`、`user_query`、`output`、`metadata`、`created_at`（epoch 秒）、可选 `kb_name`。记录摘要入库前 SHALL 清除模型思考标签（thinking tags）。加载损坏文件 SHALL 降级为空数据而非报错；对基线生成的既有文件 SHALL 直接读写。

#### Scenario: 基线笔记本文件直接可用
- **WHEN** deeptutor-go 指向基线的 notebook 目录并请求 `GET /api/v1/notebook/list`
- **THEN** 全部笔记本连同记录数正确返回，文件格式不被改写

### Requirement: notebook REST — 笔记本 CRUD
系统 SHALL 提供 `GET /list`（全部笔记本 + 总数）、`GET /statistics`（统计）、`POST /create`（name 必填，description/color/icon 可选带默认）、`GET /{notebook_id}`（详情含全部记录，未知 id 404）、`PUT /{notebook_id}`（局部更新）、`DELETE /{notebook_id}`（删除笔记本文件并同步索引）、`GET /health`。

#### Scenario: 删除笔记本
- **WHEN** `DELETE /{notebook_id}` 命中既有笔记本
- **THEN** 对应 JSON 文件删除且索引条目移除
- **AND** 未知 id 返回 404

### Requirement: notebook REST — 记录管理与 LLM 摘要
`POST /add_record` SHALL 接受 `notebook_ids`（可多本）、`record_type`、`title`、`user_query`、`output`、`metadata`、可选 `summary` 与 `kb_name`：`summary` 非空时清洗后直接使用，为空时 SHALL 调用摘要 agent（按 `metadata.ui_language` 选语言）生成，然后将同一记录加入全部目标笔记本，返回记录与实际加入的笔记本清单。`POST /add_record_with_summary` SHALL 以 SSE 流式返回：先逐块推送 `summary_chunk`，结束推送含记录的 `result`（或 `error`）。`PUT /{notebook_id}/records/{record_id}` SHALL 支持记录字段局部更新，`DELETE /{notebook_id}/records/{record_id}` 移除记录；未命中均 404。

#### Scenario: 无摘要时流式生成
- **WHEN** 客户端以空 `summary` 调用 `add_record_with_summary`
- **THEN** SSE 依次给出摘要文本块与最终 `result` 事件
- **AND** 落库的 summary 为清洗思考标签后的全文

### Requirement: question-notebook 表语义
系统 SHALL 沿用基线 SQLite schema：`notebook_entries`（自增 `id`；`session_id` 外键级联删除；`turn_id`/`question_id`；题面 `question`/`question_type`/`options_json`/`correct_answer`/`explanation`/`difficulty`；作答 `user_answer`/`user_answer_images_json`/`is_correct`；管理 `bookmarked`/`followup_session_id`/`ai_judgment`；时间戳；`UNIQUE(session_id, turn_id, question_id)`）、`notebook_categories`（`name` 唯一）、`notebook_entry_categories`（entry↔category 多对多，双向级联删除）。upsert SHALL 以唯一键为冲突判定：命中则更新，未命中则插入。

#### Scenario: 同题重复提交为更新
- **WHEN** 同一 `(session_id, turn_id, question_id)` 的作答被提交两次
- **THEN** 表中仅一行且反映最后一次作答
- **AND** 行 `id` 保持不变

### Requirement: question-notebook REST — 条目
系统 SHALL 提供：`POST /entries/upsert`（作答图片以 base64 上传时 SHALL 物化进 AttachmentStore 并仅存 `{id,url,filename,mime_type}` 引用，绝不回传 base64；`user_answer_images=null` 表示不动既有图片，空数组显式清空；单图失败丢弃该图并告警而不整体失败；session 不存在 404）、`GET /entries`（按 `category_id`/`bookmarked`/`is_correct` 过滤 + limit/offset 分页，返回 items+total）、`GET /entries/lookup/by-question`（按 session+question 定位；`missing_ok=true` 时未命中返回 204 而非 404，供 quiz 端静默探测）、`GET /entries/{entry_id}`、`PATCH /entries/{entry_id}`（`bookmarked`/`followup_session_id`/`ai_judgment`，空更新 400）、`DELETE /entries/{entry_id}`。

#### Scenario: 静默探测未保存题目
- **WHEN** quiz 界面以 `missing_ok=true` lookup 一道尚未保存的题
- **THEN** 返回 204 No Content 且不产生错误日志噪音

### Requirement: question-notebook REST — 分类
系统 SHALL 提供 `GET /categories`（含每类条目计数）、`POST /categories`（201；重名 409）、`PATCH /categories/{category_id}`（改名）、`DELETE /categories/{category_id}`，以及条目关联 `POST /entries/{entry_id}/categories` 与 `DELETE /entries/{entry_id}/categories/{category_id}`。删除分类 SHALL 级联清除其关联行但不删除条目本身。

#### Scenario: 删除分类保留条目
- **WHEN** 删除一个含 10 个条目关联的分类
- **THEN** 关联行清除、10 个条目仍在
- **AND** 后续 `GET /entries?category_id=<该 id>` 返回空集

### Requirement: list_notebook 工具与存储的对应
`list_notebook` 工具 SHALL 直接查询学习笔记本 manager（与 REST 同一数据）：索引模式输出等价于 `GET /list` 的内容渲染为 markdown（上限 50 本），钻取模式等价于读取单本记录列表（按 `created_at` 倒序，上限 80 条，标题截 100 字符、摘要截 240 字符）。工具 SHALL NOT 绕过 per-user 路径解析。

#### Scenario: 工具与 REST 数据一致
- **WHEN** 用户经 REST 新建笔记本后 LLM 立即调用 `list_notebook`
- **THEN** 新笔记本出现在工具输出的索引中

### Requirement: write_note 工具与存储的对应
`write_note` 的 append 模式 SHALL 等价于向目标笔记本 `add_record`（`record_type` 归 chat 域；正文来源见 tools-builtin spec），edit 模式 SHALL 等价于 `update_record` 的局部更新（title/正文/summary，`note` 作为 summary 替换）。目标笔记本校验、长度截断与错误编码 SHALL 与工具 spec 一致；写入后经 REST/UI 立即可见。

#### Scenario: 工具写入 UI 可见
- **WHEN** LLM 用 `write_note(mode="append", ...)` 保存记录
- **THEN** 前端 `GET /{notebook_id}` 立即包含该记录且字段完整（title/summary/正文/created_at）
