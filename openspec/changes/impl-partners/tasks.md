## 1. 配置持久化与 workspace（无 channel 依赖，先行）

- [ ] 1.1 实现 `internal/partners/config.go`：`PartnerConfig` 结构、`config.yaml` 读写（原子写：临时文件 + rename）、`auto_start` 意图语义（未显式给出保留磁盘值，全新默认 true）
- [ ] 1.2 实现 `internal/partners/slug.go`：显示名 slug 化、纯 CJK 回落 `partner-<sha1[:8]>` / `soul-<sha1[:8]>`、保留名（`workspace`/`media`/`sessions`/`_souls`）拒绝
- [ ] 1.3 实现 `internal/partners/workspace.go`：合成 scope 布局创建/打开（`workspace/` 下 KB、SOUL.md、skills、notebook、memory），验证可直接打开 Python v1.5.2 创建的 partner 目录
- [ ] 1.4 实现 `internal/partners/sessions.go`：`sessions/*.jsonl` 存取、archive/resume/branch/delete、history 查询

## 2. Manager 生命周期

- [ ] 2.1 实现 `PartnerManager`：list/create（slug 冲突 409）/get/destroy（stop → 清 `partner:<id>` cron → 删目录）
- [ ] 2.2 实现 start/stop：start 幂等返回既有实例；手动 stop 置 `auto_start=false`，任务取消每个最多等 5s
- [ ] 2.3 实现 stop_all（保留磁盘 auto_start 意图）与服务启动时自动拉起 `auto_start=true` 的 partner
- [ ] 2.4 实现 `reload_channels`：仅重建 listener、按 partner 串行化、失败记录 `last_reload_error`

## 3. Runtime 执行链与事件映射

- [ ] 3.1 实现 `runtime.go`：inbound → `ChatOrchestrator` chat turn；`UnifiedContext` 组装（session id `partner:<id>:<session_key>`、SOUL persona、skills manifest、KB 名单、`metadata.agent_identity`、不设 `wait_for_user_reply`）
- [ ] 3.2 实现三元工具策略（`enabled_tools`/`builtin_tools`/`mcp_tools` 经 `metadata.mcp_tools_filter`）
- [ ] 3.3 实现 per-session 串行锁与 backup LLM 重试（含 LLM 配置解析失败折叠进错误列表，失败回复 `Sorry, the turn failed: <最后一条错误>`）
- [ ] 3.4 实现 `mapping.go`：StreamEvent → outbound（RESULT 最终回复、terminator 文本兜底、narration `_progress`、tool hint ≤120 字符、`_stream_delta`/`_stream_end`/`_streamed` 去重）
- [ ] 3.5 实现 turn 完成后会话持久化（user 消息含 attachment、assistant 消息含事件 trace）与 session key 解析（默认 `<channel>:<chat_id>`、`session_key_override`）
- [ ] 3.6 实现 `commands.go`：斜杠命令拦截（`/help`、`/new`/`/clear`、`/branch`、`/stop`、`/sessions`、`/resume`、`/delete`、`/status`、`/history`、`/tool`）、`@bot` 后缀剥离、palette 数据

## 4. Channel 抽象与 outbound 管线

- [ ] 4.1 定义 `channels/channel.go`：`Channel`/`StreamingChannel` interface、`InboundMessage`/`OutboundMessage`、`IsAllowed` 语义（空 allow_from 拒绝并告警、`"*"` 放行）
- [ ] 4.2 实现 `bus/bus.go`（inbound/outbound 异步解耦）与 `channels/registry.go`（按名发现、内置优先、不可用 channel 记录原因——覆盖后置 channel 清单可启动）
- [ ] 4.3 实现 `router.go`：`_progress`/`_tool_hint` 过滤、`(channel, chat_id, _stream_id)` delta coalescing、`origin_message_id` 指纹抑制、指数退避重试（1s/2s/4s，`send_max_retries` 默认 3）
- [ ] 4.4 实现 `channels/schema.go`：channel 配置 schema（snake/camel 双键归一、顶层 legacy delivery 键读剥离写拒绝），服务 `GET /channels/schema` 与 422 校验

## 5. 首批四个 channel

