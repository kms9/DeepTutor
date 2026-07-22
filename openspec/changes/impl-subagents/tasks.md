## 1. 子进程原语与类型

- [ ] 1.1 实现 `internal/subagent/types.go`：`SubagentEvent{kind,text,raw,meta}`（七 channel）、`ConsultResult`、`DetectResult`、`BackendConfig`（含全部默认值：`bypassPermissions`、`workspace-write`、`never`、`forward_images=false` 等）
- [ ] 1.2 实现 `process.go` `StreamCommand`：os/exec spawn、stdin=/dev/null、stdout/stderr 逐行（UTF-8 replace）、末项 `("exit", code)`、无总超时
- [ ] 1.3 实现取消拆除：ctx 取消 → SIGTERM 进程组（`Setpgid`）→ 5s 宽限 → SIGKILL，无孤儿进程（用嵌套 fork 的测试脚本验证）
- [ ] 1.4 实现 `DetectCommand`：独立 8s 超时探测、命令不存在返回 `not installed`

## 2. Backend 实现

- [ ] 2.1 定义 `backend.go` `SubagentBackend` interface + `ConsultRequest`；实现 `registry.go`（`Get(kind)`、`DetectAll` 并发探测、未知 kind 返回空）
- [ ] 2.2 实现 `claude_code.go` 参数组装：`claude -p … --output-format stream-json --verbose --include-partial-messages` + `--resume`/`--permission-mode`/`--model`/`--effort`/`--append-system-prompt`/`extra_args`
- [ ] 2.3 实现 claude stream-json 事件映射（`mapClaudeEvent` 纯函数）：system/init→log、assistant block→text/tool/reasoning（merge id `{txt|rsn}:{msg_id}:{index}`）、tool_result、delta 累积 token 级流式、result→final_text 不重复 emit、rate_limit/control 丢弃、未知→log
- [ ] 2.4 实现 `claude_render.go`：工具 CLI 风格单行头（主参数命中、≤160 字符）、事件文本 4000 字符截断；非零退出 + 无 final_text → `success=false` + error 事件；stderr→log
- [ ] 2.5 实现图片转发（`forward_images`）：临时目录落盘、prompt 尾部路径清单、`--add-dir`、运行后清理
- [ ] 2.6 实现 `codex.go` 参数组装：`codex exec --json --skip-git-repo-check` / `resume <id>`、sandbox/bypass 值集、approval、network_access、`--ephemeral` 仅非 resume、`-m`/effort、`-i` 图片、prompt 末位
- [ ] 2.7 实现 codex JSONL 事件映射：thread.started 捕获 session id（snake/camel 变体）、item updated 增量流式（item id 为 merge id）、completed agent_message→final_text、command_execution 两行渲染、web_search 占位填充、file_change/mcp_tool_call→tool、turn.failed/error→失败、未识别→log
- [ ] 2.8 实现 `claude_models.go` 模型目录同步与 `partner.go` partner backend（以 `partner_id` 定位、经 partner runtime 开/续会话）

## 3. consult_subagent 工具与会话

- [ ] 3.1 实现 `internal/tools/builtin/consult_subagent.go`：模型仅见 `question`，服务端注入 backend/config/turn 状态；subagent capability 挂载条件（所选 KB 为 subagent 连接时独占 turn）
- [ ] 3.2 实现预算：`consult_budget`（默认 5，钳 1–12）权威执行、超出拒绝并指示自行作答、成功结果附 `[N consult(s) left …]`/`[No consults left …]`
- [ ] 3.3 实现事件转发：`subagent_event` trace 写 event_sink（metadata 四字段、merge id `<consult_index>:` 前缀）、每轮开头 `question` channel 事件
- [ ] 3.4 实现失败折叠：final_text 为空返回 `[The agent returned no answer: …]`（success=false）、backend 异常折叠为失败 tool result 不崩 turn
- [ ] 3.5 实现 `sessions.go` 跨 turn 会话注册表：session id 写回 turn 状态 + 按 `session_key` 登记，下一 turn / 侧栏直连续接

## 4. 连接持久化与 REST 面

- [ ] 4.1 实现 `connections.go`：`type: subagent` pointer KB CRUD（不索引）、创建校验（name/kind 必填 400、kind 已注册、cwd 白名单越界 400、partner_id 可见性 403/存在 400）、删除清 live session
- [ ] 4.2 实现 `config.go`：`data/user/settings/subagent.json` 读写、读取失败回落全默认、逐 backend 逐字段合并、budget 钳制
- [ ] 4.3 实现 `internal/api/routers/subagents.go`：`GET /detect`、`GET /backends/options`、`POST /backends/{kind}/sync`（partner kind 400）、`GET /partners`（admin 全量/非 admin 被指派）、connections CRUD、`GET/PUT /settings`（PUT admin 门控）
- [ ] 4.4 实现 `POST /connections/{name}/message` NDJSON 直连流：首行 `user_question`、事件行（merge id `side:` 前缀）、终行 done 帧、异常折叠、空 message 400、连接不存在 404、`chat_session_id` resume 与写回

## 5. 测试与验收（spec Scenario 落测试）

- [ ] 5.1 process 原语 Scenario 测试：中止 turn 拆除子进程（terminate→kill、无孤儿）、20 分钟级长运行不被打断（缩尺模拟：慢输出脚本 + 无超时断言）
- [ ] 5.2 事件模型测试：相同 merge_id 折叠渐进更新（累计文本序列断言）
- [ ] 5.3 claude_code golden case 测试（基线录制 stream-json 回放）：resume 参数、非零退出映射失败、渲染截断规则
- [ ] 5.4 codex golden case 测试（JSONL 回放）：默认 sandbox/approval 参数断言、command_execution 两行渲染、session id 捕获
- [ ] 5.5 工具层 Scenario 测试：budget=5 第 6 次拒绝、跨 turn resume、空回答与异常折叠、trace metadata 与 merge id 前缀
- [ ] 5.6 连接与 REST Scenario 测试：未知 kind 400、cwd 越界 400、`PUT /settings` 部分更新不覆盖其他 backend、NDJSON 帧序（user_question→事件→done）、直连续接主聊天会话
- [ ] 5.7 协议对等验收：`/api/v1/subagents` 全路由 golden spec contract test；`subagent_event` trace 事件面与基线 fixtures 对比
- [ ] 5.8 真实 CLI e2e（M4 验收矩阵，需本机安装）：Claude Code 与 Codex 各完成一次真实 consult，子进程流式输出进入 tool result 与前端侧栏
