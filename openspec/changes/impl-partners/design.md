## Context

Partner 子系统是基线中体量最大的外围域之一：`deeptutor/services/partners/`（manager/runtime/workspace/sessions/commands/scope）+ `deeptutor/partners/channels/`（BaseChannel 抽象与 19 个 channel 实现）+ `deeptutor/partners/bus/`、`deeptutor/partners/config/` + `deeptutor/api/routers/partners.py`（约 31 条 REST + partner WS）。Go 版的关键约束：

- partner 无独立引擎，每条消息进入 `ChatOrchestrator` 同一执行链（依赖 `impl-capability-chat-solve`、`impl-turn-runtime` 已交付的 orchestrator / UnifiedContext / turn 生命周期）；
- `data/partners/<id>/` 磁盘布局与基线逐一对等，Python v1.5.2 创建的数据免迁移直接打开；
- `/api/v1/partners` REST/SSE/WS 协议逐字段对等，`web/` 前端零改动；
- 首批四个 channel：feishu / telegram / slack / discord；其余后置，`matrix-e2e` 不迁移（有意差异，登记于 acceptance.md 第 1 节「不在验收范围」）。
- 基线 Python 用 asyncio task 组织 runner/listener/router；Go 版用 goroutine + channel + `context.Context` 取消树表达同一结构。

## Goals / Non-Goals

**Goals:**

- partner manager（配置持久化、生命周期、auto_start 意图语义）与 runtime（inbound → chat turn → outbound）完整迁移；
- channel adapter 抽象（`Channel` interface）+ MessageBus + outbound 分发管线（过滤/coalescing/指纹抑制/退避重试）；
- feishu / telegram / slack / discord 四个 channel 达到基线功能面（含流式编辑）；
- `/api/v1/partners` 约 31 条 REST + `/{id}/ws`（LiveTurn 缓冲 + attach 重放）+ SSE 执行流；
- soul 模板库与资产供给。

**Non-Goals:**

- 后置 channel（`weixin`、`mochat`、`matrix`、`dingtalk`、`msteams`、`napcat`、`zulip`、`email`、`mattermost`、`wecom`、`whatsapp`、`qq`）本 change 不实现，仅保证「配置存在时可启动 + 记录不可用原因」；
- `matrix-e2e` 不迁移（有意差异；Go 生态无成熟 libolm 绑定，后置的 matrix channel 仅支持非 E2E 房间）；
- chat agent loop、turn runtime、KB/skill/notebook 本体（分别由上游 change 交付，这里只消费）；
- 前端 Partners 页面（零改动复用）。

## Decisions

### D1. Go 包结构

```
internal/partners/
├── manager.go          # PartnerManager：list/create/get/start/stop/destroy/reload_channels、auto-start
├── config.go           # PartnerConfig 读写（config.yaml 原子写、auto_start 意图、snake/camel 兼容）
├── slug.go             # 显示名 slug 化 + CJK sha1 回落 + 保留名校验
├── workspace.go        # 合成 scope 布局创建/打开（对应基线 workspace.py + scope.py）
├── runtime.go          # PartnerRuntime：runner 循环、per-session 锁、backup LLM 重试
├── liveturn.go         # LiveTurn：帧缓冲、attach 重放、stop 广播（Web WS 用）
├── sessions.go         # sessions/*.jsonl 会话存取（archive/resume/branch/delete/history）
├── commands.go         # 斜杠命令注册表与执行（/help /new /tool ...）+ palette
├── souls.go            # _souls.yaml 模板库（seed、旧 seed 原地升级）
├── assets.go           # 资产供给（KB/skill/notebook 复制、报告聚合）
├── mapping.go          # StreamEvent → OutboundMessage 映射（RESULT/CONTENT/narration/tool hint/_stream_delta）
├── bus/
│   └── bus.go          # MessageBus：inbound/outbound 异步队列（Go channel 实现）
├── router.go           # OutboundRouter：过滤、coalescing、指纹抑制、指数退避重试
└── channels/
    ├── channel.go      # Channel interface + InboundMessage/OutboundMessage 类型
    ├── registry.go     # 按名注册与发现（内置优先）、不可用原因记录
    ├── schema.go       # channel 配置 schema（驱动 GET /channels/schema 与 422 校验）
    ├── feishu.go       # + feishu_card.go（markdown→card、CardKit streaming）
    ├── telegram.go     # + telegram_html.go（markdown→Telegram-safe HTML、4000 切分）
    ├── slack.go        # + slack_mrkdwn.go
    └── discord.go      # + discord_gateway.go（Gateway 心跳/重连）
internal/api/routers/partners.go   # /api/v1/partners REST + SSE
internal/api/ws/partner_ws.go      # /{partner_id}/ws（gorilla/websocket upgrade）
```

