# Delta Spec: session-store

> 事实源：`docs/golang-req/openspec/specs/session-store/spec.md`（Requirement/Scenario 原样搬运，未增删语义；「（有意差异）」标注对应 acceptance.md D-001）。

## ADDED Requirements

### Requirement: SQLite schema 兼容与免迁移打开
系统 SHALL 使用与基线一致的表结构（`CREATE TABLE IF NOT EXISTS`，幂等初始化）：

- `sessions(id TEXT PK, title, created_at REAL, updated_at REAL, compressed_summary, summary_up_to_msg_id, preferences_json)`
- `messages(id INTEGER PK AUTOINCREMENT, session_id REFERENCES sessions ON DELETE CASCADE, role, content, capability, events_json, attachments_json, metadata_json, created_at REAL, parent_message_id INTEGER)`
- `turns(id TEXT PK, session_id REFERENCES sessions ON DELETE CASCADE, capability, status DEFAULT 'running', error, created_at, updated_at, finished_at)`
- `turn_events(id INTEGER PK AUTOINCREMENT, turn_id REFERENCES turns ON DELETE CASCADE, seq INTEGER NOT NULL, type, source, stage, content, metadata_json, timestamp REAL, created_at REAL, UNIQUE(turn_id, seq))`
- `notebook_entries(..., UNIQUE(session_id, turn_id, question_id))`、`notebook_categories(id, name UNIQUE, created_at)`、`notebook_entry_categories(entry_id, category_id, PK(entry_id, category_id))`

打开老库时 SHALL 执行基线同款的增量列迁移：缺 `sessions.preferences_json`、`messages.metadata_json`、`messages.parent_message_id`、`notebook_entries.turn_id`/`user_answer_images_json`/`ai_judgment` 时 `ALTER TABLE` 补列；补 `parent_message_id` 时 SHALL 按每个 session 内 `id` 升序把消息串成线性链回填 parent。旧 UNIQUE 约束仍为 `(session_id, question_id)` 的 `notebook_entries` SHALL 重建表换成 `(session_id, turn_id, question_id)`。连接 SHALL 启用 `PRAGMA foreign_keys = ON`。位于 `data/chat_history.db` 的 legacy 库 SHALL 一次性移动到 `data/user/chat_history.db`。

#### Scenario: 直接打开 Python 老库
- **WHEN** Go 版指向 Python v1.5.2 写出的 `chat_history.db` 启动
- **THEN** 无需任何外部迁移工具即可读写全部会话、消息、turn 事件与错题本数据

#### Scenario: 缺列老库自动补齐
- **WHEN** 打开一个 `messages` 表尚无 `parent_message_id` 列的老库
- **THEN** 该列被添加且每个 session 的既有消息按 id 顺序回填为线性父链

### Requirement: 会话生命周期与 id 约定
`create_session` SHALL 生成 `unified_<epoch_ms>_<8位hex>` 形式的 id（也接受调用方显式指定 id），title 去空白后为空则取 `"New conversation"`，且 SHALL 截断到 100 字符。`ensure_session` SHALL 在给定 id 存在时返回既有会话、否则创建新会话。`get_session` SHALL 附带派生字段：`status`（最近一条 turn 的状态，无 turn 时 `"idle"`）、`active_turn_id`（最近 running turn 的 id，无则空串）、`capability`（最近一条 turn 的 capability）、`preferences`（`preferences_json` 解析后的 object），并同时返回 `id` 与 `session_id` 两个同值键。

#### Scenario: 无 turn 会话的状态
- **WHEN** 查询一个从未运行过 turn 的会话
- **THEN** `status="idle"`、`active_turn_id=""`

### Requirement: 消息追加与编辑分支
`add_message` SHALL 支持三种 parent 语义：未指定 parent 时自动挂到该 session 当前最大 id 的消息之后（线性追加）；显式指定 `parent_message_id` 时挂到该消息之下（同 parent 的兄弟消息即编辑分支）；显式指定 parent 为 NULL 时挂到会话根（编辑首条消息的场景）。消息写入 SHALL 同步刷新 `sessions.updated_at`。`events`/`attachments`/`metadata` SHALL 分别以 JSON 序列化入 `events_json`/`attachments_json`/`metadata_json`，序列化遇到不可序列化值时 SHALL 降级为其字符串表示而非失败。

