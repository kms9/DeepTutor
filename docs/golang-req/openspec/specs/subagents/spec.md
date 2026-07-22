# Subagents Specification

## Purpose
本模块定义 deeptutor-go 的 Subagent（外部 Worker Agent）子系统：主 Agent 在 chat turn 中通过 `consult_subagent` 工具向用户机器上已安装的 agent CLI（Claude Code / Codex）或某个 partner 提问，子进程的原生事件流实时转发到前端侧栏，最终回答作为 tool result 返回给主模型。模块包含 backend 抽象与注册表、子进程流式封装、跨 turn 会话记忆，以及 `/api/v1/subagents` REST 面（约 10 条：探测、模型目录、连接管理、直连消息、设置）。CLI 以宿主用户自身凭据运行（`~/.claude`、`~/.codex`），系统 SHALL NOT 经手任何 token。

- 参考实现（基线）：`deeptutor/services/subagent/`（`process.py`、`claude_code.py`、`codex.py`、`partner.py`、`registry.py`、`config.py`、`types.py`、`sessions.py`、`models.py`）、`deeptutor/capabilities/subagent/tools.py`、`deeptutor/api/routers/subagents.py`
- 依赖 spec / 里程碑：依赖：agent-loop、turn-runtime、knowledge-kb-manager（连接以 pointer KB 持久化）、partners（partner backend）；里程碑：M4

## Requirements

### Requirement: 子进程流式封装与无超时等待契约
系统 SHALL 提供统一的子进程流式原语：spawn 命令后按到达顺序逐行产出 stdout/stderr（`(channel, line)`），最后一项恒为 `("exit", "<returncode>")`；stdin 接 `/dev/null`，行以 UTF-8 解码（非法字节 replace）。对 consult 运行 SHALL NOT 设置总超时——产品契约是无条件等待 subagent 自身逻辑结束，只有子进程退出（正常或出错）才终止流；用户中止 turn 的取消 SHALL 传播到消费方并在清理阶段拆除子进程（先 `SIGTERM`，宽限 5s 后 `SIGKILL`），SHALL NOT 遗留孤儿进程。版本探测 SHALL 走独立的带超时路径（默认 8s，超时即视为探测失败），命令不存在返回 `not installed`。

#### Scenario: 用户中止 turn 时子进程被拆除
- **WHEN** consult 进行中用户停止该 chat turn
- **THEN** 取消传播到流消费方，子进程收到 terminate，5s 内未退出则被 kill，无孤儿进程残留

#### Scenario: 长运行不被超时打断
- **WHEN** Codex 一次运行耗时 20 分钟
- **THEN** 系统持续转发其事件直至进程自行退出，不因时长中断

### Requirement: 统一事件模型与结果类型
系统 SHALL 定义 backend 与上层之间唯一的越界类型：`SubagentEvent{kind, text, raw, meta}`，其中 `kind` 为七个粗粒度 channel 之一——`text`、`reasoning`、`tool`、`tool_result`、`log`、`result`、`error`；`raw` 保留 CLI 原始 JSON 对象不丢弃；`meta.merge_id` 为可选 UI 提示，用于把同一工具的 start/finish 或流式增量折叠为一行渐进更新。consult 结果 SHALL 为 `ConsultResult{final_text, session_id, success, error, event_count}`；探测结果为 `DetectResult{kind, display_name, available, version, detail}`。

#### Scenario: merge_id 折叠流式回答
- **WHEN** backend 对同一 content block 连续发出多条携带相同 `merge_id` 的 `text` 事件（累计文本）
- **THEN** 前端侧栏保留每个 merge_id 的最新文本，回答呈打字机式渐进，完成帧就地定格同一行

