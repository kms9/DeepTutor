# api-unified-ws Delta Specification

> 事实源：`docs/golang-req/openspec/specs/api-unified-ws/spec.md`（基线 v1.5.2 字节级对等）。

## ADDED Requirements

### Requirement: 连接建立与鉴权
`/api/v1/ws` SHALL 在 accept 前完成鉴权（等价基线 `ws_require_auth`）：鉴权失败 SHALL 拒绝连接而不进入消息循环；成功后当前用户身份 SHALL 绑定到本连接的整个消息处理生命周期，连接关闭时释放。单连接 SHALL 支持顺序处理任意多条客户端消息（长连接多路复用）。

#### Scenario: 未鉴权连接
- **WHEN** 客户端未携带有效凭证连接 `/api/v1/ws`
- **THEN** 连接被拒绝，不产生任何业务事件

### Requirement: 消息帧格式与未知类型
客户端消息 SHALL 为单帧 JSON 文本，按顶层 `type` 字段路由。JSON 解析失败 SHALL 回 `{type: "error", content: "Invalid JSON."}` 并继续处理后续消息；`type` 不在支持列表内 SHALL 回 `{type: "error", content: "Unknown type: <type>"}`。任一消息的处理错误 SHALL NOT 终止连接（除底层 socket 异常外）。

#### Scenario: 未知消息类型
- **WHEN** 客户端发送 `{"type": "foo"}`
- **THEN** 收到 `{type: "error", content: "Unknown type: foo"}`，连接保持打开

### Requirement: 服务端推送的容错序列化
服务端所有下行帧 SHALL 经安全发送路径：JSON 序列化 SHALL 对不可序列化值降级为字符串（等价 `default=str`），保证单个异常事件值不会毒化推送通道；发送失败 SHALL 标记连接关闭并停止后续推送，SHALL NOT 抛给消息循环。

#### Scenario: 事件含不可序列化值
- **WHEN** 某事件 metadata 携带非常规对象
- **THEN** 该帧仍以字符串化形式送达客户端，流不中断

### Requirement: message / start_turn 启动新轮
`type ∈ {message, start_turn}` SHALL 把整条消息作为 start_turn 载荷交给 turn runtime：成功后 SHALL 自动为新 turn 建立事件订阅（`after_seq=0`）并开始推送事件流（首帧为 SESSION 事件，含 `session_id`/`turn_id`）；runtime 拒绝（config 校验失败、活跃轮冲突、模型授权失败等）SHALL 回一条完整事件形状的错误帧：`{type: "error", source: "unified_ws", content: <原因>, metadata: {turn_terminal: true, status: "rejected"}, session_id: <回显>, turn_id: "", seq: 0}`。

#### Scenario: 正常起轮
- **WHEN** 客户端发送 `{"type": "start_turn", "content": "hi", "session_id": null, ...}`
- **THEN** 后续帧为该新 turn 的事件流：SESSION → 各类过程事件 → 终态 DONE

#### Scenario: 起轮被拒
- **WHEN** runtime 因活跃轮冲突拒绝 start_turn
- **THEN** 客户端收到 `status: rejected` 的 turn_terminal 错误帧，且不建立订阅

### Requirement: 事件帧结构
推送给客户端的 turn 事件帧 SHALL 保持基线 StreamEvent 序列化形状：`{type, source, stage, content, metadata, session_id, turn_id, seq, timestamp}`；`type` 取值为 foundation-stream 定义的事件枚举（`stage_start`/`stage_end`/`thinking`/`observation`/`content`/`tool_call`/`tool_result`/`progress`/`sources`/`result`/`error`/`session`/`session_meta`/`done`/`wait_for_input`）。seq 语义（单调、可回放）由 turn-runtime 保证，WS 层 SHALL 原样透传不改写。

#### Scenario: 前端按 seq 续传
- **WHEN** 客户端记录了最后收到的 `seq`
- **THEN** 断线后可用该值发起 `resume_from` 精确续传，无缺帧

