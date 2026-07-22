## Context

基线实现为 `deeptutor_cli/`（main/chat/kb/memory/common/config_cmd/session_cmd/partner 等，Typer + rich）与 `deeptutor/runtime/launcher.py`（933 行：双进程编排、端口发现、前端运行时解析、信号清理）+ `deeptutor/runtime/home.py`（runtime home 解析）。Go 版两个结构性变化（均已登记 acceptance.md 第 6 节）：

- **D-006**：`start` 从「backend + frontend 双进程」变为「Go 主服务 + Python Sidecar + 前端三进程」编排（Sidecar 缺失降级警告不阻塞）；分发从 pip 三包（`deeptutor`/`deeptutor-cli`/`deeptutor-web`）变为 Go 静态二进制 + Sidecar venv + Docker 单容器。
- **D-007**：`deeptutor init` 交互向导首期后置，由 `start` 自动初始化（建目录 + 默认 settings）覆盖必要子集。

本模块是 M5（交付面收口）唯一 change，依赖 `impl-turn-runtime`（进程内执行 turn）、`impl-partners`（partner 子命令与 auto-start）、`impl-sidecar-contract`（Sidecar 进程与健康检查）等全部上游已验收。

## Goals / Non-Goals

**Goals:**

- cobra 命令树：`run`/`chat`/`kb`/`memory`/`serve`/`start`/`partner`/`config`/`session` + 后置子命令注册机制；
- `run`/`chat` 进程内 SDK 等价执行 turn（无需起服务），REPL 全套斜杠命令（含 `/regenerate`）；
- launcher：三进程编排、端口发现与占用交互、前端运行时解析（打包占位符替换 / 源码 dev server 复用）、信号清理；
- 分发：单机（二进制 + venv + standalone 前端）与 Docker 单容器。

**Non-Goals:**

- `deeptutor init` 向导（D-007 后置）；`serve --reload` 开发热重载（可标后置）；
- `skill`/`plugin`/`notebook`/`provider`/`book` 子命令组（随对应模块里程碑，本 change 只留注册机制）；
- Sidecar 本体与其 venv 内容（`impl-sidecar-contract` 交付，这里只负责拉起/健康检查/安装指引）；
- Windows 作为首版验收平台（验收环境为 macOS/Linux，Windows 信号/进程组语义留接口）。

## Decisions

### D1. Go 包结构

```
cmd/deeptutor/
├── main.go            # cobra root（无参数输出帮助、退出码 0）
├── root.go            # 全局 flag、子命令注册机制（RegisterCommand 供后置命令组接入）
├── run.go             # deeptutor run
├── chat.go            # REPL（readline 循环、斜杠命令表）
├── kb.go              # kb 子命令组
├── memory.go          # memory 子命令组
├── serve.go           # serve
├── start.go           # start（调 launcher）
├── partner.go         # partner 子命令组
├── config_cmd.go      # config
├── session_cmd.go     # session
└── render.go          # rich 风格终端渲染 + --format json 输出
internal/runtime/launcher/
├── launcher.go        # Launch()：编排主流程（初始化 → 端口 → 三进程 → banner → 监督循环）
├── home.go            # runtime home 解析（--home > DEEPTUTOR_HOME > 默认）+ 自动初始化
├── process.go         # ManagedProcess：进程组启动、前缀日志泵、终止（TERM→8s→KILL）
├── ports.go           # 端口探测、占用进程列举（lsof/netstat）、TTY 交互、持久化
├── frontend.go        # 前端运行时解析：打包缓存+占位符替换+marker；源码 dev server 检测复用
├── sidecar.go         # Sidecar venv 发现、拉起、健康检查、缺失降级
├── health.go          # HTTP 健康检查轮询（超时、子进程早退检测）
└── signals.go         # 信号注册、幂等清理（sync.Once）、atexit 兜底
```

### D2. CLI 框架与渲染选型