### Requirement: Claude Code backend
系统 SHALL 实现 `claude_code` backend（`display_name` "Claude Code"）：探测用 `claude --version`；consult 用 `claude -p <prompt> --output-format stream-json --verbose --include-partial-messages`，按配置追加 `--resume <session_id>`、`--permission-mode`（默认 `bypassPermissions`）、`--model`、`--effort`、`--append-system-prompt`、`extra_args`。stream-json 事件映射 SHALL 与基线一致：`system/init` → log（含模型名）；`assistant` 的 text/tool_use/thinking block → text/tool/reasoning（merge id `{txt|rsn}:{msg_id}:{index}`）；`user` 的 tool_result → tool_result；`stream_event` 的 text_delta/thinking_delta 累积后以同一 merge id 发出实现 token 级流式；`result` 事件的 `result` 字段为 final_text（已流出过 assistant 文本时不重复 emit）；`rate_limit_event`/`control_request`/`control_response` 丢弃；未知事件保留为 log。工具调用渲染为 CLI 风格单行头（`Bash(cmd)`、`Read(path)`，首个命中的主参数，单行 ≤160 字符）；事件文本统一截断至 4000 字符。退出码非 0 且尚无 final_text 时 SHALL 置 `success=false` 并发 error 事件 `claude exited with code <n>`；stderr 行转发为 log。图片转发（`forward_images` 开启时）SHALL 落盘临时目录、在 prompt 尾部附加「用 Read 工具查看」的路径清单并加 `--add-dir <该目录>`，运行结束后删除临时目录。

#### Scenario: 恢复既有会话
- **WHEN** 同一 turn 内第二次 consult 且上次返回了 `session_id`
- **THEN** 命令行携带 `--resume <session_id>`，agent 延续上下文回答

#### Scenario: 非零退出映射为失败
- **WHEN** `claude` 以退出码 1 结束且未产出 result 事件
- **THEN** `ConsultResult.success=false`、`error="claude exited with code 1"`，且一条 error 事件已推给侧栏

### Requirement: Codex backend
系统 SHALL 实现 `codex` backend（`display_name` "Codex"）：探测用 `codex --version`；consult 用 `codex exec --json --skip-git-repo-check`，续接用 `codex exec resume <session_id>`。配置映射 SHALL 与基线一致：`sandbox`（默认 `workspace-write`）经 `-c sandbox_mode="…"`，bypass 值集（`bypass`、`danger-full-access-bypass`、`dangerously-bypass-approvals-and-sandbox`）改用 `--dangerously-bypass-approvals-and-sandbox`；`approval`（默认 `never`）经 `-c approval_policy="…"`；`network_access` 经 `-c sandbox_workspace_write.network_access=true`；`--ephemeral` 仅在非 resume 时附加；`-m <model>`、`-c model_reasoning_effort="…"`；图片以逐文件 `-i <path>` 附加；prompt 为最后一个位置参数。JSONL 事件映射 SHALL：`thread.started` → log 并捕获 session id（`thread_id`/`session_id` 及 camelCase 变体）；`item.*.updated` 中 `agent_message`/`assistant_message`/`reasoning` 的增量文本以 item id 为 merge id 流式发出；`item.*.completed` 的 `agent_message` 文本为 final_text；`command_execution` 的 start 渲染命令行、completed 渲染输出 + exit code（两行独立）；`web_search` start 出占位、completed 填入 query（同一 merge id）；`file_change`/`mcp_tool_call` → tool；`turn.failed`/`error` → `success=false` + error 事件；未识别 item 渲染为 log 而非丢弃。退出码非 0 且无 final_text 时同样映射为失败。

#### Scenario: 沙箱与审批默认不阻塞
- **WHEN** 未做任何 Codex 配置发起 consult
- **THEN** 命令包含 `-c sandbox_mode="workspace-write"` 与 `-c approval_policy="never"`，headless 运行不会卡在审批提示

#### Scenario: 命令执行渲染为两行
- **WHEN** Codex 运行中执行了一条 shell 命令
- **THEN** 侧栏先出现 `$ <command>` 的 tool 行，完成后出现含输出与 `(exit <code>)` 的 tool_result 行

