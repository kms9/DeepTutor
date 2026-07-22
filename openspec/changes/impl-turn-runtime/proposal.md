## Why

Turn 级运行时是 deeptutor-go 连接协议面与执行引擎的中枢：接收 start_turn 载荷后在后台运行完整用户轮、组装 `UnifiedContext`、统一分配事件 seq、持久化事件与 assistant 消息，并对外提供 `subscribe_turn` 回放、cancel、regenerate/消息分支、`ask_user` 暂停/恢复。行为规格见 `docs/golang-req/openspec/specs/turn-runtime/spec.md`（含两处已登记有意差异：D-002 `waiting_input` 状态、D-003 append-before-publish）；按 `docs/golang-req/openspec/ROADMAP.md`，本模块位于 **Wave 1（运行时内核）/ 里程碑 M1**，是 api-unified-ws 的直接上游，M1「resume/cancel/regenerate/ask_user 场景全过」的承载者。

## What Changes

- 新增 `TurnRuntimeManager`：start_turn 载荷处理与准入（config 校验、persona/llm_selection 解析、多用户模型授权门、tools 回填与 per-user 白名单、session preferences 更新）。
- 新增 Turn 状态机：`running`/`waiting_input`/`completed`/`failed`/`cancelled`（**D-002**：`ask_user` 暂停显式置 `waiting_input`），同 session 至多一个非终态 turn。
- 新增 seq 唯一分配权威（事件发布函数统一取号）与事件持久化（**D-003**：append-before-publish，崩溃后已 publish 事件可回放；镜像 `events.jsonl` 静默容错）。
- 新增后台执行与直播扇出（订阅者队列非阻塞投递、SESSION 首帧先行、orchestrator SESSION 吞并、DONE 暂扣后置发布）。
- 新增 `subscribe_turn(after_seq)` 回放 + 直播衔接（原子补发、seq 去重、终态合成 ERROR/DONE）、孤儿 running turn 恢复。
- 新增 `cancel_turn`（部分成果持久化、解除 ask_user 等待）、`submit_user_reply` 回复队列、`regenerate_last_turn` 与 `parent_message_id` 消息分支。
- 新增附件接收/存储/Sidecar 解析链路、`ContextBuilder` 历史压缩（35% 预算、摘要水位、分支失效重建）、`UnifiedContext` 全量组装（memory/persona/skills/source inventory 或 legacy 拼接）。
- 新增 assistant 消息与 trace 持久化（narration 剔除、events trace、`generated: true` 工件附件、request snapshot）、失败路径终态事件、会话标题生成（20s 限时 + `session_meta` 事件）。

## Capabilities

### New Capabilities

- `turn-runtime`: Turn 生命周期/状态机、seq 分配、事件持久化、resume/cancel/regenerate、ask_user 队列、UnifiedContext 组装的全部行为契约（以基线 v1.5.2 为对等目标，含 D-002/D-003 有意差异）。

### Modified Capabilities

（无——本 change 不修改既有 spec 的需求。）

## Impact

- **依赖的其他 change（按 ROADMAP 依赖图）**：`impl-foundation-stream`（StreamEvent/seq 语义）、`impl-session-store`（turns/turn_events/messages 表与 store API，含 D-001 seq 冲突报错语义）、`impl-runtime-registry`（ChatOrchestrator 驱动）、`impl-agent-loop`（waiter 句柄消费方）、`impl-sidecar-contract`（附件文档解析）。
- **下游解锁**：`impl-api-unified-ws`（全部 11 种消息类型的 runtime 调用面）、memory/knowledge 及各 capability change。
- **新增代码**：`internal/runtime/turn/`（manager、状态机、扇出、收尾）、`internal/session/context_builder/`（历史压缩、source inventory）、`internal/core/context.go` 扩展（UnifiedContext 全量字段）。
- **基线映射**：`deeptutor/services/session/turn_runtime.py`、`context_builder.py`、`source_inventory.py`、`deeptutor/core/context.py`、`sqlite_store.py`（turns/turn_events 语义）。
- **协议/数据影响**：沿用基线 SQLite schema（不改表结构）；`data/user/workspace` 下新增（与基线同布局的）`events.jsonl` 镜像；WS 事件面经 api-unified-ws 暴露，D-002/D-003 均无前端可见影响（见 acceptance.md §6）。