`get_messages_for_context(session_id, leaf_message_id)` SHALL 在给定 leaf 时沿 `parent_message_id` 回溯到根、按时间序返回该路径（只含 `user`/`assistant`/`system` 角色），从而排除所有旁支分支；未给 leaf 时返回全量线性消息。回溯 SHALL 有防环上限（基线为 10000 步）。

#### Scenario: 编辑消息产生兄弟分支
- **WHEN** 用户编辑消息 M（parent 为 P）并重新发送
- **THEN** 新消息以 P 为 parent 写入，与 M 互为兄弟分支
- **AND** 以新消息为 leaf 构建 LLM 上下文时不包含 M 及其子树

### Requirement: turn 状态机与单活跃约束
turn 状态 SHALL 为 `running` → `completed` | `failed` | `cancelled`；进入终态时 SHALL 写 `finished_at`。`create_turn` SHALL 生成 `turn_<epoch_ms>_<10位hex>` 形式 id；同一 session 已存在 `running` turn 时 SHALL 报错拒绝创建。进程重启后遗留的 `running` turn（库里 running 但本进程无对应执行体）SHALL 在下次访问时被判定为孤儿并置为 `failed`（错误信息标注为中断）。

#### Scenario: 并发发起第二个 turn 被拒
- **WHEN** session 有一个 running turn 时再次 `create_turn`
- **THEN** 调用失败并携带既有活跃 turn id

#### Scenario: 重启后孤儿 turn 收敛
- **WHEN** 服务重启后订阅一个库中仍为 running 的 turn
- **THEN** 该 turn 被置为 `failed`，订阅方收到合成的终态事件

### Requirement: append_turn_event 与 seq 冲突语义（有意差异）
`append_turn_event(turn_id, event)` SHALL：turn 不存在时报错；事件带 `seq>0` 时使用该 seq，否则取 `SELECT MAX(seq)+1`（空表从 1 开始）；回填 payload 的 `seq`/`turn_id`/`session_id` 后落库，并刷新 `turns.updated_at`；返回补全后的事件 payload。

（有意差异）基线对 `(turn_id, seq)` 使用 `INSERT OR REPLACE`，重复 seq 会静默覆盖旧事件。Go 版 SHALL 改为普通 INSERT：`(turn_id, seq)` 冲突时 SHALL 返回明确的冲突错误、SHALL NOT 覆盖已持久化事件。此差异已在差异登记表登记；调用方（turn runtime）的 seq 分配保证正常路径不会产生冲突。

#### Scenario: 自动分配 seq
- **WHEN** 对已有 3 条事件（seq 1..3）的 turn 追加一条 `seq=0` 的事件
- **THEN** 该事件以 `seq=4` 落库

#### Scenario: 重复 seq 报错而非覆盖（有意差异）
- **WHEN** 对同一 turn 追加两条 `seq=5` 的事件
- **THEN** 第二次追加返回冲突错误，第一条事件内容保持不变

### Requirement: 事件回放查询
`get_turn_events(turn_id, after_seq)` SHALL 返回 `seq > after_seq` 的事件、按 `seq` 升序，事件形状与 StreamEvent 信封一致（`type/source/stage/content/metadata/session_id/turn_id/seq/timestamp`，`metadata` 由 `metadata_json` 解析、解析失败回空 object）。turn 查询 API（`get_turn`/`get_active_turn`/`list_active_turns`）SHALL 附带派生字段 `last_seq = COALESCE(MAX(seq), 0)`，供断线重连客户端从 `after_seq=last_seq` 续订。

#### Scenario: 断线重连增量回放
- **WHEN** 客户端携带 `after_seq=7` 重新订阅一个已有 10 条事件的 turn
- **THEN** 仅收到 seq 8、9、10 三条事件，且按序

### Requirement: 会话列表与 imported 会话判别
`list_sessions(limit, offset)` SHALL 按 `updated_at DESC` 返回会话摘要，摘要含 `message_count`、`status`、`active_turn_id`、`capability`、`last_message`（该 session 最新一条非空内容消息）、`preferences`。外部导入的会话以 id 前缀 `imported_` 判别：`list_sessions` SHALL 排除它们，`list_imported_sessions` SHALL 只返回它们（LIKE 匹配时下划线 SHALL 按字面量处理）。导入 id SHALL 按 `imported_<source>_<external_id>` 确定性构造（非法字符替换为 `-`），同 id 重复导入 SHALL 幂等（不重复插入、不覆盖既有内容，仅可回填 `preferences.import` 中的 agent 归属字段）。

