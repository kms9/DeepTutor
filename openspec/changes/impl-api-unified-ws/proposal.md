## Why

`/api/v1/ws` 是现有 `web/` 前端的唯一统一实时通道，前端零改动复用要求其 11 种客户端消息类型的请求/响应/事件推送语义与基线**字节级对等**。行为规格见 `docs/golang-req/openspec/specs/api-unified-ws/spec.md`；按 `docs/golang-req/openspec/ROADMAP.md`，本模块位于 **Wave 1（运行时内核）/ 里程碑 M1**，是 M1 退出条件「统一 WS 全消息类型行为对等；seq 连续；resume/cancel/regenerate/ask_user 场景全过」的协议面载体，也是「前端 Chat 页全功能可用」的直接入口。

## What Changes

- 新增 `/api/v1/ws` WebSocket 端点（gin handler 内经 gorilla/websocket upgrade）：accept 前鉴权（等价基线 `ws_require_auth`），用户身份绑定连接生命周期；单连接顺序处理多消息（长连接多路复用）。
- 新增 11 种消息类型处理：`message`/`start_turn`（起轮 + 自动订阅）、`ping`（pong 心跳）、`subscribe_turn`、`resume_from`、`subscribe_session`、`check_active_turn`（含重启陈旧轮置 cancelled）、`unsubscribe`、`cancel_turn`、`submit_user_reply`、`regenerate`（自动订阅 + `regenerate_busy`/`nothing_to_regenerate` 错误码）、`user_input`（全局 bus 注册表直通输入）。
- 新增帧级容错：JSON 解析失败 / 未知 type / 缺参的标准错误帧，单消息错误不断连接；下行安全序列化（不可序列化值降级字符串，发送失败标记连接关闭）。
- 新增订阅任务管理：同 turn 同连接至多一个活跃订阅（重复订阅先停旧）、session 订阅以 `session:<id>` 为键独立管理、连接断开时全量清理订阅且不影响 turn 执行。
- 事件帧结构原样透传基线 `StreamEvent` 序列化形状（`{type, source, stage, content, metadata, session_id, turn_id, seq, timestamp}`），seq 语义不改写。

## Capabilities

### New Capabilities

- `api-unified-ws`: 统一 WebSocket 端点 `/api/v1/ws` 的全部协议契约——11 种消息类型、心跳、错误语义、连接/订阅生命周期（以基线 v1.5.2 为字节级对等目标）。

### Modified Capabilities

（无——本 change 不修改既有 spec 的需求。）

## Impact

- **依赖的其他 change（按 ROADMAP 依赖图）**：`impl-foundation-stream`（StreamEvent 序列化形状与全局 bus 注册表）、`impl-turn-runtime`（start/subscribe/cancel/reply/regenerate 全部 runtime 调用面）。WS 鉴权门在 ROADMAP 上还关联 `impl-multi-user-auth`（M1 期以其鉴权接口为依赖注入点，可先以单用户实现桩接）。
- **下游解锁**：前端 Chat 页全功能（M1 退出条件）、后续各 capability 的事件面复用。
- **新增代码**：`internal/api/ws/unified.go`（端点 + 消息循环 + 订阅管理）、`internal/api/router.go` 挂载（gin route group `/api/v1`）。
- **新增依赖**：`github.com/gorilla/websocket`。
- **基线映射**：`deeptutor/api/routers/unified_ws.py`；前端消费方 `web/lib/unified-ws.ts`（心跳/重连预期，只读参照）。
- **协议/数据影响**：对外协议面与基线逐字段对等（以 WS golden fixtures 验收）；`check_active_turn` 的 `status` 可能出现 `waiting_input`（turn-runtime D-002，前端将其与 `running` 同等视为「有轮在进行」，无前端改动）。
