# Delta Spec: foundation-stream

> 事实源：`docs/golang-req/openspec/specs/foundation-stream/spec.md`（Requirement/Scenario 原样搬运，未增删语义）。

## ADDED Requirements

### Requirement: StreamEvent 信封字段
系统 SHALL 定义 `StreamEvent`，其 JSON 序列化（`to_dict`）恰好包含 9 个字段且键名固定：`type`（字符串枚举）、`source`（产生方，如 `"deep_solve"`）、`stage`（产生方内阶段，如 `"planning"`）、`content`（人类可读文本）、`metadata`（任意结构化 JSON object）、`session_id`、`turn_id`、`seq`（整数）、`timestamp`（Unix epoch 秒，浮点）。未显式赋值时默认值 SHALL 为：字符串字段空串、`metadata` 空 object、`seq=0`、`timestamp` 为创建时刻。序列化 SHALL NOT 增删字段或改名。

#### Scenario: 默认事件序列化
- **WHEN** 以 `type=content, content="Hello"` 构造事件并序列化
- **THEN** 输出 JSON 恰好含上述 9 个键，`source`/`stage`/`session_id`/`turn_id` 为 `""`、`seq` 为 `0`、`metadata` 为 `{}`

### Requirement: 事件类型枚举
系统 SHALL 支持且仅支持以下 `type` 枚举值（wire 值为小写下划线字符串）：`stage_start`、`stage_end`、`thinking`、`observation`、`content`、`tool_call`、`tool_result`、`progress`、`sources`、`result`、`error`、`session`、`session_meta`、`done`、`wait_for_input`。Go 版 SHALL NOT 发出枚举之外的 `type`。

#### Scenario: 枚举完整性
- **WHEN** 对比 Go 版与 Python 基线的事件类型集合
- **THEN** 两者的 15 个 wire 值一一对应

### Requirement: 便捷发射器的 metadata 约定
系统 SHALL 保持基线便捷发射器的 metadata 形状约定：`tool_call` 事件 `content` 为工具名、`metadata.args` 为参数 object；`tool_result` 事件 `content` 为结果文本、`metadata.tool` 为工具名；`progress` 事件 `metadata` 含 `current`/`total` 整数；`sources` 事件 `metadata.sources` 为来源 object 数组；`result` 事件把结果 payload 直接并入 `metadata`。trace 元数据（如 `trace_layer`）SHALL 可与上述键合并共存。

#### Scenario: tool_call 事件形状
- **WHEN** 发射工具调用 `rag`、参数 `{"query":"x"}`
- **THEN** 事件为 `type=tool_call, content="rag", metadata.args={"query":"x"}`

### Requirement: StreamBus 扇出与历史回放
`StreamBus` SHALL 为单个 chat turn 的事件通道：`emit` 把事件追加进内部 history 并推送给所有活跃订阅者；`subscribe` 返回异步事件流，订阅时 SHALL 先回放订阅时刻之前的全部 history，再持续消费实时事件。回放范围快照 SHALL 与订阅者队列注册在同一原子步骤内确定，以保证回放期间新 emit 的事件只经队列投递一次，任何事件 SHALL NOT 被单个订阅者收到两次或漏收。

#### Scenario: 迟到订阅者补齐历史
- **WHEN** bus 已 emit 3 个事件后新订阅者加入，随后又 emit 2 个事件
- **THEN** 该订阅者按序收到全部 5 个事件，无重复

#### Scenario: 回放期间的并发 emit 不重复
- **WHEN** 订阅者正在回放 history 时生产者并发 emit 新事件
- **THEN** 新事件仅通过实时队列投递一次

### Requirement: 关闭语义
`close` SHALL 将 bus 置为 closed 并向所有订阅者投递终止信号，订阅流随之正常结束；closed 之后的 `emit` SHALL 被静默丢弃。已 closed 的 bus 被新订阅者订阅时，SHALL 回放完 history 后立即结束流。

#### Scenario: 关闭后订阅
- **WHEN** bus close 后新订阅者订阅
- **THEN** 收到全部历史事件后流立即结束，不阻塞

### Requirement: seq 单调递增且由 turn runtime 分配
`seq` SHALL 是 turn 内事件的单调递增序号，从 1 开始。分配职责 SHALL 在 turn runtime 的发布点（对应基线 `_publish_live_event`），而非 `StreamBus` 或事件产生方：capability/tool 发出的事件默认 `seq=0`，发布时若 `seq<=0` 则分配 `next_seq` 并自增；若事件已带 `seq>0` 则保留该值并把 `next_seq` 推进到 `max(next_seq, seq+1)`。同一 turn 内已分配的 seq SHALL 严格单调递增、SHALL NOT 重复。消费端（订阅回放）SHALL 以 seq 去重：`seq <= last_seq` 的事件跳过。

#### Scenario: 默认事件获得递增 seq
- **WHEN** 同一 turn 依次发布 3 个 `seq=0` 的事件
- **THEN** 发布后事件 seq 依次为 1、2、3

#### Scenario: 显式 seq 推进计数器
- **WHEN** 发布一个 `seq=10` 的事件后再发布一个 `seq=0` 的事件
- **THEN** 后者获得 `seq=11`

### Requirement: wait_for_input 暂停与输入投递
`StreamBus` SHALL 支持暂停等待用户输入：`wait_for_input(prompt, timeout)` 先 emit 一个 `type=wait_for_input`、`content=prompt` 的事件，然后阻塞等待；`submit_input(content)` SHALL 把输入投递给全部等待者并清空等待者列表。`timeout` 为空时无限等待（交互式客户端）；超时后 SHALL 返回空字符串而非报错（headless/CLI 场景）。

#### Scenario: 前端回复解除等待
- **WHEN** capability 调用 `wait_for_input("请确认")`、随后 WS 层调用 `submit_input("好的")`
- **THEN** 等待方返回 `"好的"`，且总线上出现过一条 `wait_for_input` 事件

#### Scenario: 超时返回空串
- **WHEN** `wait_for_input` 设定有限 timeout 且无人提交输入
- **THEN** 超时后返回 `""`，不抛异常

### Requirement: 按 turn_id 的 bus 注册表
系统 SHALL 维护 turn_id → StreamBus 的进程内注册表（`register_bus` / `unregister_bus` / `get_bus`），使 WebSocket 层收到 `user_input` 类消息时能按 turn_id 定位到活跃 bus 并调用 `submit_input`。查询不存在的 turn_id SHALL 返回空而非报错。

#### Scenario: 按 turn 路由用户输入
- **WHEN** turn `t1` 的 bus 已注册且 WS 收到面向 `t1` 的用户输入
- **THEN** 输入被投递到 `t1` 的 bus；面向未注册 turn 的输入被安全忽略

### Requirement: NDJSON 序列化
系统 SHALL 提供事件到单行 JSON 字符串的序列化（NDJSON，UTF-8 不转义非 ASCII），供 CLI JSON 输出与日志消费者使用，字段与 `to_dict` 一致。

#### Scenario: 中文内容不转义
- **WHEN** 序列化 `content="你好"` 的事件
- **THEN** 输出单行 JSON 中包含字面 `你好` 而非 `\uXXXX` 转义
