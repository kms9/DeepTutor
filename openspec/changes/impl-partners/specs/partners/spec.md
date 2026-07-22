## ADDED Requirements

### Requirement: Partner 配置持久化与数据布局兼容
系统 SHALL 将每个 partner 持久化在 `data/partners/<partner_id>/` 下（anchored 到 admin workspace root），布局与基线逐一对等：`config.yaml`（配置）、`sessions/`（`*.jsonl` 会话）、`media/<channel>/`（媒体缓存）、`workspace/`（合成 scope root，其下为 chat 用户 workspace 的完整克隆布局，含 `knowledge_bases/` 与 `user/workspace/SOUL.md`、`skills/`、`notebook/`、`memory/`）。`config.yaml` 字段 SHALL 至少包含：`name`、`description`、`channels`、`llm_selection`、`backup_llm_selection`、`model`（legacy）、`language`、`emoji`、`color`、`avatar`（内联 data URL）、`soul_origin`、`enabled_tools`、`builtin_tools`、`mcp_tools`、`auto_start`。写入 SHALL 使用原子写（临时文件 + rename）。`auto_start` 是生命周期意图而非普通配置字段：保存配置时未显式给出 auto_start SHALL 保留磁盘上的既有值，全新 partner 默认 `true`。partner id 与 soul id SHALL 由显示名 slug 化生成（ASCII 小写字母数字，其余字符折叠为连字符；纯 CJK 名回落为 `partner-<sha1[:8]>` / `soul-<sha1[:8]>` 的稳定哈希 id），保留名 `workspace`、`media`、`sessions`、`_souls` SHALL NOT 用作 partner id。

#### Scenario: 打开基线创建的 partner 目录
- **WHEN** Go 版指向一个由 Python v1.5.2 创建的 `data/partners/my-bot/` 启动并列出 partners
- **THEN** `my-bot` 出现在列表中，`config.yaml` 全部字段被正确读取，SOUL.md、sessions、KB 副本可被 runtime 直接使用

#### Scenario: 纯 CJK 名字获得稳定 id
- **WHEN** 用名字「数学助手」创建 partner 两次
- **THEN** 第一次生成形如 `partner-xxxxxxxx` 的稳定 id；第二次因同名 slug 相同而返回 409 冲突

### Requirement: Partner 生命周期管理
系统 SHALL 提供 partner 的 start/stop/destroy/auto-start 生命周期。start SHALL 创建 runtime 实例（runner 循环、outbound router、每个启用 channel 一个 listener 任务），已在运行时幂等返回既有实例。stop SHALL 取消全部任务（每个任务最多等待 5s）、停掉 channel，手动 stop SHALL 将 `auto_start` 置 false；进程整体关闭（stop_all）SHALL 保留磁盘上的 auto_start 意图，使宿主重启后同一批 partner 回来。服务启动时 SHALL 自动拉起所有 `auto_start=true` 的 partner。destroy SHALL 先 stop、清理该 partner 名下 cron job（owner 为 `partner:<id>`）、然后删除整个 partner 目录。`reload_channels` SHALL 只重建 channel listener（runner 与 router 不动），按 partner 串行化，失败时 listener 保持下线并把错误记录到 `last_reload_error` 供 UI 展示。

#### Scenario: 手动停止后不随服务重启复活
- **WHEN** 用户对运行中的 partner 调 `POST /{id}/stop`，随后重启 DeepTutor
- **THEN** 该 partner 的 `auto_start` 已被置 false，重启后不自动启动

#### Scenario: 服务关闭保留启动意图
- **WHEN** `auto_start=true` 的 partner 正在运行时进程收到关闭信号
- **THEN** stop_all 停止实例但磁盘 `auto_start` 仍为 true，下次启动自动拉起

