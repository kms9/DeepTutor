## 1. eino 组件桥接与基础设施

- [ ] 1.1 引入 `cloudwego/eino` 依赖并锁定版本；定义 `EventSink`（StreamBus 薄封装）与 trace metadata 构造（call_id / trace_kind / call_state / call_role）
- [ ] 1.2 实现 `einoToolAdapter`：`core.Tool` → `tool.InvokableTool`（Info 来自 registry schema，InvokableRun 转发 `registry.Execute`）；registry OpenAI schema ↔ `schema.ToolInfo` 互转（含 `raw_parameters`）
- [ ] 1.3 实现 `RelayModelStream`：消费 `StreamReader[*schema.Message]`，content/thinking 增量事件（`llm_chunk`）、内联 `<think>`/`<thinking>` 标签增量分流状态机（跨 chunk 暂扣 + 流末冲刷），返回完整 assistant 消息（原始文本保留标签）

## 2. 消息组装与工具挂载

- [ ] 2.1 实现会话组装 `messages.go`：system prompt 字节稳定 → 压缩摘要（`[Conversation summary]` header 的 system 条目）→ 历史 → 末尾 user 消息；KB seed / capability pre-loop seed/briefing 拼 user 消息；附件多模态准备（图片内联、文档 extracted_text）
- [ ] 2.2 实现 `ComposeEnabledTools` 四步组装（toggle 白名单 ∩ enabled_tools、条件挂载门控、capability owned、恒挂载）+ 有序去重 + `allowed_builtin_tools`/`forced`/`suppressed`/exclusive 裁剪 + `imagegen`/`videogen` 未配置剔除
- [ ] 2.3 实现 deferred 池：初始 schema 排除、system prompt 单行清单、`load_tools` 加载进会话级已加载集、`mcp_tools_filter` × 用户 grant 交集裁剪、PageIndex 隐式预加载、准备失败静默降级
- [ ] 2.4 实现 KB seed 预检索：≤3 KB 并行 hybrid 检索、每 KB 截断 4000 字符、sources 事件先行、失败/空结果跳过

## 3. 工具分发

- [ ] 3.1 实现 `DispatchToolCalls`：errgroup 并行（上限 8，超限截断 + warning progress）、JSON arguments 解析（失败回退 `{}`）、服务端 kwargs 注入器（`rag` mode、sandbox 工作目录、`cron` owner 等）、结果按 index 配对
- [ ] 3.2 实现同批判重：（工具名 + 规范化参数）判重 stub；`ask_user` 同批第二个起一律重复且抑制 tool_call/tool_result 事件；工具异常转 `success=false` 结果不中断该批
- [ ] 3.3 实现 tool_call/tool_result 事件产生（`_` 前缀私有参数剥离、`tool_metadata` 附带）与 retrieve 类子 trace（running/complete/error progress + event_sink 进度直通）

## 4. 轮循环与收敛

- [ ] 4.1 实现 `runLoop` 主循环：每轮 `WithTools` + `Stream`、call_status running/complete（`call_role: narration|finish`）、tool_call 轮追加消息继续、无 tool_call 即 finish、stage `responding` 包裹、sources 合并、capability result（`response`/`completed`/`rounds`/`tool_steps` + UsageTracker 汇总）
- [ ] 4.2 实现轮次预算（`max_rounds` 默认 8 下限 1、`_min_loop_rounds` 地板）与强制收敛（warning progress + 收敛指令 user 消息 + 关闭工具的收尾调用，`call_kind: llm_final_response`）
- [ ] 4.3 实现中途失败挽救收敛（非首轮失败走收敛、首轮失败上抛、挽救再失败发协议兜底终答）与空终答 nudge（一次为限，之后 fallback 文案）
- [ ] 4.4 实现上下文窗口守护（90% 预算、最早 tool 消息逐条 snip + warning）与 `_context_checkpoint.summary` 折叠（回到轮初边界 + `[Context checkpoint]` system 消息）
- [ ] 4.5 实现 `ask_user` 暂停/恢复：首个 `pause_for_user` 胜出、waiter 缺席时问题摘要作终答、回复格式化替换进 `pause_tool_call_id` 对应 tool 消息 + `ask_user_resolved` progress 续跑、waiter 空值结束循环（`completed=false`）、pause 与 terminate 互斥
- [ ] 4.6 实现 provider 兼容性回退：`stream_options` 剥除重试、工具 schema 不兼容降级（warning + 本轮后续不带 tools）、图片剥离重试、无原生 tool calling 能力时整轮不带 schema 且不注入工具清单

## 5. Scenario 测试与协议对等验收

- [ ] 5.1 用 mock/replay ChatModel 将 spec 全部 Scenario 落为 Go 测试：快速终答/工具轮继续、预算收敛、第三轮失败挽救/首轮失败上抛、空终答 nudge、摘要位次、三个 mounting 场景、load_tools、双 KB seed、事件序、think 跨 chunk、web_search 去重/ask_user 去重、videogen 子 trace 进度、窗口 snip、ask_user 续跑/未获回复、tool schema 回退
- [ ] 5.2 WS golden fixtures 回放对等验收：以 acceptance.md §3.1 录制的「普通 chat turn（content/thinking/tool_call/tool_result/sources/result/done）」与「ask_user 暂停→恢复」fixtures，经 mock provider 回放逐事件 diff（忽略 timestamp，seq 连续、顺序一致），对应 M1 矩阵「Agent Loop」行