- CLI：`spf13/cobra`（项目约束）。子命令注册机制：`root.go` 暴露 `RegisterCommand(cmd *cobra.Command)`，后置命令组（skill/plugin/notebook/provider/book）作为独立文件 `init()` 注册即可接入，无需改 root。
- 终端渲染：`charmbracelet/lipgloss` + `glamour`（markdown 渲染），对齐基线 rich 的面板/表格/流式体验；REPL 行编辑用 `chzyer/readline`（Ctrl-C 中断语义可控）。备选 `pterm`（组件更全但样式定制弱）；渲染层收敛在 `render.go`，选型可替换。
- `run`/`chat` 走进程内 SDK 路径：直接构装 orchestrator + turn runtime（与 API 服务同一套 `internal/runtime` 装配函数），不经 HTTP。config 值解析：`json.Unmarshal` → `true/false/null` 字面量 → 字符串。

### D3. Launcher 关键接口（D-006 三进程）

```go
// ManagedProcess 封装一个受管子进程。
type ManagedProcess struct {
    Name   string   // "backend" | "sidecar" | "frontend"（日志前缀）
    Cmd    *exec.Cmd // SysProcAttr: POSIX Setsid（等价基线 start_new_session）；Windows CREATE_NEW_PROCESS_GROUP
}
func (p *ManagedProcess) Start(ctx context.Context) error        // 启动 + stdout/stderr 前缀泵 goroutine
func (p *ManagedProcess) Terminate(grace time.Duration) error     // killpg SIGTERM → grace(8s) → SIGKILL
func (p *ManagedProcess) Exited() <-chan int                      // 监督循环消费

type LaunchPlan struct {
    Home         string            // runtime home（已初始化）
    BackendPort  int
    FrontendPort int
    Env          map[string]string // DEEPTUTOR_HOME/BACKEND_PORT/FRONTEND_PORT/PORT/
                                   // NEXT_PUBLIC_API_BASE/NEXT_PUBLIC_AUTH_ENABLED/
                                   // DEEPTUTOR_API_BASE_URL/DEEPTUTOR_AUTH_ENABLED …
}

// Launch：初始化 → 端口解析 → 依序拉起 backend/sidecar/frontend →
// 健康检查（backend 60s、frontend 120s、sidecar 失败降级警告）→ banner → 监督循环。
// 任一受管进程意外退出 → 逆序清理 → 返回使进程以退出码 1 结束。
func Launch(ctx context.Context, plan LaunchPlan) error
```

- Go 主服务进程形态：`start` 默认以 **同一二进制** re-exec `deeptutor serve --host … --port …` 作为受管子进程（保持三进程模型对称、崩溃隔离、日志前缀统一），而非 in-process 起服务。理由：与基线 launcher 语义一致（backend 是受管子进程），监督循环/退出码语义无需特判。
- Sidecar 发现顺序：`DEEPTUTOR_SIDECAR_PYTHON` 显式指定 → 安装布局约定路径（与二进制同级的 `sidecar-venv/`）→ 源码 checkout venv；全部缺失 → 打印安装指引警告、跳过该进程（D-006 降级语义）。
- 健康检查：backend 轮询 `/api/v1/system/health`，frontend 轮询 `http://127.0.0.1:<port>/`，等待期间 select 监听 `Exited()` 实现「早退立即报错」。

### D4. 端口与前端运行时

- `ports.go`：TCP connect 探测；占用列举 POSIX 用 `lsof -nP -iTCP:<port> -sTCP:LISTEN`（Windows netstat+tasklist 留接口）；TTY 判定 `golang.org/x/term.IsTerminal`；换端口从冲突口 +1 向上找空闲且与另一口互斥，写回 system.json（经 `impl-foundation-config` 的 settings 写路径）；终止占用进程 SIGTERM→5s→KILL→3s 再验。`serve --port` 显式指定跳过交互。
- `frontend.go`：
  - 打包模式：standalone 产物复制到 `data/user/runtime/web/`，全树文本替换 `__NEXT_PUBLIC_API_BASE_PLACEHOLDER__` / `__NEXT_PUBLIC_AUTH_ENABLED_PLACEHOLDER__`；marker 文件记录 `{source_path, source_mtime, api_base, auth_enabled}`，命中即跳过复制；Node 20+ 检测（`node --version`），缺 Node 报错退出。
  - 源码模式：`web/package.json` 存在即 `npm run dev -- --port <port>`；启动前读 `web/.next/dev/lock` 的 pid/port——进程存活且健康检查通过则复用（跳过前端端口冲突检查）；lock 指向确为 Next 进程但不健康则终止重启；否则报错。