### Requirement: backend 注册表与 partner backend
系统 SHALL 维护 backend 注册表：`claude_code`、`codex` 两个本地 CLI backend，外加 `partner` backend（consult 目标为本机某个 partner，走 partner runtime 而非子进程）。`detect_all` SHALL 并发探测全部本地 CLI backend 并返回 DetectResult 列表；`get_backend(kind)` 按 kind 取 backend，未知 kind 返回空。partner backend SHALL 以 `partner_id` 而非 `cwd` 定位目标，consult 即在该 partner 上开（或续接）一个会话，与用户从 partner 页面发起的会话等价。

#### Scenario: 未知 backend kind 被拒绝
- **WHEN** 以 `agent_kind="foo"` 创建连接
- **THEN** 返回 400 `Unknown agent kind: 'foo'`

### Requirement: consult_subagent 工具与轮次预算
系统 SHALL 提供 `consult_subagent` 工具，仅当所选 KB 为 subagent 连接时由 subagent capability 自动挂载并独占该 turn。模型可见参数 SHALL 仅有 `question`；backend kind、cwd/partner_id、解析后的 BackendConfig、turn 级状态（消耗计数与 session id）由服务端注入（`_subagent`），SHALL NOT 由模型提供。预算 SHALL 在工具内权威执行：每 turn 最多 `consult_budget` 次（默认 5，钳制在 1–12），超出时拒绝执行并返回指示模型自行作答的文本（`success=false`）；每次成功返回的内容 SHALL 在正文后附剩余次数提示（`[N consult(s) left …]` 或 `[No consults left …]`）。事件转发 SHALL：每条 SubagentEvent 以统一 trace kind `subagent_event` 写入 event_sink，metadata 携带 `subagent_kind`、`subagent_name`、`subagent_channel`、`consult_index`，merge id 加 `<consult_index>:` 前缀防跨轮碰撞；每轮开头先发一条 `question` channel 事件（DeepTutor 提给 agent 的问题）使 transcript 呈对话状。final_text 为空时 SHALL 返回 `success=false` 的 `[The agent returned no answer: <error 或占位>]`；backend 抛异常 SHALL 折叠为失败 tool result，SHALL NOT 让 turn 崩溃。返回的 `session_id` SHALL 写回 turn 状态并按 `session_key`（chat session + 连接名）记入跨 turn 会话注册表，使下一 turn 与侧栏直连续接同一 agent 会话。

#### Scenario: 预算耗尽后拒绝
- **WHEN** 模型在同一 turn 第 6 次调用 `consult_subagent`（budget=5）
- **THEN** 工具不执行 CLI，返回「预算已达上限、立即作答」的失败结果

#### Scenario: 跨 turn 续接同一会话
- **WHEN** 上一 turn 的 consult 已在会话注册表登记 session id，用户在同一 chat session 发起新 turn
- **THEN** 本 turn 首次 consult 即携带该 session id resume，agent 保有此前上下文

### Requirement: 连接以 pointer KB 持久化
系统 SHALL 把 subagent 连接存储为 `type: subagent` 的知识库条目（per-user，经 KB 管理器），metadata 含 `agent_kind`、`cwd`、`partner_id`、时间戳；不做任何索引，仅作为聊天 composer 可选择的指针。创建时 SHALL 校验：`name` 与 `agent_kind` 必填（400）；kind 必须已注册（400）；本地 CLI 连接的 `cwd` 非空时须通过路径白名单校验（越界 400）；partner 连接的 `partner_id` 必填、须通过当前用户的 partner 可见性检查（非授权 403）且 partner 存在（400）。删除连接 SHALL 移除该 pointer KB（不触碰任何文件）并清掉该连接名下所有已记住的 live session id。

