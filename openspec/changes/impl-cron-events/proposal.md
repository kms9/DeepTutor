# Change: impl-cron-events

## Why

deeptutor-go 需要两套进程内基础设施：cron 服务（LLM 经 `cron` 工具创建的定时任务的持久化、调度与执行——到期以任务 message 创建新的 chat turn 或 partner 轮次并把结果送回原会话/原 IM 渠道）与全局 EventBus（capability 完成类事件的进程内发布/订阅，与 per-turn StreamBus 相互独立）。本 change 依据模块行为规格 `docs/golang-req/openspec/specs/cron-events/spec.md` 立项，对应 `docs/golang-req/openspec/ROADMAP.md` Wave 2「领域服务与工具」、里程碑 M2（acceptance.md M2 矩阵「Cron + EventBus」项：cron 任务持久化、触发执行 turn）。单机单进程模型：一个 in-process scheduler 独占 job store，不引入外部队列。

## What Changes

- cron job 模型与持久化：`cron/jobs.json`（admin workspace 下，`{"version":1,"jobs":[...]}` 原子写），job 字段与基线一致（id/name/message/schedule/owner/enabled/delete_after_run/created_at_ms/state 含 run_history ≤10 条）；owner 区分 `chat`/`partner` 两型；损坏 store 改名 `.corrupt-<ts>` 保留取证后按空 store 继续。
- 调度表达式与校验：`at`（过去时间拒绝，`delete_after_run` 缺省 true）/ `every`（最小 30 秒）/ `cron`（5 段表达式 + IANA `tz`），下一次运行时间计算与基线语义一致。
- 调度循环：单协程 tick、休眠至最近到期（上限 60 秒）、增删即时唤醒、单 job 异常不中断循环、启动清理过期 `at` 一次性 job、过期重复型 job 首个 tick 补跑一次。
- chat job 执行：提醒式 prompt 包装、owner session 对话史组装 UnifiedContext（`source="cron"`、`turn_id` 形如 `cron-<job_id>-<rand>`）、经 orchestrator 跑完整 chat 轮次、结果以 user/assistant 消息对回写 session；session 已删除记 `error`。
- partner job 执行：partner 未运行记 `skipped`；运行中注入 InboundMessage 到 partner 消息总线（M4 前可 stub 为 skipped）。
- cron 工具服务侧约束：owner key 过滤 list/cancel、`_cron_in_context` 内拒绝再 schedule（防自增殖任务链）。
- 全局 EventBus：`SOLVE_COMPLETE`/`QUESTION_COMPLETE`/`CAPABILITY_COMPLETE` 三事件、非阻塞发布、handler 异常隔离、subscribe/unsubscribe 幂等、flush 与优雅停止。
- 桌面通知（可选增强，best-effort，非验收必需项）。

## Capabilities

### New Capabilities

- `cron-events`: cron 定时任务子系统（模型/持久化/调度/chat 与 partner 执行/工具侧约束）与全局 EventBus 的 Go 侧行为契约。

### Modified Capabilities

（无）

## Impact

- **依赖的其他 change**：
  - `impl-turn-runtime`：chat job 到期执行需经 orchestrator/turn runtime 跑完整 chat 轮次并组装 UnifiedContext；session 消息回写经 session store（turn-runtime 已依赖 `impl-session-store`）。
  - 间接依赖：`impl-foundation-config`（store 路径）。partners 的 partner job 执行在 M4 交付，此前 stub 为 `skipped`（spec 依赖说明）。
- **被依赖**：`impl-tools-builtin` 的 `cron` 工具消费本模块 service；capability 完成钩子（如自动保存 notebook 记录）订阅 EventBus。
- **受影响代码**：新增 `internal/cron/`（store/schedule/scheduler/executor）与 `internal/events/`（EventBus）。
- **前端影响**：无直接协议面（cron 结果以普通会话消息呈现）；提醒结果出现在聊天历史属 M2 Scenario 验收。