### Requirement: 消息进入 ChatOrchestrator 同一执行链
系统 SHALL 令 partner 无独立引擎：每条 inbound 消息成为一个由 `ChatOrchestrator` 执行的 chat turn（与产品 chat 完全同一 agent loop），并运行在该 partner 的合成用户 scope 内，使 rag/skills/notebook/memory 等工具原生解析到 partner workspace。构造 `UnifiedContext` 时 SHALL：以 `partner:<id>:<session_key>` 为 session id；注入 SOUL.md 内容为 persona；注入 partner scope 下的 skills manifest 与 KB 名单；`metadata.agent_identity` 携带 partner 的 name/description 以替换系统提示中的产品身份；不设置 `wait_for_user_reply`（ask_user 暂停时挂起的问题即回复，用户下一条 IM 消息开启新 turn）。工具策略 SHALL 遵循三元语义：`enabled_tools=None` 表示全部用户可开工具、`[]` 表示无、列表为白名单；`builtin_tools=None` 表示不设门控（按各自 context 条件挂载）、列表为白名单；`mcp_tools` 经 `metadata.mcp_tools_filter` 传入管线。同一 session_key 的消息 SHALL 串行执行（per-session 锁）；turn 失败且配置了 `backup_llm_selection` 且与主选择不同时 SHALL 用 backup 重试一次，仍失败时回复 `Sorry, the turn failed: <最后一条错误>`。LLM 配置解析失败（如未配置激活模型）SHALL 折叠进错误列表并触发 backup 重试，SHALL NOT 以裸 Internal error 逃逸。

#### Scenario: 主模型失败回落 backup
- **WHEN** partner 配置了 `backup_llm_selection`，一条消息在主模型上 turn 失败且无输出
- **THEN** 系统用 backup 选择重跑该 turn 一次，成功则回复 backup 结果

#### Scenario: 工具白名单收窄内置工具面
- **WHEN** partner 的 `builtin_tools` 配置为 `["rag"]`
- **THEN** 该 partner 的 turn 中仅 `rag` 内置工具按其 context 条件挂载，memory/web_fetch 等其余内置工具不挂载

### Requirement: 事件到 IM 的映射与会话持久化
系统 SHALL 按基线契约把 chat loop 的 StreamEvent 映射到 IM 投递：`RESULT`（`metadata.response`）为最终回复；`CONTENT` 且 `call_kind=llm_final_response` 的文本累积为 terminator 文本，最终回复为空时以其代替；narration 轮（`call_role=narration` 完成时）在 IM channel 且 `send_progress` 开启时作为 `_progress` 消息投递；`TOOL_CALL` 在 `send_tool_hints` 开启时渲染为单行提示（`⚙ tool(args…)`，上限 120 字符）。开启 streaming 的 channel（inbound 带 `_wants_stream`，且要求 send_progress 同时开启）SHALL 将每轮文本以 `_stream_delta` 消息实时发布，`_stream_id` 为 `{turn_id}:{call_id}`；轮完成时发布 `_stream_end`；当实时流出的文本与最终回复一致时最终 outbound 标记 `_streamed`，channel 层 SHALL NOT 重复发送。每个 turn 完成后 SHALL 向 session store 追加 user 消息（含 attachment 记录）与 assistant 消息（含该 turn 的事件 trace，用于 Web 端刷新后重建活动面板）。session key 默认 `"<channel>:<chat_id>"`，channel 可用 `session_key_override` 覆盖（如 Slack 线程级会话）。

#### Scenario: 流式回复不重复投递
- **WHEN** telegram 配置 `streaming=true` 且回复文本已通过 `_stream_delta` 完整流出
- **THEN** 最终 outbound 带 `_streamed` 标记，channel 不再发送第二条相同消息

#### Scenario: 关闭 progress 后 narration 不投递
- **WHEN** 某 channel 配置 `send_progress=false`
- **THEN** narration 文本与流式增量都不投递到该 channel，仅最终回复送达

