## Context

基线实现位于 `deeptutor/services/subagent/`（process/claude_code/codex/partner/registry/config/types/sessions/models）、`deeptutor/capabilities/subagent/tools.py` 与 `deeptutor/api/routers/subagents.py`。核心是三件事：(1) 以 asyncio 子进程逐行流式读取 CLI 的 stream-json/JSONL 输出并映射为统一 `SubagentEvent`；(2) `consult_subagent` 工具在 turn 内执行预算、把事件写入 event_sink（trace kind `subagent_event`）并维护跨 turn 会话注册表；(3) 约 10 条 REST（探测/模型目录/连接 CRUD/直连 NDJSON/设置）。

Go 版约束：子进程用标准库 `os/exec` 流式封装（任务技术约束）；consult 运行**无总超时**（spec 契约——只有子进程退出才终止流），取消依赖 `context.Context` 传播 + 进程组拆除；CLI 以宿主用户自身凭据运行，系统不经手 token。依赖 `impl-tools-builtin` 的工具协议（tool 定义 + event_sink）、`impl-knowledge` 的 KB 管理器（pointer KB）、`impl-partners` 的 partner runtime（partner backend）。

## Goals / Non-Goals

**Goals:**

- `os/exec` 子进程流式原语（逐行、exit 终项、SIGTERM→SIGKILL 拆除、探测独立超时）；
- `SubagentEvent`/`ConsultResult`/`DetectResult` 统一类型与 `claude_code`/`codex`/`partner` 三 backend；
- `consult_subagent` 工具（预算、trace 转发、会话注册表）；
- pointer KB 连接持久化与 `/api/v1/subagents` REST 面（含直连 NDJSON 流）。

**Non-Goals:**

- 不实现任何 agent CLI 本体或其安装管理（Claude Code / Codex 是用户机器上的运行期依赖）；
- 不做 token/凭据管理（CLI 用宿主用户自己的 `~/.claude`、`~/.codex`）；
- partner runtime 本体（由 `impl-partners` 交付；partner backend 在其之上薄封装）；
- 前端侧栏 UI（零改动复用，靠 `subagent_event` trace 事件面对等）。

## Decisions

### D1. Go 包结构

```
internal/subagent/
├── types.go          # SubagentEvent / ConsultResult / DetectResult / BackendConfig
├── process.go        # 子进程流式原语（os/exec + bufio 逐行、exit 终项、进程组拆除）
├── backend.go        # SubagentBackend interface + ConsultRequest
├── claude_code.go    # + claude_render.go（stream-json 映射、工具单行头渲染）
├── claude_models.go  # Claude Code 模型目录同步（对应基线 claude_models.py）
├── codex.go          # codex exec JSONL 映射与参数组装
├── partner.go        # partner backend（走 partner runtime）
├── registry.go       # backend 注册表 + DetectAll 并发探测
├── config.go         # subagent.json 读写、逐 backend 逐字段合并、budget 钳制
├── sessions.go       # 跨 turn 会话注册表（session_key → live session id）
└── connections.go    # pointer KB 连接 CRUD（经 KB 管理器）+ cwd 白名单校验
internal/tools/builtin/consult_subagent.go   # consult_subagent 工具（预算 + trace 转发）
internal/api/routers/subagents.go            # /api/v1/subagents REST + NDJSON 流
```

### D2. 子进程流式原语（os/exec）

```go
// StreamLine 是子进程输出的最小单元；Channel ∈ {"stdout","stderr","exit"}。
type StreamLine struct {
    Channel string
    Line    string // exit 时为 returncode 字符串
}

// StreamCommand spawn 命令并返回逐行流。
// - stdin 接 /dev/null；stdout/stderr 各一个 reader goroutine，按到达顺序写入同一 channel；
// - 行以 UTF-8 解码，非法字节 replace；
// - 无总超时：channel 只在进程退出后关闭，最后一项恒为 ("exit", code)；
// - ctx 取消 → 先 SIGTERM 进程组，宽限 5s 后 SIGKILL（POSIX 经 Setpgid+killpg 覆盖 CLI 自身 fork 的子进程，防孤儿）。
func StreamCommand(ctx context.Context, argv []string, opt CmdOptions) (<-chan StreamLine, error)

// DetectCommand 独立的带超时探测路径（默认 8s）；命令不存在 → "not installed"。
func DetectCommand(ctx context.Context, argv []string, timeout time.Duration) (version string, err error)
```

要点：无总超时是通过「不给 `StreamCommand` 传 deadline ctx」实现的，取消只来自 turn 取消；`exec.Cmd.SysProcAttr{Setpgid: true}`（POSIX）/ Job Object 或 `taskkill /T`（Windows，后置）保证整树拆除。

### D3. SubagentBackend interface

```go
type SubagentBackend interface {
    Kind() string        // "claude_code" | "codex" | "partner"
    DisplayName() string
    Detect(ctx context.Context) DetectResult // 走 DetectCommand 的 8s 超时路径；partner backend 恒可用

    // Consult 执行一轮咨询：事件经 emit 回调实时上抛（工具层负责写 event_sink / NDJSON），
    // 阻塞直至子进程退出或 ctx 取消；错误折叠进 ConsultResult 而非 panic/裸 error。
    Consult(ctx context.Context, req ConsultRequest, emit func(SubagentEvent)) ConsultResult
}

type ConsultRequest struct {
    Question  string
    SessionID string        // 非空则 resume
    Cwd       string        // 本地 CLI backend
    PartnerID string        // partner backend
    Config    BackendConfig // 服务端解析后的 per-backend 配置
    Images    []ImageRef    // forward_images 开启时
}

type Registry struct { /* ... */ }
func (r *Registry) Get(kind string) (SubagentBackend, bool)
func (r *Registry) DetectAll(ctx context.Context) []DetectResult // errgroup 并发探测本地 CLI backend
```