### D2. 关键接口签名

Channel 抽象（对等基线 `BaseChannel`）与 outbound 管线：

```go
// InboundMessage / OutboundMessage 的 Meta 携带 _wants_stream、_stream_id、
// _progress、_tool_hint、_streamed、origin_message_id、session_key_override 等基线内部键。
type InboundMessage struct {
    Channel   string
    ChatID    string
    SenderID  string
    Text      string
    Media     []MediaRef
    Meta      map[string]any
}

type OutboundMessage struct {
    Channel string
    ChatID  string
    Text    string
    Files   []string
    Meta    map[string]any
}

// Channel 是 adapter 接口；send 失败必须返回 error 以触发 OutboundRouter 统一重试。
type Channel interface {
    Name() string
    Start(ctx context.Context, inbound chan<- InboundMessage) error // 长驻监听，ctx 取消即退出
    Stop(ctx context.Context) error
    Send(ctx context.Context, msg OutboundMessage) error
    IsAllowed(senderID string) bool // allow_from 语义：空列表拒绝所有，"*" 放行
}

// StreamingChannel 可选实现：就地编辑流式，缓冲按 streamID（{turn_id}:{call_id}）键控。
type StreamingChannel interface {
    Channel
    SendDelta(ctx context.Context, chatID, streamID, delta string, meta map[string]any) error
    EndStream(ctx context.Context, chatID, streamID string) error
}

// ChannelFactory 注册进 registry；构建期校验配置（如空 allow_from 报错）。
type ChannelFactory func(cfg map[string]any, deps ChannelDeps) (Channel, error)

// OutboundRouter：过滤 → coalescing → 指纹抑制 → 重试投递。
type OutboundRouter struct { /* ... */ }
func (r *OutboundRouter) Dispatch(ctx context.Context, msg OutboundMessage) // 退避 1s/2s/4s，send_max_retries 默认 3
```

Manager / Runtime / LiveTurn：

```go
type PartnerManager struct { /* ... */ }
func (m *PartnerManager) Start(ctx context.Context, id string) (*PartnerRuntime, error) // 幂等
func (m *PartnerManager) Stop(ctx context.Context, id string, persistIntent bool) error // 手动 stop 置 auto_start=false
func (m *PartnerManager) StopAll(ctx context.Context) error                             // 保留磁盘 auto_start
func (m *PartnerManager) Destroy(ctx context.Context, id string) error                  // stop → 清 cron → 删目录
func (m *PartnerManager) ReloadChannels(ctx context.Context, id string) error           // 仅重建 listener

type PartnerRuntime struct { /* runner goroutine、sessionLocks map[string]*sync.Mutex、router、channels */ }
func (rt *PartnerRuntime) HandleInbound(ctx context.Context, msg InboundMessage)        // 斜杠命令拦截 → chat turn

// LiveTurn 与 socket 解耦：断线不终止 turn，attach 重放缓冲帧。
type LiveTurn struct { /* frames []json.RawMessage、subscribers、cancel func */ }
func (t *LiveTurn) Attach(conn *websocket.Conn) error // 重放 resuming/user_echo/缓冲 stream_event 后续流
func (t *LiveTurn) Stop()                             // 取消 turn 并广播 stopped
```

### D3. Channel SDK 选型

| Channel | 选型 | 理由 | 备选与否决理由 |
| --- | --- | --- | --- |
| feishu | `larksuite/oapi-sdk-go/v3` | 官方 SDK；内置开放平台 WebSocket 长连接（`larkws`，对等基线 `im.message.receive_v1` 免公网接入）；覆盖 IM 收发、媒体上传下载、CardKit streaming card API | 手写 WS + REST：协议面大（鉴权/心跳/加解密），无必要 |
| telegram | `mymmrac/telego` | 完整覆盖当前 Bot API（长轮询、edit_message_text、媒体组、命令菜单），持续维护，支持自定义 HTTP client（`proxy` 配置需要） | `go-telegram-bot-api/v5`：2022 年后基本停更、Bot API 版本落后，otherwise 事实标准；若 telego 出现阻塞性问题以此为回退 |
| slack | `slack-go/slack`（含 `socketmode` 子包） | 社区事实标准；Socket Mode（`bot_token`+`app_token`）开箱即用，事件 ack、文件 upload、reactions 全覆盖；握手超时经自定义 dialer 实现基线 30s 语义 | 手写 Socket Mode：协议繁琐，无必要 |
| discord | `bwmarrin/discordgo` | 社区事实标准；Gateway（IDENTIFY/心跳/RECONNECT/INVALID_SESSION）与 REST v10、429 `retry_after` 处理内建；`intents` 直接可配 | disgo：功能相当但生态与文档弱于 discordgo |