### Requirement: Channel adapter 抽象与 outbound 分发管线
系统 SHALL 定义 channel adapter 接口，语义与基线 `BaseChannel` 对等：`start()`（长驻监听）、`stop()`、`send(msg)`（失败必须抛错以触发统一重试）、可选 `send_delta(chat_id, delta, metadata)`（就地编辑流式，按 `_stream_id` 而非仅 chat_id 键控缓冲）、`is_allowed(sender_id)`（`allow_from` 空列表拒绝所有并告警，含 `"*"` 放行所有）。inbound/outbound 经异步 MessageBus 解耦。outbound 分发器 SHALL：按 `_progress`/`_tool_hint` 与 channel 的 `send_progress`/`send_tool_hints` 生效标志过滤；对同一 `(channel, chat_id, _stream_id)` 的连续 `_stream_delta` 做合并（coalescing）以降低编辑 API 调用频率；对同一 `origin_message_id` 的完全重复回复做指纹抑制；发送失败按指数退避重试（默认 1s/2s/4s，`send_max_retries` 可配，默认 3）。channel 注册 SHALL 支持按名发现内置 channel 与外部插件（内置优先，不可被插件遮蔽），无法加载的内置 channel（缺可选依赖）SHALL 记录错误原因而非静默消失。channel 的 delivery 与 streaming 开关配置 SHALL 同时接受 snake_case 与 camelCase 键（`send_progress`/`sendProgress` 等）。

#### Scenario: 空 allow_from 拒绝所有发送者
- **WHEN** 某 channel 配置 `allow_from: []` 且收到消息
- **THEN** 消息被拒绝且记录告警；若在 channel 构建期检测到空 allowFrom SHALL 报错提示设置 `["*"]` 或具体用户 id

#### Scenario: 发送失败指数退避后放弃
- **WHEN** channel `send` 连续抛错 3 次
- **THEN** 分发器按 1s、2s 间隔重试两次，第三次失败后记录异常并放弃该消息（不阻塞后续消息）

### Requirement: Feishu 与 Telegram channel（首批）
系统 SHALL 实现 feishu channel：使用 Feishu/Lark 开放平台 WebSocket 长连接接收事件（`im.message.receive_v1`，无需公网 IP），配置字段为 `app_id`、`app_secret`、`encrypt_key`、`verification_token`、`allow_from`、`react_emoji`（默认 THUMBSUP）、`group_policy`（`open`/`mention`，默认 mention）及 delivery/streaming 开关。SHALL 支持：消息去重（按 message_id 的有序 LRU 缓存）；收到消息后加表情回应；文本/富文本（post）/图片/文件消息解析与媒体下载；出站按内容复杂度自动选择消息格式——复杂 markdown（代码块/表格/标题/加粗/列表/超长文本）用 interactive card，否则用 post 或 text；单卡至多一个表格（超出拆分为多张卡）。streaming SHALL 使用 CardKit streaming card：先创建 `streaming_mode: true` 的卡片并发送，随后以递增 sequence 更新 `streaming_md` 元素实现打字机效果，更新间隔节流（≥0.5s）。断线 SHALL 自动重连（5s 间隔）。

系统 SHALL 实现 telegram channel：使用 Bot API 长轮询（无需 webhook/公网 IP），配置字段为 `token`、`allow_from`、`proxy`、`reply_to_message`、`group_policy`（`open`/`mention`，默认 mention）、连接池与 `stream_edit_interval`（默认 0.6s，≥0.1）及 delivery/streaming 开关。allow_from 匹配 SHALL 兼容基线的 `id|username` 双段写法。SHALL 支持：markdown → Telegram-safe HTML 渲染（代码块/行内代码保护、表格转 `<pre>` 等宽对齐、链接/加粗/斜体/删除线/列表），消息按 4000 字符切分；bot 命令菜单从 partner 命令注册表生成（`/start` + 命令面板）；媒体组缓冲、typing 指示、回复线程。streaming SHALL 采用「先发一条消息、随后按节流间隔 edit_message_text 就地编辑」模式，编辑期间 SHALL 以剥离 markdown 的纯文本预览呈现，`_stream_end` 时渲染最终 HTML；缓冲按 `_stream_id` 键控。

#### Scenario: 复杂 markdown 走卡片
- **WHEN** partner 回复包含代码块与表格
- **THEN** feishu channel 以 interactive card 发送，表格数超过单卡上限时拆分为多张卡片全部送达

#### Scenario: CardKit 流式打字机
- **WHEN** feishu 配置 `streaming: true` 且回复正在生成
- **THEN** 先出现一张流式卡片，其 markdown 元素随 `_stream_delta` 增量更新，`_stream_end` 后定格为完整文本