事件映射（claude_code 的 stream-json、codex 的 JSONL）实现为纯函数 `mapClaudeEvent(raw json.RawMessage, st *mapState) []SubagentEvent` / `mapCodexEvent(...)`，便于以基线录制的原始 CLI 输出做 golden case 表驱动单测；merge id、4000 字符截断、160 字符工具单行头等渲染规则集中在 `claude_render.go`。

### D4. consult_subagent 工具与会话注册表

- 工具定义仅暴露 `question` 参数；backend kind、cwd/partner_id、BackendConfig、turn 级状态经服务端注入（对应基线 `_subagent` 注入），依托 `impl-tools-builtin` 的工具协议中「服务端注入参数」机制。
- 预算与剩余提示在工具内权威执行；turn 级消耗计数与 session id 挂在 turn 状态上（`impl-turn-runtime` 提供 turn-scoped 存储）。
- 事件转发：`emit` 回调 → 包装为 trace kind `subagent_event` 写 event_sink，metadata 带 `subagent_kind`/`subagent_name`/`subagent_channel`/`consult_index`，merge id 加 `<consult_index>:` 前缀；直连 NDJSON 路径复用同一 `emit`，merge id 前缀改 `side:`。
- 会话注册表 `sessions.go`：`map[sessionKey]sessionID`（sessionKey = chat session + 连接名），进程内存态 + 惰性持久化，与基线语义一致（live session 记忆）。

### D5. REST 落位与设置合并

- gin route group `/api/v1/subagents`；`PUT /settings` 挂 admin 门控（`impl-multi-user-auth` 守卫）。
- NDJSON：`c.Writer` 逐行 `json.Encoder.Encode` + `Flush`，首行 `user_question`、终行 `done` 帧；backend 异常折叠为 `error` channel + `done{success:false}`。
- 设置合并：`PUT /settings` 逐 backend 逐字段合并（读现值 → 仅覆盖提交字段 → 原子写回 `data/user/settings/subagent.json`）；`consult_budget` 钳制 1–12、非法回落 5；读取失败回落全默认值。
- 连接 CRUD 经 KB 管理器写 `type: subagent` pointer KB；cwd 白名单与 partner 可见性校验分别复用 `impl-knowledge` 的路径策略与 `impl-multi-user-auth` 的 partner_access。

### D6. 与 Python 基线文件映射

| Python 基线 | Go 落位 |
| --- | --- |
| `deeptutor/services/subagent/process.py` | `internal/subagent/process.go` |
| `deeptutor/services/subagent/types.py` | `internal/subagent/types.go` |
| `deeptutor/services/subagent/base.py` | `internal/subagent/backend.go` |
| `deeptutor/services/subagent/claude_code.py` | `internal/subagent/claude_code.go` + `claude_render.go` |
| `deeptutor/services/subagent/claude_models.py`、`models.py` | `internal/subagent/claude_models.go` |
| `deeptutor/services/subagent/codex.py` | `internal/subagent/codex.go` |
| `deeptutor/services/subagent/partner.py` | `internal/subagent/partner.go` |
| `deeptutor/services/subagent/registry.py` | `internal/subagent/registry.go` |
| `deeptutor/services/subagent/config.py` | `internal/subagent/config.go` |
| `deeptutor/services/subagent/sessions.py` | `internal/subagent/sessions.go` |
| `deeptutor/services/subagent/images.py` | claude_code.go 内图片转发逻辑（临时目录 + `--add-dir`） |
| `deeptutor/capabilities/subagent/tools.py` | `internal/tools/builtin/consult_subagent.go` |
| `deeptutor/api/routers/subagents.py` | `internal/api/routers/subagents.go` |

## Risks / Trade-offs

- [CLI 输出格式随 Claude Code / Codex 版本演进] → 映射为纯函数 + 基线录制的 golden case 表驱动单测；未知事件按 spec 保留为 log 不丢弃，新增事件类型只降级不破坏。
- [无总超时 + CLI 卡在交互提示 = 挂死整个 turn] → BackendConfig 默认值（`bypassPermissions`、`workspace-write`+`never`）保证 headless 不进审批；用户中止 turn 的取消路径始终可拆除进程（spec Scenario 覆盖）。
- [进程组拆除的平台差异] → POSIX 用 `Setpgid` + `syscall.Kill(-pgid, …)`；Windows 形态随 `impl-cli-launcher` 的分发范围决定，首版验收平台为 macOS/Linux（acceptance.md 第 2 节），Windows 拆除留后置任务。
- [并发 consult 同一连接可能交叉污染 session 注册表] → 注册表写入持锁；turn 内 session id 挂 turn 状态、注册表只在 consult 完成后写回，与基线时序一致。
- [pointer KB 依赖 KB 管理器语义（不索引、composer 可选）] → 与 `impl-knowledge` 约定 `type: subagent` 条目跳过索引管线，本 change 增加契约测试防回归。
