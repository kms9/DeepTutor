# cron-events Specification

## Purpose

本模块定义两套进程内基础设施在 Go 侧的目标行为：cron 服务（LLM 经 `cron` 工具创建的定时任务的持久化、调度与执行——到期时以任务 message 创建一个新的 chat turn 或 partner 轮次并把结果送回原会话/原 IM 渠道）与全局 EventBus（capability 完成类事件的进程内发布/订阅，与 per-turn 的 StreamBus 扇出机制相互独立）。单机单进程模型：一个 in-process scheduler 独占 job store，不引入外部队列。

- 参考实现（基线）：`deeptutor/services/cron/service.py`、`executor.py`、`deeptutor/tools/cron_tool.py`、`deeptutor/events/event_bus.py`
- 依赖 spec / 里程碑：依赖 foundation-config（store 路径）、turn-runtime + session-store（chat job 执行）、partners（partner job 执行，M4 前可 stub 为 skipped）；被 tools-builtin（cron 工具）依赖；里程碑：M2

## Requirements

### Requirement: cron job 模型与持久化
job SHALL 持久化于 admin workspace 下的 `cron/jobs.json`（`{"version":1,"jobs":[...]}`，原子写）。job 字段 SHALL 与基线一致：`id`（短随机 id）、`name`（缺省取 message 前 48 字符）、`message`（到期执行的完整指令，必填非空）、`schedule`、`owner`、`enabled`、`delete_after_run`、`created_at_ms`、`state`（`next_run_at_ms`/`last_run_at_ms`/`last_status`/`last_error`/`run_history` 最多保留 10 条，含时长与错误）。owner SHALL 区分 `chat`（`user_id`/`is_admin`/`session_id`/`language`）与 `partner`（`partner_id`/`channel`/`chat_id`/`session_key`/`channel_meta`），owner key 为 `chat:<user_id>` 或 `partner:<partner_id>`。store 损坏时 SHALL 将其改名为 `.corrupt-<ts>` 备份后按空 store 继续，SHALL NOT 用空视图覆盖损坏文件。

#### Scenario: 损坏 store 保留取证
- **WHEN** `jobs.json` 内容非法 JSON 时服务加载
- **THEN** 原文件改名为 `.corrupt-<timestamp>` 保留
- **AND** 服务以空任务集继续运行

### Requirement: 调度表达式与校验
schedule SHALL 支持三种：`at`（epoch ms 一次性，过去时间拒绝）、`every`（秒级间隔，最小 30 秒）、`cron`（5 段 cron 表达式 + 可选 IANA `tz`，无 tz 用服务器本地时区；非法表达式、未知时区或永不触发的表达式拒绝）。下一次运行时间计算 SHALL 与基线语义一致：`at` 未到期返回该时刻否则 None；`every` 返回 now+间隔；`cron` 按表达式在指定时区求下一触发点。`at` 类型的 `delete_after_run` 缺省 SHALL 为 true。

#### Scenario: 过去的 at 时间被拒
- **WHEN** 以 `at` 为 5 分钟前的时刻创建 job
- **THEN** 创建失败并报"时间已过去"

#### Scenario: 带时区的 cron 表达式
- **WHEN** 以 `cron_expr="0 9 * * 1-5", tz="Asia/Hong_Kong"` 创建 job
- **THEN** `next_run_at_ms` 为香港时区下一个工作日 09:00 对应的 epoch ms

### Requirement: 调度循环
scheduler SHALL 单协程运行：每轮 tick 执行全部 `enabled` 且 `next_run_at_ms <= now` 的 job；随后休眠至最近到期时刻，上限 60 秒（保证外部编辑的 store 一分钟内被拾取），任何 job 增删 SHALL 立即唤醒循环。执行后 SHALL 更新 state 与 run_history；`delete_after_run` 或 `at` 类型 job 运行一次后删除，重复型 job 重算下次时间（算不出则删除）。tick 内单 job 异常 SHALL 记为该 job 的 `error` 状态，不得中断循环。服务启动时 SHALL 清理已过期的 `at` 一次性 job；到期时间已过的重复型 job 保持"到期"状态在首个 tick 补跑一次。

#### Scenario: 宕机跨越到期时间
- **WHEN** 服务停机期间某 every 型 job 的到期时刻已过，服务重启
- **THEN** 该 job 在首个 tick 补跑一次并重排下次时间
- **AND** 停机期间过期的 at 型 job 被直接清除不补跑