### D5. 信号与清理

- `signals.go`：`signal.Notify(SIGINT, SIGTERM, SIGHUP)`（Windows SIGBREAK 留接口）；清理经 `sync.Once` 幂等；顺序 前端 → Sidecar → 主服务，每个 `Terminate(8s)`；`defer` 链兜底（Go 无 atexit，以 main 顶层 defer + panic recover 覆盖全部退出路径）。Ctrl+C 即 SIGINT 同路径。

### D6. 分发形态（D-006）

| 形态 | 组成 | 编排 |
| --- | --- | --- |
| 本机安装 | `deeptutor` 静态二进制（CGO_ENABLED=0，含 CLI+服务）+ `sidecar-venv/`（安装脚本 `python -m venv` + `pip install deeptutor-sidecar`）+ `web-standalone/` 前端产物 | `deeptutor start` 按 D3 发现顺序自动定位三者 |
| Docker 单容器 | 单镜像分层：Go 二进制 / Sidecar venv / 前端产物；入口进程即 `deeptutor start`（非 TTY 语义：端口冲突直接失败） | 数据经单一 `data/` 卷挂载 |

基线 pip 三包（`deeptutor`/`deeptutor-cli`/`deeptutor-web`）不复刻；命令面与数据布局不变。`deeptutor init` 后置（D-007），`home.go` 的自动初始化（建目录 + 默认 settings 生成）覆盖其必要子集。

### D7. 与 Python 基线文件映射

| Python 基线 | Go 落位 |
| --- | --- |
| `deeptutor_cli/main.py` | `cmd/deeptutor/main.go` + `root.go` |
| `deeptutor_cli/chat.py` | `cmd/deeptutor/chat.go` |
| `deeptutor_cli/kb.py` | `cmd/deeptutor/kb.go` |
| `deeptutor_cli/memory.py` | `cmd/deeptutor/memory.go` |
| `deeptutor_cli/common.py`、`_tool_result.py` | `cmd/deeptutor/render.go` |
| `deeptutor_cli/config_cmd.py`、`session_cmd.py`、`partner.py` | `cmd/deeptutor/{config_cmd,session_cmd,partner}.go` |
| `deeptutor_cli/init_cmd.py`、`init_wizard.py` | 不迁移（D-007 后置；必要子集入 `launcher/home.go`） |
| `deeptutor/runtime/launcher.py` | `internal/runtime/launcher/{launcher,process,ports,frontend,health,signals}.go` |
| `deeptutor/runtime/home.py` | `internal/runtime/launcher/home.go` |
| （基线无）Sidecar 编排 | `internal/runtime/launcher/sidecar.go`（D-006 新增） |

## Risks / Trade-offs

- [三进程编排的失败组合多（Sidecar 缺 venv / 前端缺 Node / 端口交互）] → 每条失败路径有确定性错误文案与退出码；launcher e2e 用假子进程（sleep/立即退出/占端口脚本）矩阵化测试。
- [re-exec 自身作为 backend 增加一层进程] → 换来与基线一致的监督/退出码语义与日志前缀统一；资源开销可忽略；若后续要 in-process 可在 `Launch` 内部替换而不动 CLI 面。
- [Go 无 atexit，panic 路径可能漏清理] → main 顶层 `defer cleanup()` + `recover`；清理幂等（sync.Once）；e2e 断言「launcher 任何退出路径无子进程残留」（`pgrep` 检查）。
- [占位符替换遍历 standalone 产物全树较慢] → marker 缓存命中即跳过（spec 要求）；替换只扫文本类扩展名。
- [`.next/dev/lock` 格式随 Next.js 版本变化] → 解析失败按「不满足」分支报错提示手动处理，不猜测；锁定前端为仓库内 `web/`（Next.js 16）降低漂移。
- [REPL 的 Ctrl-C 语义（中断 turn 不退进程）与 readline 库交互复杂] → turn 执行放独立 goroutine + context cancel，SIGINT 在 REPL 态只触发 cancel；以 PTY 集成测试覆盖该 Scenario。
- [CLI 输出「命令级对比」验收主观性] → 以 M5 矩阵为准：对齐行为与信息要素而非逐字节输出；`run --format json` 输出做 schema 断言。