#### Scenario: 长回复分片发送
- **WHEN** 回复渲染后超过 4000 字符
- **THEN** 消息被切分为多条按顺序发送，代码块不被从中截断

#### Scenario: 群聊 mention 策略
- **WHEN** `group_policy=mention` 且群消息未 @ 该 bot
- **THEN** 消息被忽略；@ 了 bot 时剥离 mention 后进入执行链

### Requirement: Slack 与 Discord channel（首批）
系统 SHALL 实现 slack channel：使用 Socket Mode（`bot_token` + `app_token`），WebSocket 握手 SHALL 设超时（基线 30s）并在超时时显式报错（提示 WSS 可能被网络策略阻断而 HTTPS auth 正常）。配置含 `reply_in_thread`（默认 true）、`react_emoji`（默认 eyes）、`group_policy`（`open`/`mention`/`allowlist`，默认 mention）、`group_allow_from`、DM 策略（`dm.enabled`/`dm.policy`/`dm.allow_from`）。SHALL 支持：事件确认（ack）后处理 `message`/`app_mention`，忽略 bot/子类型消息并对 mention 场景去重（同一消息的 message 与 app_mention 只处理 app_mention）；对触发消息加表情回应；频道/群消息使用线程级 session key（`slack:<channel>:<thread_ts>`），DM 不用线程；出站 markdown SHALL 转换为 Slack mrkdwn（含表格转列表、残留加粗/标题修正），文件经 upload 接口发送（单文件失败不阻断文本投递）。

系统 SHALL 实现 discord channel：直连 Discord Gateway WebSocket（IDENTIFY/HELLO/心跳/RECONNECT/INVALID_SESSION 处理，断线 5s 重连）收事件，出站走 REST API（v10），配置含 `token`、`allow_from`、`intents`（默认 37377）、`group_policy`（`mention`/`open`，默认 mention）及 streaming 开关。SHALL 支持：忽略 bot 消息；guild 消息按 mention 策略过滤（mentions 数组或 `<@id>`/`<@!id>` 文本）；附件下载（>20MB 跳过并以占位文本告知）；回复引用；处理期间周期性 typing 指示；REST 429 按 `retry_after` 重试。消息按 2000 字符切分。streaming SHALL 采用「首次 POST 创建消息、随后按节流间隔（0.8s）PATCH 编辑」模式；文本超过 2000 字符时冻结当前消息、溢出部分开新消息继续流式；`_stream_end` 时完成最终渲染。

#### Scenario: 线程内会话隔离
- **WHEN** 同一 Slack 频道的两个线程分别向 bot 提问
- **THEN** 两个线程各自形成独立 session（session key 含 thread_ts），互不共享上下文，回复落在各自线程内

#### Scenario: Socket Mode 握手超时报错
- **WHEN** `auth_test` 成功但 WSS 连接 30s 未完成
- **THEN** channel 启动失败并给出可诊断的错误信息，触发 channel manager 的启动失败记录

#### Scenario: 流式中途溢出开新消息
- **WHEN** streaming 中累计文本超过 2000 字符
- **THEN** 当前消息定格为第一段，后续文本在新消息中继续流式，最终全部内容完整送达

#### Scenario: Gateway 断线自动重连
- **WHEN** Gateway 返回 op=7（RECONNECT）或连接异常断开
- **THEN** channel 在 5s 后重建连接并重新 IDENTIFY，期间不丢失已入队的 outbound 消息（由重试机制兜底）

### Requirement: 其余 channel 后置与 matrix-e2e 不迁移
以下 channel SHALL 作为后置 requirement 排期在首批四个 channel 之后（Go SDK/协议可用性逐个评估）：`weixin`、`mochat`、`matrix`、`dingtalk`、`msteams`、`napcat`、`zulip`、`email`、`mattermost`、`wecom`、`whatsapp`、`qq`。在其实现前，包含这些 channel 配置的 partner SHALL 可正常启动，未实现的 channel 记为「不可用 + 原因」，SHALL NOT 阻塞其他 channel 或 Web 聊天。`matrix-e2e`（matrix 端到端加密，基线经 `matrix-nio[e2e]`/libolm）明确标注**不迁移**（有意差异）：Go 版 matrix channel（后置）仅支持非 E2E 房间。

