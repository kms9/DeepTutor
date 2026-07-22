## Why

deeptutor-go 需要迁移 Subagent（外部 Worker Agent）子系统：主 Agent 在 chat turn 中通过 `consult_subagent` 工具向用户机器上已安装的 agent CLI（Claude Code / Codex）或某个 partner 提问，子进程原生事件流实时转发到前端侧栏，最终回答作为 tool result 返回主模型。行为规格见 `docs/golang-req/openspec/specs/subagents/spec.md`；按 `docs/golang-req/openspec/ROADMAP.md`，本模块属于 Wave 4、里程碑 M4（依赖图：tools-builtin → subagents）。CLI 以宿主用户自身凭据运行（`~/.claude`、`~/.codex`），系统不经手任何 token。

## What Changes

- 新增子进程流式原语：`os/exec` spawn 后按到达顺序逐行产出 stdout/stderr，末尾恒为 exit 项；consult 运行无总超时（等 subagent 自身逻辑结束），取消时 SIGTERM → 5s 宽限 → SIGKILL 不遗留孤儿进程；版本探测走独立 8s 超时路径。
- 新增统一事件模型：`SubagentEvent{kind, text, raw, meta}`（七个粗粒度 channel），`ConsultResult` / `DetectResult` 结果类型，`merge_id` 折叠流式增量。
- 新增 `claude_code` backend：`claude -p … --output-format stream-json` 事件映射（text/tool/reasoning/tool_result、token 级流式、result、非零退出映射失败、图片转发临时目录）。
- 新增 `codex` backend：`codex exec --json` JSONL 事件映射（thread.started 捕获 session id、item 增量流式、command_execution 两行渲染、sandbox/approval 默认不阻塞）。
- 新增 backend 注册表与 `partner` backend（consult 目标为本机 partner，走 partner runtime 而非子进程）、`detect_all` 并发探测。
- 新增 `consult_subagent` 工具：模型仅见 `question` 参数，服务端注入 backend 状态；`consult_budget` 权威预算（默认 5，1–12 钳制）、剩余次数提示、`subagent_event` trace 转发、跨 turn 会话注册表续接。
- 新增连接以 pointer KB 持久化（`type: subagent`，cwd 白名单 / partner 可见性校验）与 `/api/v1/subagents` REST 面（约 10 条：探测、模型目录、连接 CRUD、直连 NDJSON 消息、设置读写）。

## Capabilities

### New Capabilities

- `subagents`: Subagent 子系统全部行为——子进程流式封装与无超时等待契约、统一事件模型、claude_code / codex / partner backend、注册表、consult_subagent 工具与轮次预算、pointer KB 连接持久化、REST 面、直连 NDJSON 消息流、BackendConfig 默认值。

### Modified Capabilities

（无——本 change 不修改既有 spec 的 Requirement。）

## Impact

- 依赖的其他 change（按 ROADMAP 依赖图，合入前需已验收）：
  - `impl-tools-builtin`：`consult_subagent` 以内置工具形态接入工具挂载管线（tools → sub 边）。
  - 间接/关联：`impl-agent-loop`、`impl-turn-runtime`（事件写入 event_sink、turn 级状态与取消传播）、`impl-knowledge`（连接以 pointer KB 经 KB 管理器持久化）、`impl-partners`（partner backend 以 partner runtime 为 consult 目标；partner backend 可后于本 change 主体接入）。
- 新增 Go 包：`internal/subagent/`（process、backends、registry、sessions、config）、`internal/tools/` 下 `consult_subagent` 工具、`internal/api/` 下 subagents router。
- 外部依赖：仅标准库 `os/exec`（子进程）+ 既有 gin（REST/NDJSON），不引入新 SDK；被调用的 Claude Code / Codex CLI 是用户机器上的运行期依赖，非构建依赖。
- 数据面：设置持久化于 `data/user/settings/subagent.json`；连接为 per-user pointer KB；均与基线布局对等。
- 前端影响：无——`/api/v1/subagents` REST 与 `subagent_event` trace 事件面逐字段对等，`web/` 零改动。
