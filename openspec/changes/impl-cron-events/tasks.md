# Tasks: impl-cron-events

## 1. job 模型与持久化

- [ ] 1.1 `internal/cron/model.go`：Job/Schedule/Owner（chat|partner + owner key）/State/RunRecord（run_history ≤10）字段与基线对齐
- [ ] 1.2 `internal/cron/store.go`：`cron/jobs.json` 加载/原子写、损坏改名 `.corrupt-<ts>` 保留取证后空 store 继续（不覆盖原文件）

## 2. 调度表达式与循环

- [ ] 2.1 `internal/cron/schedule.go`：`at`（过去拒绝、`delete_after_run` 缺省 true）、`every`（≥30s）、`cron`（robfig/cron `ParseStandard` + `time.LoadLocation` 校验 tz、永不触发拒绝）；`NextRun` 三型语义与基线一致
- [ ] 2.2 `internal/cron/scheduler.go`：单协程 tick 循环（执行到期 → 更新 state/run_history → 睡 min(最近到期, 60s)）、增删 job 经 wake channel 即时唤醒、单 job 异常 recover 记 error 不中断
- [ ] 2.3 启动语义：过期 `at` 一次性 job 清除不补跑；过期重复型 job 首个 tick 补跑一次并重排
- [ ] 2.4 运行后处理：`delete_after_run`/`at` 型运行一次删除；重复型重算下次（算不出删除）

## 3. executor 与工具服务侧

- [ ] 3.1 `internal/cron/executor.go` chat job：提醒式 prompt 包装、owner scope 显式传递、对话史组装 UnifiedContext（`source="cron"`、`cron_job_id`、`turn_id=cron-<job_id>-<rand>`）、orchestrator 跑完整轮次、user/assistant 消息对回写 session（metadata 带 `cron_job_id`）
- [ ] 3.2 chat job 错误路径：session 已删除记 `error="session no longer exists"`；轮次无产出记最后 error 或 `"turn produced no answer"`
- [ ] 3.3 partner job：M4 前 stub —— partner 未运行统一记 `status="skipped", error="partner not running"`；预留 InboundMessage 注入接口（sender `cron`、携带 channel/chat_id/session_key/`_cron_job_id`）
- [ ] 3.4 `internal/cron/service.go`（cron 工具消费）：`Schedule`（校验入库返回 job id + 首次运行时间）、`List`/`Cancel`（owner key 过滤）、`_cron_in_context` 内拒绝 schedule
- [ ] 3.5 `internal/cron/notify.go`：桌面通知可选增强（best-effort、失败静默；按 spec「桌面通知」条目为有意差异允许省略或跨平台库实现）

## 4. 全局 EventBus

- [ ] 4.1 `internal/events/bus.go`：三事件类型与负载字段、非阻塞 Publish（有界队列 + 单处理协程按订阅顺序派发）、handler recover 隔离、无订阅者静默丢弃
- [ ] 4.2 Subscribe/Unsubscribe（幂等）、`Flush`（限时排空）、`Stop`（限时排空后停协程）

## 5. 测试与验收（spec Scenario 落测试 + 协议对等）

- [ ] 5.1 将本 spec 全部 10 个 Scenario 落为 Go 测试：损坏 store 取证、过去 at 拒绝、tz cron next-run（Asia/Hong_Kong golden case）、宕机补跑/清理、提醒落入原会话（mock LLM + orchestrator 集成）、会话已删除、partner 未运行 skipped、cron 轮次内拒绝再调度、handler 异常不阻塞、通知失败不影响投递
- [ ] 5.2 robfig parser 与基线 croniter 的表达式 golden case 对照（工作日/月末/DST 切换样例）
- [ ] 5.3 M2 验收：cron 任务持久化、触发执行 turn 端到端（acceptance.md M2 矩阵「Cron + EventBus」项）；chat job 产生的会话消息经 sessions REST contract test 验证与基线结构一致（acceptance.md 3.1 A 类）