后置 channel 清单（本 change 不实现，registry 记录「不可用 + 原因」）：`weixin`、`mochat`、`matrix`、`dingtalk`、`msteams`、`napcat`、`zulip`、`email`、`mattermost`、`wecom`、`whatsapp`、`qq`。其中 `matrix-e2e` 明确不迁移（有意差异）：基线依赖 `matrix-nio[e2e]`/libolm，Go 无成熟等价物；后置 matrix channel 仅支持非 E2E 房间。

### D4. REST/WS 落位（gin + gorilla/websocket）

- `/api/v1/partners` 以 gin route group 组织，整组挂 `require_admin` 中间件（来自 `impl-multi-user-auth` 的守卫）；SSE 用 gin 的 `c.Stream` + `text/event-stream`。
- `/{partner_id}/ws` 在 gin handler 内以 gorilla/websocket upgrade；认证在 upgrade 前完成（失败 close 4001），auto_start=false 未运行时 4003、不存在 4004；懒启动按 partner `sync.Mutex` 去重。
- 秘密字段掩码：对响应 JSON 做深度遍历，字段名（大小写不敏感）含 `token`/`secret`/`password`/`api_key`/`apikey`/`encrypt_key` 时替换为 `***`；实现为独立的 `maskSecrets(v any) any` 纯函数，单测覆盖嵌套 map/slice。
- channels 校验 schema 驱动（`channels/schema.go` 单一事实源，同时服务 `GET /channels/schema` 与 422 校验），snake_case/camelCase 双键兼容在 schema 层归一。

### D5. 与 Python 基线文件映射

| Python 基线 | Go 落位 |
| --- | --- |
| `deeptutor/services/partners/manager.py` | `internal/partners/manager.go` + `config.go` + `slug.go` |
| `deeptutor/services/partners/runtime.py` | `internal/partners/runtime.go` + `mapping.go` + `liveturn.go` |
| `deeptutor/services/partners/workspace.py`、`scope.py` | `internal/partners/workspace.go` |
| `deeptutor/services/partners/sessions.py` | `internal/partners/sessions.go` |
| `deeptutor/services/partners/commands.py` | `internal/partners/commands.go` |
| `deeptutor/partners/bus/` | `internal/partners/bus/bus.go` |
| `deeptutor/partners/channels/base.py`、`registry.py`、`manager.py` | `internal/partners/channels/channel.go`、`registry.go` + `router.go` |
| `deeptutor/partners/channels/{feishu,telegram,slack,discord}.py` | `internal/partners/channels/{feishu,telegram,slack,discord}.go` |
| `deeptutor/partners/config/` | `internal/partners/channels/schema.go` |
| `deeptutor/api/routers/partners.py` | `internal/api/routers/partners.go` + `internal/api/ws/partner_ws.go` |
| soul seed 模板（基线内置） | `internal/partners/souls.go` + 嵌入模板（`embed.FS`） |

### D6. 并发模型

- 每个 partner runtime 一个根 `context.Context`；listener/runner/router 均为该 ctx 下的 goroutine，stop = cancel + 每任务 5s 等待。
- per-session 串行：`map[sessionKey]*sync.Mutex`（惰性创建），保证同一 session_key 消息串行执行。
- LiveTurn 帧缓冲用 `sync.RWMutex` 保护的 slice，attach 时先在锁内 snapshot 再订阅，避免丢帧/重帧。
- outbound coalescing：router 内按 `(channel, chatID, streamID)` 维护 pending delta，定时器触发合并投递。

## Risks / Trade-offs

- [telego 相对年轻，长期维护性存在不确定] → 选型已留回退（`go-telegram-bot-api`）；channel 实现面向自有 `Channel` interface，SDK 只在 adapter 内部使用，替换成本局限于单文件。
- [feishu CardKit streaming card API 变动快] → streaming 失败时降级为「整卡替换」或非流式发送；spec 只要求打字机效果与最终定格，节流参数可配。
- [四 channel 的真实账号 e2e 无法进 CI] → CI 内 channel 层用 mock transport 测消息映射/切分/重试；真实账号 e2e 作为 M4 手工验收项（acceptance.md M4 矩阵），tasks.md 中单列。
- [LiveTurn 缓冲无上限会占内存] → 单 turn 帧缓冲设上限（如 5k 帧），超限丢弃最早的 stream_event 但保留 user_echo 与终态帧；attach 语义降级为「尽力重放」。
- [基线 channel 配置存在 snake/camel 混用的历史数据] → schema 层双键归一 + 读取时剥离顶层 legacy delivery 键，写入时拒绝（422），与 spec 一致。
- [markdown → 各 IM 方言渲染是行为对等的长尾] → 渲染器（card/HTML/mrkdwn）以基线输出为 golden case 做表驱动单测。
