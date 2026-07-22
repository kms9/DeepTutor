# Tasks: impl-session-store

## 1. schema 与打开流程

- [ ] 1.1 实现 `internal/session/schema.go`：七张表的 `CREATE TABLE IF NOT EXISTS` DDL 与幂等 `EnsureSchema`
- [ ] 1.2 实现 `Open`：modernc.org/sqlite DSN（`foreign_keys(1)`、`busy_timeout`、WAL）、`SetMaxOpenConns(1)` 单写者 + 互斥 + 事务辅助
- [ ] 1.3 实现 `migrate.go`：legacy `data/chat_history.db` 搬迁、增量补列（`preferences_json`/`metadata_json`/`parent_message_id`/`turn_id`/`user_answer_images_json`/`ai_judgment`）、parent 线性链回填、notebook UNIQUE 约束重建

## 2. 会话与消息

- [ ] 2.1 实现会话生命周期：`CreateSession`（`unified_<epoch_ms>_<8hex>`、title 归一化截断）、`EnsureSession`、`GetSession`（`status`/`active_turn_id`/`capability`/`preferences` 派生 + id/session_id 双键）
- [ ] 2.2 实现 `AddMessage` 三态 parent（Auto/Explicit/Root）、`sessions.updated_at` 刷新、JSON 序列化降级为字符串表示
- [ ] 2.3 实现 `GetMessagesForContext`：leaf 回溯排除旁支、角色过滤、防环上限 10000、无 leaf 返回全量线性消息
- [ ] 2.4 实现 `ListSessions`/`ListImportedSessions`：`updated_at DESC`、摘要派生字段、`imported_` 前缀判别（LIKE 下划线转义）、确定性导入 id 与幂等导入

## 3. turn 与事件

- [ ] 3.1 实现 turn 状态机：`CreateTurn`（单活跃约束报错携带既有 turn id）、`FinishTurn`（终态写 `finished_at`）、`MarkOrphanIfRunning` 孤儿收敛
- [ ] 3.2 实现 `AppendTurnEvent`：seq 自动分配（`MAX(seq)+1`）/显式 seq、payload 回填、`turns.updated_at` 刷新、普通 INSERT + `(turn_id,seq)` 冲突映射 `ErrSeqConflict`（有意差异 D-001）
- [ ] 3.3 实现 `GetTurnEvents(after_seq)` 增量回放（StreamEvent 信封形状、metadata 解析失败回空 object）与 turn 查询 API 的 `last_seq` 派生

## 4. 错题本

- [ ] 4.1 实现 `UpsertNotebookEntries`：`(session_id, turn_id, question_id)` 冲突键、部分更新语义（未带图片列表不清空 `user_answer_images_json`）、空 question/question_id 跳过、session 不存在报错
- [ ] 4.2 实现条目更新字段白名单（`bookmarked`/`followup_session_id`/`user_answer`/`is_correct`/`ai_judgment`）与分类多对多级联

## 5. sessions REST

- [ ] 5.1 实现 gin 路由组 `/api/v1/sessions`：list（limit/offset 校验）、get（404）、PATCH 重命名、DELETE 级联删除
- [ ] 5.2 实现 branch-selection PUT、消息配对删除（running 409、附件清理回调、返回 `{deleted, attachment_ids, turn_id, was_running}`）、quiz-results POST（空 answers 400、`[Quiz Performance]` 消息 + notebook upsert）
- [ ] 5.3 实现 `GET /sessions/{id}` 内嵌 events 的 1 MiB UI 截断（`tool_result`/`observation`、`_truncated` 标记、不回写库）

## 6. 测试与验收

- [ ] 6.1 把 `session-store` spec 的全部 Scenario（11 个 Requirement / 15 个 Scenario）逐条落为 Go 测试，含 `-race` 并发 `AppendTurnEvent` 用例与 D-001 冲突用例
- [ ] 6.2 数据兼容验收：用 Python v1.5.2 基线库快照跑免迁移打开 + 全表 CRUD（acceptance.md §4 M0 矩阵「Go 骨架：SQLite store 打开基线库 CRUD 通过」、§3.2 B 类）
- [ ] 6.3 反向兼容验收：Go 写入后的库回切 Python 基线打开并读写（acceptance.md §3.2 第 2 条）
- [ ] 6.4 协议对等验收：`/api/v1/sessions` 路由族对 golden spec 做 contract test（status/结构/字段名/类型逐项 diff，acceptance.md §3.1、§4 M1 矩阵「sessions REST」）