### Requirement: chat job 执行 — 创建 turn 并回写会话
到期的 chat job SHALL 在 owner 的用户 scope 内执行：将 job message 包装为提醒式 prompt（要求以用户语言自然转达、不暴露内部 job id），取 owner session 的既有对话史组装 UnifiedContext（`active_capability="chat"`，metadata 含 `source="cron"`、`cron_job_id`、`turn_id` 形如 `cron-<job_id>-<rand>`），经 orchestrator 跑完整 chat 轮次；取得最终回复后把 user prompt 与 assistant 回复两条消息（metadata 带 `cron_job_id`）追加到该 session，使结果出现在用户聊天历史中。session 已删除 SHALL 记 `error`；轮次无产出时以最后一条 error 或"turn produced no answer"记 `error`。

#### Scenario: 定时提醒落入原会话
- **WHEN** 某 chat job 到期且 owner session 存在
- **THEN** 以该 session 上下文执行一轮 chat，提醒结果作为新的 user/assistant 消息对出现在会话末尾

#### Scenario: 会话已删除
- **WHEN** job 到期但 `owner.session_id` 已不存在
- **THEN** 本次运行记 `status="error", error="session no longer exists"`，job 按其类型正常清理/重排

### Requirement: partner job 执行 — 注入消息总线
到期的 partner job SHALL 定位 owner partner 实例：未运行时记 `skipped`；运行中时把提醒 prompt 作为 InboundMessage（sender `cron`，携带原 channel/chat_id/session_key 与 `_cron_job_id`）注入 partner 消息总线执行一轮，最终回复若未经流式送出则显式 publish OutboundMessage 使其沿原 IM 渠道送达。

#### Scenario: partner 未运行
- **WHEN** partner job 到期而该 partner 已停止
- **THEN** 本次运行记 `status="skipped", error="partner not running"`

### Requirement: cron 工具的服务侧约束
`cron` 工具动作 SHALL 满足：`schedule` 经 schedule 校验入库并返回 job id 与首次运行时间；`list` 与 `cancel` SHALL 以 owner key 过滤——一个会话/partner 只能看见并取消自己的 job；cron 触发的轮次内（context 标记 `_cron_in_context`）SHALL 拒绝新的 `schedule`（防自增殖任务链）。工具参数面契约见 tools-builtin spec。

#### Scenario: 定时任务内禁止再调度
- **WHEN** cron 触发的 chat 轮次中 LLM 又调用 `cron(action="schedule", ...)`
- **THEN** 返回失败结果"不能在定时任务运行中调度新任务"，store 无变化

### Requirement: 全局 EventBus
系统 SHALL 提供进程内单例 EventBus：事件类型为 `SOLVE_COMPLETE`、`QUESTION_COMPLETE`、`CAPABILITY_COMPLETE`；事件负载含 `type`、`task_id`、`user_input`、`agent_output`、`tools_used`、`success`、`metadata`、`event_id`、`timestamp`。发布 SHALL 非阻塞（入队后台队列，处理协程按订阅顺序逐个调用 handler）；单个 handler 异常 SHALL 记日志且不影响其他 handler 与队列继续处理；无订阅者的事件静默丢弃。系统 SHALL 支持 subscribe/unsubscribe（幂等）、`flush`（限时等待队列排空）与优雅停止（限时排空后取消处理协程）。EventBus 与 StreamBus 的职责边界：StreamBus 面向单轮 StreamEvent 对多消费者扇出（协议见 foundation-stream spec），EventBus 面向跨模块的"能力完成"粗粒度通知（如自动保存 notebook 记录的钩子）。

#### Scenario: handler 异常不阻塞总线
- **WHEN** 两个 handler 订阅 `CAPABILITY_COMPLETE` 且第一个抛出异常
- **THEN** 异常被记录，第二个 handler 仍收到事件
- **AND** 后续事件继续正常派发

### Requirement: 桌面通知（可选增强）
交互式提醒结果 MAY 触发本地桌面通知（基线为 macOS `osascript`，且仅限 chat job 与 web channel 的 partner job），为 best-effort：失败静默、绝不影响正常投递。（有意差异：Go 版允许省略该增强或以跨平台通知库实现；不作为验收必需项。）

#### Scenario: 通知失败不影响投递
- **WHEN** 桌面通知子进程失败或超时
- **THEN** job 运行状态仍按轮次结果记录，会话内投递不受影响
