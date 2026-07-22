# Change: impl-notebook

## Why

deeptutor-go 需要在 Go 侧承载两套笔记本子系统：学习笔记本（JSON 文件存储 + `/api/v1/notebook`）与错题/刷题笔记本（SQLite 三表 + `/api/v1/question-notebook`），并为 `list_notebook`/`write_note` 两个 chat 工具提供与 REST 同源的数据面。本 change 依据模块行为规格 `docs/golang-req/openspec/specs/notebook/spec.md` 立项，对应 `docs/golang-req/openspec/ROADMAP.md` Wave 2「领域服务与工具」、里程碑 M2（acceptance.md M2 矩阵「Notebook / Question Notebook」项：contract test 通过、前端对应页面可用）。存储格式与表 schema 必须与基线兼容，Web 与 CLI 操作同一份数据。

## What Changes

- 学习笔记本文件存储：`notebooks_index.json` + 每本 `<notebook_id>.json`，记录字段与基线一致（`type` 六域、epoch 秒时间戳），摘要入库前清除 thinking tags，损坏文件降级为空数据，基线文件直接读写。
- notebook REST：笔记本 CRUD（list/statistics/create/get/update/delete/health）+ 记录管理（`add_record` 多本加入 + LLM 摘要生成、`add_record_with_summary` SSE 流式、记录局部更新与删除）。
- question-notebook 表语义：沿用基线 SQLite schema（`notebook_entries`/`notebook_categories`/`notebook_entry_categories`，`UNIQUE(session_id, turn_id, question_id)` upsert）。
- question-notebook REST：条目组（upsert 含 base64 图片物化进 AttachmentStore、过滤分页、`missing_ok=true` 探测 204、PATCH/DELETE）+ 分类组（计数/201/409/改名/级联清除关联不删条目）。
- `list_notebook`/`write_note` 工具与存储对应：同一 manager 数据源、per-user 路径解析、append→`add_record` / edit→`update_record` 映射。

## Capabilities

### New Capabilities

- `notebook`: 两套笔记本子系统的 Go 侧行为契约——学习笔记本文件存储与 REST、question-notebook SQLite 表语义与 REST、`list_notebook`/`write_note` 工具与存储的对应关系。

### Modified Capabilities

（无）

## Impact

- **依赖的其他 change**：
  - `impl-session-store`：SQLite 库（`notebook_entries` 等三表与 sessions 同库）、AttachmentStore（作答图片物化）、session 外键级联。
  - 间接依赖：`impl-foundation-config`（路径服务）、`impl-llm-provider`（记录摘要生成的 LLM 调用）。
- **被依赖**：`impl-tools-builtin` 的 `list_notebook`/`write_note` 消费本模块 manager 接口。
- **受影响代码**：新增 `internal/notebook/`（文件存储 service + question-notebook store）与 `internal/api/` 下 notebook / question-notebook 路由组。
- **数据兼容**：基线 notebook JSON 文件与 SQLite 表免迁移直接读写（acceptance.md 3.2 B 类）。
- **前端影响**：Notebook 页与 quiz 错题本页全功能。