#### Scenario: 配置了未实现 channel 的 partner 可启动
- **WHEN** 某 partner 的 `channels` 同时含 `telegram` 与 `zulip`（后者未实现）
- **THEN** partner 正常启动，telegram listener 工作，zulip 记录为不可用且原因可查询

### Requirement: Partner 斜杠命令
系统 SHALL 在 runner 执行 turn 之前拦截斜杠命令（以 `/` + 字母开头），命令集与基线一致：`/help`、`/new`（别名 `/clear`，归档当前会话）、`/branch`（IM 上降级为归档 + 提示 Web 端分支）、`/stop`、`/sessions`（列会话）、`/resume <key>`、`/delete <key>`、`/status`（partner/会话/模型/工具状态）、`/history [n]`（1–50，默认 10）、`/tool [on|off <name>|reset]`（修改并持久化 `enabled_tools`）。命令 SHALL 支持 `@bot` 后缀剥离。命令结果直接作为回复返回，SHALL NOT 进入 agent loop。系统 SHALL 提供 `GET /commands/palette` 返回命令面板（command/description/arg_hint）。

#### Scenario: /tool off 持久化
- **WHEN** 用户在 IM 中发送 `/tool off web_search`
- **THEN** 回复更新后的工具清单，且 `config.yaml` 的 `enabled_tools` 持久化了该变更

### Requirement: /api/v1/partners REST 面
系统 SHALL 暴露 `/api/v1/partners` 路由（整组 admin 门控，同基线），按功能组约 31 条：

- Soul 模板库：`GET /souls`、`POST /souls`、`GET /souls/{soul_id}`、`PUT /souls/{soul_id}`、`DELETE /souls/{soul_id}`、`GET /soul-sources`（库 + 当前用户可见 persona）
- 目录/静态：`GET ""`（列表，channels 仅键名）、`GET /recent`、`GET /channels/schema`（schema 驱动的 channel 配置 UI 元数据）、`GET /tool-options`（排除 `read_memory`/`write_memory`）、`GET /commands/palette`
- CRUD 与生命周期：`POST ""`（创建：soul 来源 default/library/persona/custom 解析、channels 校验 422、llm_selection 校验、avatar 校验 data:image 且 ≤200k 字符、可选 assets 供给、`start` 默认 true）、`GET /{partner_id}`（`include_secrets` 查询参数控制原文/掩码）、`PATCH /{partner_id}`（channels 变更触发 reload_channels，失败返回 500 并提示 stop/start）、`POST /{partner_id}/start`（持久化 auto_start=true）、`POST /{partner_id}/stop`、`DELETE /{partner_id}`、`POST /{partner_id}/channels/reload`
- Soul 实例：`GET /{partner_id}/soul`、`PUT /{partner_id}/soul`
- 资产：`GET /{partner_id}/assets`、`POST /{partner_id}/assets`、`DELETE /{partner_id}/assets/{asset_type}/{name}`
- 历史与会话：`GET /{partner_id}/history`（session_key/session_id/limit）、`GET /{partner_id}/sessions`、`POST /{partner_id}/sessions/archive`、`POST /{partner_id}/sessions/resume`、`POST /{partner_id}/sessions/branch`、`POST /{partner_id}/sessions/delete`
- 聊天：`POST /{partner_id}/chat`、`POST /{partner_id}/chat/execute-stream`（SSE）

响应中的 channel 秘密字段（字段名含 `token`/`secret`/`password`/`api_key`/`apikey`/`encrypt_key`，大小写不敏感）SHALL 深度遍历掩码为 `***`，仅 `include_secrets=true`（编辑表单）返回原文。历史遗留的顶层 delivery 键（`send_progress`/`send_tool_hints` 及 camelCase 变体）SHALL 在读取时剥离、在写入 channels 时拒绝（422）。

#### Scenario: 默认响应掩码秘密
- **WHEN** 调 `GET /{id}` 不带 `include_secrets`
- **THEN** `channels.telegram.token` 等秘密字段值为 `***`，非秘密字段原样返回