#### Scenario: 导入会话不混入常规历史
- **WHEN** 库中同时存在原生会话与 `imported_codex_abc` 会话
- **THEN** `GET /api/v1/sessions` 结果不含 imported 会话

### Requirement: sessions REST 路由
系统 SHALL 暴露与基线逐字段对等的 `/api/v1/sessions` 路由族：

- `GET /api/v1/sessions?limit&offset`：`limit` ∈ [1,200] 默认 50，`offset` ≥ 0；返回 `{"sessions":[...]}`
- `GET /api/v1/sessions/{session_id}`：返回会话 + `messages` + `active_turns`；不存在时 404 `"Session not found"`
- `PATCH /api/v1/sessions/{session_id}`：body `{"title": 1..100 字符}`，重命名后返回 `{"session": {...}}`
- `DELETE /api/v1/sessions/{session_id}`：级联删除并尽力清理附件存储，返回 `{"deleted": true, "session_id": ...}`
- `PUT /api/v1/sessions/{session_id}/branch-selection`：body `{"selected_branches": {parent_message_id: child_id}}`，合并写入 session preferences 并原样返回
- `DELETE /api/v1/sessions/{session_id}/messages/{message_id}`：删除该消息所属的整个 user/assistant 配对及关联 turn 与 turn_events；对应 turn 仍在 running 时 SHALL 返回 409；消息不存在时 404；成功时返回 `{deleted, attachment_ids, turn_id, was_running}` 并清理关联附件
- `POST /api/v1/sessions/{session_id}/quiz-results`：body `{answers: [...], turn_id}`，`answers` 为空时 400；把成绩汇总为 `[Quiz Performance]` 文本追加为一条 `role=user`、`capability=deep_question` 的消息，并 upsert 错题本条目；返回 `{recorded, session_id, answer_count, notebook_count, content}`

#### Scenario: 删除运行中的消息被拒
- **WHEN** 对一条其 turn 仍为 running 的消息调用 DELETE
- **THEN** 返回 409，库中消息与 turn 均未删除

### Requirement: 超大事件载荷的 UI 截断
`GET /api/v1/sessions/{session_id}` 返回消息内嵌 `events` 时，SHALL 对 `type` 为 `tool_result`/`observation` 的事件做载荷截断：`content` 以及 `metadata.tool_metadata` 的 `content`/`answer` 字段超过 1 MiB（1024*1024 字符）时截断并追加 `"\n\n[... content truncated]"`，同时在该事件上标记 `_truncated: true`。截断仅影响 UI 返回，SHALL NOT 影响持久化数据与 LLM 上下文。

#### Scenario: RAG 大结果被截断
- **WHEN** 某消息内嵌一条 `tool_result` 事件、content 长 2 MiB
- **THEN** 接口返回中该 content 为 1 MiB + 截断提示，事件带 `_truncated: true`，库内原文不变

### Requirement: 错题本（notebook）语义
`upsert_notebook_entries` SHALL 以 `(session_id, turn_id, question_id)` 为冲突键 upsert：冲突时仅更新 `user_answer`、`is_correct`、`updated_at`（调用带 `user_answer_images` 列表时额外更新 `user_answer_images_json`；未带时 SHALL 保留库中既有图片引用不被清空）；`question` 或 `question_id` 为空的条目跳过；session 不存在时报错。条目更新 API 允许修改的字段 SHALL 限于 `bookmarked`、`followup_session_id`、`user_answer`、`is_correct`、`ai_judgment`。分类为多对多：`notebook_entry_categories` 关联删除 SHALL 随条目/分类级联。

#### Scenario: 部分 upsert 不清空答题图片
- **WHEN** 某条目已存图片引用，随后一次仅带 `is_correct` 变更的 upsert 到达
- **THEN** `user_answer_images_json` 保持原值

### Requirement: 存储并发模型
store 的全部读写 SHALL 串行化到单写者（基线以进程内互斥 + 每调用独立连接实现；Go 版可用等价机制），每个操作在事务内完成、异常回滚，SHALL NOT 泄漏连接。以本 spec 各 API 的可观察行为为准，实现细节不作约束。

#### Scenario: 并发追加事件不交错损坏
- **WHEN** 多个 goroutine 并发对同一 turn `append_turn_event`
- **THEN** 所有事件均落库、seq 无重复无空洞（正常自动分配路径）
