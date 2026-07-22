# Proposal: impl-foundation-stream

## Why

前端与全部 tool/capability 消费端逐字段依赖统一的 `StreamEvent` 信封与单 turn 扇出总线 `StreamBus`（基线 `deeptutor/core/stream.py`、`deeptutor/core/stream_bus.py`），Go 版必须先冻结这一事件协议，其他一切流式面（WS、CLI、SDK）才能对等。本模块是 ROADMAP 中 Wave 0（基础契约）/ M0 里程碑的成员，行为规格见 `docs/golang-req/openspec/specs/foundation-stream/spec.md`。

## What Changes

- 定义 `StreamEvent` 信封：JSON 序列化恰好 9 个固定键（`type`/`source`/`stage`/`content`/`metadata`/`session_id`/`turn_id`/`seq`/`timestamp`）及默认值语义。
- 定义 15 个事件类型枚举（wire 值小写下划线），Go 版不发出枚举之外的 `type`。
- 实现便捷发射器的 metadata 形状约定（`tool_call`/`tool_result`/`progress`/`sources`/`result`，trace 元数据可合并共存）。
- 实现 `StreamBus`：emit 追加 history 并推送订阅者；subscribe 先原子快照回放 history 再消费实时事件，保证不重不漏。
- 实现关闭语义：close 投递终止信号、closed 后 emit 静默丢弃、closed bus 的新订阅回放完即结束。
- 定义 seq 语义：由 turn runtime 发布点分配（本模块提供约定与消费端按 seq 去重），单调递增从 1 开始。
- 实现 `wait_for_input`/`submit_input` 暂停等待与输入投递（timeout 超时返回空串）。
- 实现 turn_id → StreamBus 进程内注册表（`register_bus`/`unregister_bus`/`get_bus`）。
- 实现 NDJSON 序列化（UTF-8 不转义非 ASCII，单行 JSON）。

## Capabilities

### New Capabilities

- `foundation-stream`: deeptutor-go 的统一流式事件协议 `StreamEvent` 与单 turn 异步扇出总线 `StreamBus`（含 seq 语义、wait_for_input、bus 注册表、NDJSON）。

### Modified Capabilities

（无）

## Impact

- 新建代码：`deeptutor-go/internal/core/`（`StreamEvent` 信封与枚举、便捷发射器、`StreamBus`、bus 注册表、NDJSON 序列化）。
- 依赖的其他 change：无（按 ROADMAP 依赖图，Wave 0 模块相互无依赖）。
- 被依赖：`impl-turn-runtime`（seq 分配的落点）、`impl-api-unified-ws`（WS 推送与 `user_input` 路由），以及所有发事件的 tool/capability 模块；`impl-sidecar-contract` 的进度事件转发也以本信封为出口。
- 协议对等：序列化结果须与 Python 基线逐字段对等，由 `contracts/` 下 WS golden fixtures 保证（acceptance.md §3.1）。
- 不修改 Python 基线与 `web/` 前端。
