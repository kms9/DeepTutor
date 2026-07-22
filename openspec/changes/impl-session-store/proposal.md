# Proposal: impl-session-store

## Why

会话/消息/turn 事件是 DeepTutor 的核心持久层，Go 版必须以现有 SQLite 库 `data/user/chat_history.db` 为唯一事实源、免迁移打开 Python v1.5.2 老库并双向兼容，`/api/v1/sessions` 路由族还要与前端逐字段对等。本模块是 ROADMAP 中 Wave 0（基础契约）/ M0 里程碑的成员（M0 交付「SQLite store 打开基线库 CRUD 通过」，sessions REST 完整对等在 M1「chat 主链路」矩阵验收），行为规格见 `docs/golang-req/openspec/specs/session-store/spec.md`。

## What Changes

- 实现与基线一致的 SQLite schema（sessions/messages/turns/turn_events/notebook_entries/notebook_categories/notebook_entry_categories）幂等初始化，与打开老库时的增量列迁移（补列、parent 线性链回填、notebook UNIQUE 约束重建、legacy 库位置搬迁、`PRAGMA foreign_keys = ON`）。
- 实现会话生命周期：`create_session`/`ensure_session`/`get_session`（含 `status`/`active_turn_id`/`capability`/`preferences` 派生字段与 id/session_id 双键）。
- 实现消息追加与编辑分支：`add_message` 三种 parent 语义、JSON 序列化降级、`get_messages_for_context` 沿 parent 链回溯（防环上限）。
- 实现 turn 状态机：`running → completed|failed|cancelled`、单 session 单活跃 turn 约束、重启后孤儿 turn 收敛为 `failed`。
- 实现 `append_turn_event`（seq 自动分配 / 显式 seq；**（有意差异 D-001）** `(turn_id, seq)` 冲突改为报错而非 `INSERT OR REPLACE` 覆盖）与 `get_turn_events(after_seq)` 增量回放、`last_seq` 派生字段。
- 实现会话列表与 imported 会话判别（`imported_` 前缀、确定性 id、幂等导入）。
- 实现 `/api/v1/sessions` REST 路由族（list/get/rename/delete/branch-selection/消息配对删除/quiz-results），含超大事件载荷的 UI 截断（1 MiB + `_truncated` 标记，仅影响返回不影响持久化）。
- 实现错题本（notebook）upsert 与分类多对多语义。
- 实现存储并发模型：全部读写串行化到单写者，事务内完成、异常回滚。

## Capabilities

### New Capabilities

- `session-store`: deeptutor-go 的会话持久层与 sessions REST API——SQLite schema 兼容免迁移、store API（消息分支/turn 状态机/事件回放/错题本）、`/api/v1/sessions` 路由族。

### Modified Capabilities

（无）

## Impact

- 新建代码：`deeptutor-go/internal/session/`（schema、迁移、store）与 `deeptutor-go/internal/api/`（sessions REST，gin route group）。
- 技术选型：`modernc.org/sqlite` + `database/sql`（免 CGO），REST 用 `gin-gonic/gin`。
- 依赖的其他 change（按 ROADMAP 依赖图与模块 spec 声明）：`impl-foundation-config`（`PathService` 提供 `chat_history.db` 路径）、`impl-foundation-stream`（`StreamEvent` 事件信封形状）。
- 被依赖：`impl-turn-runtime`（store → turn，事件持久化与 resume）、间接支撑 `impl-api-unified-ws` 的断线回放。
- 有意差异：D-001（`append_turn_event` 冲突报错，acceptance.md §6），前端影响登记为「无」。
- 不修改 Python 基线与 `web/` 前端；Go 写入后的库须能回切 Python 打开（回滚前提）。