#### Scenario: 创建时校验非法 channels
- **WHEN** `POST ""` 的 `channels` 不符合 channel 配置 schema
- **THEN** 返回 422 并携带字段级错误信息，partner 不被创建

### Requirement: Web 聊天（HTTP / SSE / WebSocket）与刷新可存活的流式 turn
系统 SHALL 提供三种 Web 聊天入口。HTTP `POST /{id}/chat` 一问一答；SSE `POST /{id}/chat/execute-stream` 流式输出 `session`/`thinking`/`content`/`done`/`error` 事件。WebSocket `/{partner_id}/ws` SHALL：连接前完成认证（失败 close 4001）；partner 未运行时按 auto_start 意图懒启动——`auto_start=false` 且非显式 start 时返回错误并以 4003 关闭（对应 HTTP 409 语义），不存在时 4004；并发连接触发的懒启动 SHALL 按 partner 加锁去重。帧协议与基线一致：客户端发 `{"content", "session_id"?, "chat_id"?, "session_key"?, "attachments"?}` 或 `{"action": "stop"|"attach"}`；服务端发 `{"type": "stream_event"|"content"|"done"|"error"|"proactive"|"resuming"|"user_echo"|"stopped"}`。Web turn SHALL 以实例持有的 LiveTurn 运行、与 socket 解耦：turn 期间断线不终止 turn；客户端重连发 `attach` 时 SHALL 重放缓冲的全部帧（含开启该 turn 的用户消息 echo）并续流；`stop` 取消进行中的 turn 并广播 `stopped`。同一 session_key 同时至多一个 turn（重复 start 返回既有 turn）。附件 SHALL 按 base64 落盘到 partner media 目录，尺寸受 chat attachment 策略限制（单文件/批量超限分别 413）。web session key 解析：显式 `session_key` 原样使用；否则 `web:<session_id>` / `web:<chat_id>` / 兜底 `partner:<partner_id>`。

#### Scenario: 页面刷新后续看流式回答
- **WHEN** Web 端在 turn 进行中刷新页面并重连后发送 `{"action":"attach"}`
- **THEN** 服务端回放 `resuming`、`user_echo` 与已缓冲的 `stream_event` 帧，随后继续实时推送直到 `content` + `done`

#### Scenario: 停用 partner 不被 Web 聊天悄悄唤醒
- **WHEN** partner `auto_start=false` 且未运行，Web 端直接连 WS 发消息
- **THEN** 返回错误「需要先启动 partner」并以 4003 关闭；显式 `POST /{id}/start` 后方可聊天

### Requirement: Soul 模板库与资产供给
系统 SHALL 维护共享 soul 模板库 `data/partners/_souls.yaml`：不存在时用内置默认模板 seed（companion、math-tutor、coding-assistant、research-helper、language-tutor）；加载时对「未被用户改动的旧版 seed」（内容命中已退役 seed 全文或 seed id 上仍含 TutorBot 字样）SHALL 原地升级为当前模板，用户自建/已编辑条目 SHALL NOT 触碰。资产供给 SHALL 在**请求用户**的上下文中解析来源（KB 经 `resolve_kb` 权限解析、skill 按用户自有 → 内置 → 管理员指派顺序解析、notebook 从用户 notebook 目录），然后把文件整体复制进 partner workspace（KB 整树、skill 整目录、notebook JSON + index 条目合并）；已存在的同名资产幂等跳过。供给结果 SHALL 返回 `{"copied": {...}, "errors": [{"type","name","error"}]}` 报告，单项失败不中断其余项。资产删除 SHALL 校验名字不含路径分隔符且不以 `.` 开头。

#### Scenario: 供给部分失败仍返回报告
- **WHEN** `POST /{id}/assets` 请求复制两个 KB，其中一个源目录缺失
- **THEN** 成功的 KB 出现在 `copied.knowledge_bases`，失败的以 `{type:"knowledge_base", name, error}` 出现在 `errors`，HTTP 状态仍为 200

#### Scenario: 旧 TutorBot seed 自动升级
- **WHEN** `_souls.yaml` 中 `math-tutor` 条目内容与退役 seed 全文一致
- **THEN** 下次加载时该条目被替换为当前模板；另一条被用户改写过的条目保持原样