### Requirement: ping 心跳
`type: "ping"` SHALL 立即回 `{type: "pong"}`。心跳约定（前端行为，服务端 SHALL 兼容）：客户端每 30s 发一次 ping，45s 未收到任何帧则判定连接死亡并重连；`ping`/`pong` 帧 SHALL NOT 被当作业务事件消费，但 SHALL 刷新客户端的活跃时间戳。服务端 SHALL NOT 主动要求客户端心跳，任意业务帧同样刷新活跃判定。

#### Scenario: 空闲连接保活
- **WHEN** 无活跃 turn 时客户端周期性发送 ping
- **THEN** 每次都收到 pong，连接不被判死

### Requirement: subscribe_turn 与 resume_from
`type: "subscribe_turn"`（参数 `turn_id`、可选 `after_seq` 默认 0）与 `type: "resume_from"`（参数 `turn_id`、可选 `seq` 默认 0）SHALL 语义一致：以给定水位调用 runtime 的 `subscribe_turn` 回放 + 直播（先持久化事件后接直播队列，见 turn-runtime spec）。`turn_id` 缺失 SHALL 回 `{type: "error", content: "Missing turn_id."}`。对同一 `turn_id` 重复订阅 SHALL 先取消旧订阅任务再建新订阅（同 turn 同连接至多一个活跃订阅）。

#### Scenario: 刷新页面后恢复直播
- **WHEN** 客户端重连后发送 `{"type": "resume_from", "turn_id": "t1", "seq": 57}`
- **THEN** 收到 seq>57 的回放事件并续接直播，直至该 turn 的 DONE

#### Scenario: 重复订阅同一 turn
- **WHEN** 同连接对 `t1` 连续发送两次 subscribe_turn
- **THEN** 旧订阅停止，事件只按新订阅推送一份（无双份帧）

### Requirement: subscribe_session
`type: "subscribe_session"`（参数 `session_id`、可选 `after_seq`）SHALL 解析该 session 的当前活跃 turn 并等价于对其 subscribe_turn；无活跃 turn 时订阅 SHALL 静默结束（不报错、无帧）。`session_id` 缺失 SHALL 回 `Missing session_id.` 错误。session 订阅 SHALL 以 `session:<session_id>` 为键管理，与 turn 订阅互不挤占。

#### Scenario: 订阅无活跃轮的会话
- **WHEN** 对一个没有活跃 turn 的 session 发送 subscribe_session
- **THEN** 不收到任何事件帧，也不收到错误

### Requirement: check_active_turn
`type: "check_active_turn"`（参数 `session_id`）SHALL 回 `{type: "active_turn_info", turn_id, status}`：store 有活跃 turn 且本进程存在存活执行体时返回其 `turn_id` 与状态；活跃行存在但执行体已死（重启遗留）时 SHALL 把该 turn 置为 `cancelled`（原因 `Stale turn after restart`）并返回 `{turn_id: "", status: "none"}`；无活跃行同样返回 `{turn_id: "", status: "none"}`。`session_id` 缺失回错误。（配合 turn-runtime 的 waiting_input 有意差异：`status` 可能为 `running` 或 `waiting_input`，前端把两者都视为「有轮在进行」。）

#### Scenario: 重启后的陈旧 running 行
- **WHEN** 服务器重启后客户端 check_active_turn
- **THEN** 陈旧 turn 被置 cancelled，响应为 `status: "none"`，随后的 start_turn 不再被活跃轮冲突拒绝

### Requirement: unsubscribe
`type: "unsubscribe"` SHALL 停止本连接上匹配的订阅任务：携带 `turn_id` 时停对应 turn 订阅，携带 `session_id` 时停对应 session 订阅，两者可同时携带；匹配不到订阅 SHALL 为 no-op，不回错误。取消订阅 SHALL NOT 影响 turn 本身的执行。

#### Scenario: 退订后轮继续运行
- **WHEN** 客户端 unsubscribe 一个仍在运行的 turn
- **THEN** 事件推送停止，但 turn 照常完成并持久化，稍后可重新 subscribe_turn 补看