#### Scenario: cwd 越界被拒
- **WHEN** 创建 claude_code 连接时 `cwd` 指向白名单外的路径
- **THEN** 返回 400 且不创建连接

### Requirement: /api/v1/subagents REST 面
系统 SHALL 暴露以下路由（约 10 条）：
- `GET /detect` — 并发探测本机各 CLI 可用性，返回 `{"backends": [DetectResult...]}`
- `GET /backends/options` — 各 backend 已同步的 model + reasoning-effort 选项（设置页）
- `POST /backends/{kind}/sync` — 重拉某 backend 的模型目录（仅本地 CLI backend，partner kind 返回 400）
- `GET /partners` — 当前用户可连接/可 consult 的 partner 卡片（admin 全量、非 admin 仅被指派者；与 admin 门控的 partner CRUD 面区分）
- `GET /connections` / `POST /connections` / `DELETE /connections/{name}` — 连接 CRUD
- `POST /connections/{name}/message` — 直连对话（见下一条 Requirement）
- `GET /settings` / `PUT /settings` — 读取/更新 subagent 设置，PUT 为 admin 门控（部署级）

`PUT /settings` SHALL 逐 backend、逐字段合并（保存一个 backend 的设置不得覆盖另一 backend 或未提交字段）；`consult_budget` 越界值钳制到 1–12，非法值回落默认 5。设置持久化于 `data/user/settings/subagent.json`，读取失败时 SHALL 回落全默认值（功能在 CLI 检出后零配置可用）。

#### Scenario: 部分更新不覆盖其他 backend
- **WHEN** `PUT /settings` 只提交 `{"backends": {"codex": {"model": "o4"}}}`
- **THEN** `claude_code` 的既有配置与 `codex` 未提交字段全部保留，仅 `codex.model` 更新

### Requirement: 直连消息流（NDJSON）
系统 SHALL 支持绕过主模型直接与已连接 subagent 对话：`POST /connections/{name}/message` 以 `application/x-ndjson` 流式返回该次运行。请求带 `chat_session_id` 时 SHALL 经会话注册表 resume 与主聊天 consult 相同的 live session（agent 保持完整上下文），完成后把新 session id 记回注册表。流内容 SHALL：首行 `{"channel": "user_question", "text": <消息>}`；随后每条事件为 `{"channel": <kind>, "text": ...}`（merge id 加 `side:` 前缀与主聊天 consult 的 merge id 隔离命名空间）；结束行 `{"done": true, "success": ..., "session_id": ...}`；backend 异常折叠为 `{"channel":"error"}` + `{"done":true,"success":false}`。空 message 返回 400，连接不存在返回 404。

#### Scenario: 侧栏直连续接主聊天会话
- **WHEN** 主聊天某 turn consult 过连接 A，用户随后在侧栏对 A 发消息并携带同一 `chat_session_id`
- **THEN** 该次运行 resume 相同的 agent session，NDJSON 流以 user_question 开头、done 帧结尾

### Requirement: BackendConfig 字段与默认值
per-backend 配置 SHALL 含：`enabled`（默认 true）、`model`/`effort`（空 = CLI 自身默认）、`system_prompt`（CC 经 `--append-system-prompt` 注入的被程序化咨询提示）、`permission_mode`（CC，默认 `bypassPermissions`）、`sandbox`（Codex，默认 `workspace-write`）、`approval`（Codex，默认 `never`）、`network_access`（默认 false）、`ephemeral`（默认 false）、`forward_images`（默认 false，用户逐 backend 显式开启）、`extra_args`（逃生舱）。默认值 SHALL 保证 headless 运行永不阻塞在审批提示上（阻塞即挂死整个 turn，因等待无超时）。

#### Scenario: 默认配置零阻塞
- **WHEN** 用户仅连接 CLI 未做任何配置即发起 consult
- **THEN** Claude Code 以 `bypassPermissions`、Codex 以 `workspace-write`+`never` 运行，全程无审批停顿
