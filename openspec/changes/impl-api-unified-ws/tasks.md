## 1. 端点骨架与连接生命周期

- [ ] 1.1 引入 `gorilla/websocket`；在 gin route group `/api/v1` 挂载 `GET /ws`，实现 upgrade 前 `WSAuthenticator` 鉴权（失败 HTTP 拒绝不进消息循环）+ 单用户默认实现
- [ ] 1.2 实现 `wsConn`：读循环（顺序 JSON 帧路由、`Invalid JSON.`/`Unknown type: <type>` 错误帧、单消息错误不断连）+ 单写者 goroutine + `sendCh`
- [ ] 1.3 实现 `safeSend` 容错序列化（不可序列化值降级字符串、`WriteMessage` 失败置 closed 停止推送）与事件 `ToWire()`（固定字段序 `{type, source, stage, content, metadata, session_id, turn_id, seq, timestamp}`，透传不改写）
- [ ] 1.4 实现连接关闭清理：cancel 全部订阅并等退出、close sendCh、释放身份绑定；连接级 panic 尽力回错误帧后同样清理

## 2. 订阅任务管理

- [ ] 2.1 实现 `subscription` 表（turn 键 = `turn_id`、session 键 = `session:<session_id>` 互不挤占）与 `stopSubscription`（cancel + 等 done）
- [ ] 2.2 实现订阅转发 goroutine：消费 runtime 事件通道 → ToWire → sendCh，ctx select 防泄漏（goleak 验证）
- [ ] 2.3 实现重复订阅先停旧（同 turn 同连接单活跃订阅）与 `unsubscribe`（turn_id/session_id 可同携、no-op 容忍、不影响 turn 执行）

## 3. 消息类型处理

- [ ] 3.1 实现 `message`/`start_turn`：整条消息作 StartTurn 载荷、成功自动订阅（after_seq=0）、RejectedError 转完整事件形状错误帧（`source: "unified_ws"`、`turn_terminal`/`status: "rejected"`、session_id 回显、seq 0）
- [ ] 3.2 实现 `ping` → `{type: "pong"}` 立即应答
- [ ] 3.3 实现 `subscribe_turn`（`after_seq` 默认 0）与 `resume_from`（`seq` 默认 0）同语义路由；缺 `turn_id` 回 `Missing turn_id.`
- [ ] 3.4 实现 `subscribe_session`：解析活跃 turn 等价订阅、无活跃轮静默结束、缺参回 `Missing session_id.`
- [ ] 3.5 实现 `check_active_turn`：`active_turn_info` 响应、执行体已死置 `cancelled`（`Stale turn after restart`）回 `status: "none"`、`status` 透出 `running`/`waiting_input`（D-002）
- [ ] 3.6 实现 `cancel_turn`（成功无响应帧、失败 `Turn not found: <turn_id>`）与 `submit_user_reply`（answers 清洗归一、未在等待回 `Turn <turn_id> is not awaiting a user reply.`）
- [ ] 3.7 实现 `regenerate`：overrides 透传、成功自动订阅、失败 rejected 帧带 `reason: regenerate_busy|nothing_to_regenerate`
- [ ] 3.8 实现 `user_input`：全局 bus 注册表定位 → 投递并唤醒全部等待者清空队列；缺参 `Missing turn_id for user_input.`、无 bus `No active bus for turn: <turn_id>`

## 4. Scenario 测试与协议对等验收

- [ ] 4.1 将 spec 全部 Scenario 落为 Go 测试（httptest + gorilla client + mock/真 runtime）：未鉴权拒绝、未知类型不断连、不可序列化值降级、正常起轮/起轮被拒、seq 续传、ping 保活、resume_from 恢复直播、重复订阅无双帧、无活跃轮 session 订阅静默、重启陈旧行置 cancelled、退订后轮继续、取消运行轮、v2 多问回复、regenerate 成功/nothing_to_regenerate、无活跃 bus、直播中断线后 resume_from 补齐
- [ ] 4.2 WS golden fixtures 回放对等验收（acceptance.md §3.1 全部 7 类场景 + M1 矩阵「统一 WS 全消息类型」）：普通 chat turn、subscribe_turn(after_seq) 断线重连、cancel_turn、regenerate（parent_message_id 分支）、ask_user 暂停→submit_user_reply 恢复、check_active_turn/subscribe_session/ping-pong——mock LLM 回放逐事件 diff（忽略 timestamp；seq 连续单调、事件顺序一致；11 种消息类型全覆盖）
