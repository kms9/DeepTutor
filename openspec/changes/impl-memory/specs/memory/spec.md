# memory Delta Specification

## ADDED Requirements

### Requirement: 三层目录布局与数据兼容
系统 SHALL 在 per-user memory 根目录下维护与基线一致的布局：`trace/<surface>/<YYYY-MM-DD>.jsonl`（L1，UTC 日切）、`L2/<surface>.md` 与 `L2/<surface>.meta.json`、`L3/{recent,profile,scope,preferences}.md` 与对应 `.meta.json`、`backup/<timestamp>/`（v1 迁移归档）。surface 集合固定为 `chat`、`notebook`、`quiz`、`kb`、`book`、`partner`、`cowriter`；L3 slot 固定为 `recent`、`profile`、`scope`、`preferences`。指向既有基线数据目录启动时 SHALL 直接续写，SHALL NOT 要求任何格式迁移。根路径 SHALL 经路径服务在调用时解析（多用户上下文感知）；partner 轮次 SHALL 支持将 memory 路径覆盖到 owner（admin）的 scope，而其他服务仍留在 partner scope。

#### Scenario: 老数据免迁移续写
- **WHEN** deeptutor-go 首次指向基线 Python 生成的 `data/.../memory/` 目录启动并追加一条 chat trace 事件
- **THEN** 事件追加到既有 `trace/chat/<今日>.jsonl` 文件尾部
- **AND** 既有 L2/L3 文档可被完整读取与解析

#### Scenario: 遗留 v1 文件归档
- **WHEN** memory 根目录下存在 v1 布局文件（`PROFILE.md` 或 `SUMMARY.md`）
- **THEN** 启动迁移将根目录下除 `trace`/`L2`/`L3`/`backup` 外的松散文件移入 `backup/<UTC 时间戳>/`
- **AND** 该迁移幂等，重复启动不再触发

### Requirement: L1 trace append-only 语义
L1 事件 SHALL 为 JSONL 单行记录，字段为 `id`（`<surface>:<26 位 Crockford-base32 ULID>`，前 10 字符编码毫秒时间戳）、`ts`（UTC ISO 8601）、`surface`、`kind`、`payload`、可选 `session_id` / `turn_id`。追加 SHALL 按 surface 串行化（防止并发交错写坏行），且 SHALL 永不向调用方抛错——任何失败仅记日志吞掉，绝不影响产生事件的业务面。系统 SHALL 提供按时间顺序遍历（可按 `ts >= since` 过滤且跳过早于 cutoff 日期的文件）、按 id 集合跨文件回查、计数与最新时间戳查询；坏行 SHALL 跳过不中断遍历。

#### Scenario: trace 写失败不影响业务轮次
- **WHEN** chat 轮次结束时 trace 目录不可写
- **THEN** 追加操作记录 warning 日志后静默返回
- **AND** 轮次正常完成

### Requirement: L2/L3 文档模型与条目寻址
L2/L3 文档 SHALL 为带结构约定的 markdown：条目携带 `m_<ULID>` 形式的 entry id，条目来源以 footnote 引用记录，引用值可为 L1 trace id（`<surface>:<ULID>`）、L2 entry id、snapshot ref（`<surface>:<实体 id>`）或 L3 使用的裸 surface 名（白名单）。文档写入 SHALL 原子（临时文件 + rename）。系统 SHALL 支持按 layer+key 读原文/解析文档、整篇覆盖保存（workbench 手工编辑）、按 entry id 删除单条；对文档的结构化修改以 ops（Add/Edit 等）表达并原子应用，返回逐条接受/拒绝的报告。

#### Scenario: 删除单条记忆条目
- **WHEN** 调用 `DELETE /api/v1/memory/doc/L2/chat/entry/{entry_id}` 且该 id 存在
- **THEN** 该条目从 `L2/chat.md` 中移除并原子回写
- **AND** id 不存在时返回 404