- [ ] 5.1 feishu（`larksuite/oapi-sdk-go/v3`）：WS 长连接收 `im.message.receive_v1`、message_id LRU 去重、表情回应、文本/post/图片/文件解析与媒体下载、断线 5s 重连
- [ ] 5.2 feishu 出站：markdown 复杂度检测选 card/post/text、单卡至多一表格拆分、CardKit streaming card（`streaming_mode: true` + 递增 sequence + ≥0.5s 节流）
- [ ] 5.3 telegram（`mymmrac/telego`）：长轮询、`id|username` allow_from、markdown → Telegram-safe HTML、4000 字符切分（代码块不截断）、命令菜单、媒体组缓冲、typing、回复线程
- [ ] 5.4 telegram streaming：先发后编（`stream_edit_interval` 默认 0.6s、编辑期纯文本预览、`_stream_end` 渲染最终 HTML、按 `_stream_id` 键控）
- [ ] 5.5 slack（`slack-go/slack` Socket Mode）：握手 30s 超时显式报错、ack 后处理 `message`/`app_mention` 去重、表情回应、线程级 session key（`slack:<channel>:<thread_ts>`）、mrkdwn 渲染、文件 upload 单失败不阻断
- [ ] 5.6 discord（`bwmarrin/discordgo`）：Gateway 收事件（心跳/RECONNECT/INVALID_SESSION、5s 重连）、mention 过滤、附件下载（>20MB 占位）、typing、REST 429 重试、2000 字符切分
- [ ] 5.7 discord streaming：POST 创建 + 0.8s 节流 PATCH 编辑、超 2000 字符冻结当前消息开新消息续流

## 6. REST 面与 Web 聊天

- [ ] 6.1 实现 `internal/api/routers/partners.go` 目录/静态组：`GET ""`（channels 仅键名）、`/recent`、`/channels/schema`、`/tool-options`（排除 read/write_memory）、`/commands/palette`
- [ ] 6.2 实现 CRUD 与生命周期组：`POST ""`（soul 来源解析、channels 422、llm_selection 校验、avatar data:image ≤200k、assets 供给、start 默认 true）、`GET /{id}`（`include_secrets`）、`PATCH /{id}`（channels 变更触发 reload、失败 500 提示 stop/start）、start/stop/DELETE/channels/reload
- [ ] 6.3 实现秘密字段深度掩码 `maskSecrets`（`token`/`secret`/`password`/`api_key`/`apikey`/`encrypt_key` 大小写不敏感 → `***`）并接入全部响应路径
- [ ] 6.4 实现 soul 组（`/souls` CRUD、`/soul-sources`、`/{id}/soul`）与 `souls.go` 模板库（seed 五模板、旧 TutorBot seed 原地升级、用户条目不触碰）
- [ ] 6.5 实现资产组（`/{id}/assets` GET/POST/DELETE）与 `assets.go`（请求用户上下文解析来源、幂等复制、`{copied, errors}` 报告、删除名字校验）
- [ ] 6.6 实现历史与会话组（`/history`、`/sessions` + archive/resume/branch/delete）
- [ ] 6.7 实现 `POST /{id}/chat`（HTTP 一问一答）与 `POST /{id}/chat/execute-stream`（SSE：session/thinking/content/done/error）
- [ ] 6.8 实现 `/{partner_id}/ws` + `liveturn.go`：upgrade 前认证（4001）、懒启动语义（4003/4004、按 partner 加锁去重）、帧协议、LiveTurn 缓冲 + attach 重放（resuming/user_echo/缓冲帧）+ stop 广播、同 session_key 单 turn、base64 附件落盘与 413 限制、web session key 解析

## 7. 测试与验收（spec Scenario 落测试）

- [ ] 7.1 配置/生命周期 Scenario 测试：基线目录直开、CJK 稳定 id + 409、手动 stop 不复活、stop_all 保留意图
- [ ] 7.2 执行链 Scenario 测试（mock LLM）：backup 回落、`builtin_tools=["rag"]` 白名单收窄
- [ ] 7.3 事件映射与管线 Scenario 测试：`_streamed` 不重复投递、`send_progress=false` 不投 narration、空 allow_from 拒绝、退避 3 次后放弃
- [ ] 7.4 channel 单测（mock transport）：feishu 卡片选择与拆分、CardKit 流式定格、telegram 4000 切分与 mention 策略、slack 线程隔离与握手超时报错、discord 溢出开新消息与 Gateway 重连；markdown 渲染器以基线输出为 golden case
- [ ] 7.5 REST/WS Scenario 测试：默认掩码、非法 channels 422、attach 刷新续流、auto_start=false 4003、资产部分失败报告、旧 seed 升级；对 `/api/v1/partners` 全路由跑 golden spec contract test（协议对等验收）
- [ ] 7.6 数据兼容验收：Go 写入的 partner 数据回切 Python 后端可正常打开（acceptance.md 3.2 反向兼容）
- [ ] 7.7 真实账号 e2e（M4 验收矩阵，手工/半自动）：feishu、telegram、slack、discord 各完成收发消息、流式回复、附件三项演练并记录结果
- [ ] 7.8 后置 channel 启动验收：配置含 `telegram`+`zulip` 的 partner 可启动，zulip 记为不可用且原因可查询
