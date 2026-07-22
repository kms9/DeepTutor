## 1. 类型与执行体骨架

- [ ] 1.1 扩展 `UnifiedContext` 全量字段（conversation_history/attachments/memory_context/persona_context/skills_manifest/source_index/metadata 等）与 `TurnInfo`/`SessionInfo`/`RejectedError`/`UserReply` 类型
- [ ] 1.2 实现 `turnExecution`：seq `atomic.Int64` 计数器（自带 seq>0 推进语义）、订阅者集合、内存 buffer、`replyQ` 回复队列、turn ctx/cancel
- [ ] 1.3 实现 Turn 状态机迁移函数（`running ↔ waiting_input`、非终态 → 终态单向）与「活跃 turn」查询（覆盖两个非终态，D-002）

## 2. 事件发布与持久化

- [ ] 2.1 实现 `publishEvent` 唯一发布路径：补齐 session_id/turn_id、统一取号、append-before-publish（store 同步落 `turn_events`，D-003；依赖 store 的 D-001 冲突报错语义）、turn 行已删时跳过不级联
- [ ] 2.2 实现订阅者非阻塞扇出（队列满丢弃本条）与 `events.jsonl` 工作区镜像（失败静默容忍）
- [ ] 2.3 实现 WAL + 单 writer 批量 group-commit 优化，并用 1 万事件回放基准验证 acceptance.md §3.4 指标

## 3. start_turn 准入与后台执行

- [ ] 3.1 实现 `StartTurn` 准入链：capability 默认、runtime-only config 键剥离、request schema 校验拒绝、`ensure_session`、persona 解析持久化、llm_selection 授权门（非 admin 无 grant 拒绝 / 固定首个授权模型）、tools None 回填 + per-user 白名单过滤、session preferences 更新、turn 行创建
- [ ] 3.2 实现后台执行 goroutine：返回前发布首帧 SESSION（regenerate 轮附加标记）、orchestrator SESSION 吞并、DONE 暂扣后置发布（未产出则合成 `status: completed`）
- [ ] 3.3 实现附件链路：规范化（缺 id 生成）、base64 入附件存储回填 url（失败非致命）、Sidecar 文档解析写 `extracted_text`、落库副本清空 base64

## 4. ContextBuilder 与 UnifiedContext 组装

- [ ] 4.1 实现历史压缩：35% 窗口预算（下限 256）、摘要 40%、水位 `summary_up_to_msg_id`、分支祖先链失效重建、超预算 LLM 摘要（原文重建 vs 折叠、成功才推水位、失败降级）、最终逐条丢弃兜底
- [ ] 4.2 摘要 trace 事件（stage `summarize_context`）经注入回调走 `publishEvent` 统一取号
- [ ] 4.3 实现 UnifiedContext 组装：memory_context（memory_references 非空才读 L3）、persona 回退链、skills_manifest（单行清单 + always 全文 + admin 指派）、chat 轮 source inventory（manifest + `source_index` + `first_seen_turn`）、非 chat 轮 legacy 拼接（引用内容经分析压缩）、metadata 全字段、quiz 追问轮 system 消息前置

## 5. subscribe / cancel / reply / regenerate

- [ ] 5.1 实现 `SubscribeTurn`：store 回放（seq > after_seq）→ 持锁原子挂接 + buffer 补发 → 直播转发，全程 seq 去重；无本地执行时二次 store catch-up；终态未见 DONE 合成 ERROR（failed）/DONE（`synthesized: true`），非终态不合成
- [ ] 5.2 实现孤儿轮恢复：StartTurn 前置 failed、SubscribeTurn 先终态化再合成收尾；存活性只看内存 execs
- [ ] 5.3 实现 `CancelTurn`：存活执行体取消并等收尾 / store 非终态直接置 cancelled / 否则 false；收尾发 ERROR(`Turn cancelled`)+DONE、部分成果 assistant 消息（narration 剔除 + `generated: true` 工件）、各步骤独立容错、解除 ask_user 等待
- [ ] 5.4 实现 `SubmitUserReply` 回复队列 + waiter 闭包注入（等待置 `waiting_input`、恢复置 `running`，D-002）、收尾先摘队列保证迟到回复 false
- [ ] 5.5 实现 `RegenerateLastTurn`：活跃轮 `regenerate_busy` / 无 user 消息 `nothing_to_regenerate`、删除末尾 assistant、overrides → snapshot → preferences 三级载荷重建、runtime 键注入、不触发标题生成；`parent_message_id` 消息分支（兄弟分支挂接、上下文只取祖先链、缺席 legacy 追加）

## 6. 收尾持久化与标题

- [ ] 6.1 实现正常收尾：assistant 消息（content 段筛选规则 + narration 轮剔除 + thinking 清洗兜底）、`events` trace 字段（done/session 除外）、工件附件按 url 去重、user 消息 request snapshot metadata
- [ ] 6.2 实现失败路径：DONE 前异常发 ERROR+DONE（`status: failed`）后置 failed；DONE 后持久化失败只记日志置 failed 不再发事件；全路径保证离开非终态（收尾 `sync.Once` + 独立 recover）
- [ ] 6.3 实现会话标题生成：哨兵值判定、LLM 生成（20s 限时、清洗、80 字符截断）、失败回退 user 消息 50 字符、`session_meta` 事件发布、已改名短路

## 7. Scenario 测试与协议对等验收

- [ ] 7.1 将 spec 全部 Scenario 落为 Go 测试（mock LLM + 临时 SQLite）：config 校验拒绝、无授权拒绝、waiting_input 迁移、并发起轮拒绝、统一取号、崩溃回放（kill 进程或模拟）、DONE 后置、断线重连去重、failed 轮合成收尾、孤儿轮、取消暂停轮/部分成果、终态后 reply false、编辑分支/regenerate_busy、PDF 附件全链路、摘要水位失效/摘要失败降级、chat 多源/legacy 拼接、narration 不入答案、capability 异常终态、标题生成/超时回退
- [ ] 7.2 WS golden fixtures 回放对等验收（acceptance.md §3.1 + M1 矩阵「统一 WS 全消息类型」）：`subscribe_turn(after_seq)` 断线重连、`cancel_turn`、`regenerate`（parent_message_id 分支）、`ask_user` 暂停→恢复四类 fixtures 经 mock provider 回放逐事件 diff（忽略 timestamp；seq 连续单调、顺序一致）