### Requirement: consolidator 四操作
系统 SHALL 提供 LLM 驱动的四种 consolidator 模式：`update`（分 chunk 增量提取新 L1/L2 事实并入文档，凭 `.meta.json` 的 seen-ids 集合只处理新增源）、`audit`（对照原始证据对文档做行级核对与修正）、`dedup`(对全文做迭代式行级去重，可指定 `iterations`）、`merge`（合并整理）。所有模式产出的条目 SHALL 携带指向来源（trace id / L2 entry id）的引用；引用池校验不通过的 op SHALL 被拒绝。prompts SHALL 按 `{en,zh}` 双语提供，可指定 `llm_selection`（profile_id + model_id）覆盖默认模型。`L3/preferences.md` SHALL NOT 被 `update`/`audit` 自动 consolidate（仅允许 `dedup`/`merge`）——它只由 `write_memory` 工具写入。

#### Scenario: preferences 拒绝自动 consolidate
- **WHEN** 客户端 `POST /api/v1/memory/runs/start` 请求 `layer=L3, key=preferences, mode=update`
- **THEN** 返回 405 且说明 preferences 由 `write_memory` 工具维护

#### Scenario: update 只消费新增事件
- **WHEN** 对 `L2/chat` 连续执行两次 `update` 而两次之间无新 L1 事件
- **THEN** 第二次运行不产生新增条目（seen-ids 已覆盖全部来源）

### Requirement: consolidator run 生命周期
长耗时 consolidator 工作 SHALL 由 run manager 承载：`POST /runs/start` 校验 layer/key 后启动一个 run 并返回 run 句柄；同一文档同时只允许一个活动 run（冲突返回 409）。run SHALL 在客户端断开后继续执行；事件带单调 `seq`，客户端可经 `GET /runs/{id}/events?since=N`（SSE）从任意游标重放缓冲事件并阻塞等待新事件直至 run 终态。系统 SHALL 支持 `GET /runs/{id}`（状态查询）、`GET /runs?layer=&key=`（活动+近期 run 列表）、`POST /runs/{id}/cancel`（协作式取消，非活动返回 409）、`POST /runs/{id}/undo`（回滚该 run 最近一次文档写入，维护 undo 栈；无可回滚返回 409）。基线的 legacy 端点 `POST /doc/{layer}/{key}/{update|audit|dedup}` SHALL 保留为薄包装：启动 run 并将其事件以 SSE 内联流式返回。

#### Scenario: 刷新页面后重连 run
- **WHEN** update run 进行中客户端断开，随后以 `since=<已见最大 seq>` 重连 events 端点
- **THEN** 服务端重放 seq 之后的缓冲事件并继续推送新事件直至 run 结束

### Requirement: memory REST 工作台面
系统 SHALL 提供与基线一致的 `/api/v1/memory` 工作台路由组（约 27 条）：总览组（`GET /overview` 返回全部 11 份文档的存在性/更新时间/条目数/L1 backlog 及 backup 列表；`GET /backup`）；文档组（`GET/PUT /doc/{layer}/{key}`、`GET /doc/{layer}/{key}/lines` 行号视图、`DELETE .../entry/{id}`、`POST .../reset` 删除文档与 meta sidecar（活动 run 存在时 409）、`POST .../apply` 提交先前预览的 ops（preferences 405））；解析组（`GET /resolve_entry/{entry_id}` 按固定 surface 顺序扫描 7 份 L2 定位条目归属，首个命中即返回，未命中 404）；run 组（见上一 Requirement）；设置组（`GET/PUT /settings` 读写 `memory:` 设置子树，defaults 合并返回）；trace 组（`GET /trace/{surface}` 分页、`DELETE /trace/{surface}`、`DELETE /trace/{surface}/day/{date}`）；snapshot 组（见下一 Requirement）。layer/key 校验 SHALL 与基线一致（未知 surface/slot 404，非法 layer 400）。

#### Scenario: overview 报告 backlog
- **WHEN** `L2/chat.md` 上次更新后又产生了 5 条 chat trace 事件，客户端请求 `GET /overview`
- **THEN** `chat` 行的 `backlog=5`
- **AND** 不存在的文档行返回 `exists=false` 且 backlog 为全部事件数

### Requirement: snapshot 工作区镜像
系统 SHALL 为每个 surface 提供 snapshot 视图：`GET /snapshot/{surface}` 实时从 workspace 派生当前实体列表并附带与上次持久化状态的 `pending_changes` 差异；`POST /snapshot/{surface}/refresh` 将差异提交入 `changes.jsonl` 并更新 `last_refresh`；`GET /snapshot/{surface}/changes` 分页读取变更日志；`DELETE /snapshot/{surface}/changes` 清空。

#### Scenario: 刷新提交待定变更
- **WHEN** 用户新建了一个 notebook 后调用 `POST /snapshot/notebook/refresh`
- **THEN** 新实体作为变更记录写入该 surface 的 `changes.jsonl`
- **AND** 随后的 `GET /snapshot/notebook` 中 `pending_changes` 为空

### Requirement: read_memory 存储语义
`read_memory` 工具读取 SHALL 等价于按 `recent`、`profile`、`scope`、`preferences` 固定顺序拼接非空 L3 文档原文（`\n\n---\n\n` 分隔，尾部换行）；四份全空时返回固定的"暂无记忆，可从 Memory 页构建"提示文本。

#### Scenario: 空记忆提示
- **WHEN** 新用户（无任何 L3 文档）触发 `read_memory`
- **THEN** 返回固定的 no-memory 提示文本而非空串

### Requirement: write_memory 存储语义
`write_memory` 的写入 SHALL 仅作用于 `L3/preferences.md`：`op=add` 在 `Preferences` 节追加新条目，`op=edit` 按 `target_id` 改写既有条目；两者的来源引用 SHALL 设为调用方传入的本轮 L1 trace id。缺 `target_id` 的 edit、未命中的 target SHALL 产生 `accepted=false` 报告。写入 SHALL 与其他写路径共用 per-file 异步锁并原子落盘；`reason` 仅记日志供 workbench 观测。

#### Scenario: edit 缺 target_id 被拒
- **WHEN** `write_memory` 以 `op=edit` 且无 `target_id` 到达存储层
- **THEN** 返回 `accepted=false, reason="edit requires target_id"` 且文件未被修改

### Requirement: 并发写保护
对同一 L2/L3 文档的全部写路径（overwrite、entry 删除、consolidator、apply ops、preference 写入）SHALL 经同一 per-path 互斥锁串行化，杜绝并发写导致的文档撕裂；trace 追加锁按 surface 独立。

#### Scenario: consolidate 与手工保存并发
- **WHEN** `L2/chat` 的 update run 正在写文档时用户提交 `PUT /doc/L2/chat`
- **THEN** 两次写按锁顺序串行完成，最终文件为其中一次完整写入的内容