### Requirement: cancel_turn
`type: "cancel_turn"`（参数 `turn_id`，缺失回错误）SHALL 调用 runtime 的 cancel：成功无响应帧（终态经订阅流的 ERROR/DONE 事件到达）；turn 不存在或已终态 SHALL 回 `{type: "error", content: "Turn not found: <turn_id>"}`。

#### Scenario: 取消运行中的轮
- **WHEN** 客户端对运行中的 turn 发 cancel_turn
- **THEN** 订阅流上收到 `Turn cancelled` 的 ERROR 与 `status: cancelled` 的 DONE

### Requirement: submit_user_reply
`type: "submit_user_reply"`（参数 `turn_id` 必填；`text` 可选，允许空串；`answers` 可选，`[{questionId|id, text}]` 列表，无有效 questionId 的条目剔除，清洗后为空视为无 answers）SHALL 把回复投递给暂停在 `ask_user` 的 turn：接受后无响应帧（后续进展经订阅流可见）；turn 未在等待（已结束、被取消、或从未提问）SHALL 回 `{type: "error", content: "Turn <turn_id> is not awaiting a user reply."}`。

#### Scenario: v2 多问回复
- **WHEN** 客户端提交 `answers: [{questionId: "q1", text: "A"}, {questionId: "q2", text: ""}]`
- **THEN** 回复被接受，turn 从 `waiting_input` 回到 `running`，订阅流出现 `ask_user_resolved` progress 后循环续跑

### Requirement: regenerate
`type: "regenerate"`（参数 `session_id` 必填；`overrides` 可选 dict，支持 `capability`/`tools`/`knowledge_bases`/`language`/`config`/`notebook_references`/`history_references`/`llm_selection`/`book_references`）SHALL 调用 runtime 的 regenerate：成功后 SHALL 对新 turn 自动订阅（`after_seq=0`）；失败 SHALL 回完整事件形状错误帧，`metadata` 含 `turn_terminal: true`、`status: "rejected"`、`reason` 为错误码——`regenerate_busy`（有轮在跑）或 `nothing_to_regenerate`（无历史 user 消息 / session 不存在）。

#### Scenario: 重新生成上一答案
- **WHEN** 客户端对已完成会话发送 regenerate
- **THEN** 末尾 assistant 消息被替换为新 turn 的产出，客户端自动收到新 turn 事件流

#### Scenario: 无可重跑内容
- **WHEN** 对空会话发送 regenerate
- **THEN** 收到 `reason: "nothing_to_regenerate"` 的 rejected 错误帧

### Requirement: user_input（StreamBus 直通输入）
`type: "user_input"`（参数 `turn_id` 必填、`content` 字符串）SHALL 经全局 bus 注册表定位该 turn 的活跃 StreamBus 并投递输入，解除 capability 侧 `wait_for_input` 等待（与 `submit_user_reply` 的 runtime 回复队列是两条独立通道：`user_input` 服务于 bus 级 `WAIT_FOR_INPUT` 事件的等待方）。`turn_id` 缺失回 `Missing turn_id for user_input.`；无活跃 bus SHALL 回 `{type: "error", content: "No active bus for turn: <turn_id>"}`。输入投递 SHALL 唤醒当前全部等待者并清空等待队列。

#### Scenario: 无活跃 bus
- **WHEN** 对已结束的 turn 发送 user_input
- **THEN** 收到 `No active bus for turn` 错误帧

### Requirement: 连接关闭与订阅清理
连接断开（客户端主动或异常）时服务端 SHALL：取消本连接的全部订阅任务并等待其退出、释放用户身份绑定。连接级未捕获异常 SHALL 尽力回一条 `{type: "error", content: <异常>}` 后走同样清理。订阅任务被取消 SHALL NOT 影响 turn 执行与事件持久化（重连后可回放）。

#### Scenario: 直播中断线
- **WHEN** turn 直播推送期间客户端断开连接
- **THEN** 服务端清理该连接的订阅任务，turn 继续在后台完成
- **AND** 客户端重连后经 `resume_from` 可完整补齐缺失事件
