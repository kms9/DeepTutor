# CLI & Launcher Specification

## Purpose
本模块定义 deeptutor-go 的命令行界面与一键启动器：`deeptutor` CLI 覆盖单轮 capability 执行（`run`）、交互式 chat REPL、KB 与 memory 管理、API 服务（`serve`）以及 `start` 一键拉起完整应用；launcher 负责 Go 主服务 + Python Sidecar + 前端三个进程的编排——端口发现与占用处理、健康检查、生命周期与退出清理。同时约定单机分发形态：Go 静态二进制 + Sidecar venv，以及 Docker 单容器。命令面与基线 Python CLI 行为对齐，进程编排在基线双进程（backend + frontend）之上增加 Sidecar（有意差异）。

- 参考实现（基线）：`deeptutor_cli/`（`main.py`、`chat.py`、`kb.py`、`memory.py`、`common.py`、`init_cmd.py`）、`deeptutor/runtime/launcher.py`（933 行）、`deeptutor/runtime/home.py`
- 依赖 spec / 里程碑：依赖：foundation-config、turn-runtime、capability-chat-solve、knowledge-kb-manager、sidecar-bridge；里程碑：M5（交付面收口）

## Requirements

### Requirement: CLI 命令面总览
系统 SHALL 提供 `deeptutor` 单一入口，无参数时输出帮助。首版命令集 SHALL 含：`run`（单轮 capability 执行）、`chat`（REPL）、`kb`、`memory`、`serve`、`start`、`partner`、`config`、`session`。基线还有 `skill`（别名 `skills`）、`plugin`、`notebook`、`provider`、`book` 子命令组，SHALL 随对应模块 spec 的里程碑排期，CLI 框架 SHALL 预留子命令注册机制使其可独立接入。所有子命令的输出默认 rich 风格终端渲染，`run` 支持 `--format json` 输出机器可读结果。

#### Scenario: 无参数输出帮助
- **WHEN** 用户执行 `deeptutor`
- **THEN** 打印命令清单与说明后以非错误状态退出

### Requirement: deeptutor run 单轮执行
系统 SHALL 实现 `deeptutor run <capability> "<message>"`：直接在进程内以 SDK 等价路径执行一个 turn（无需先起服务），capability 可为 `chat`、`deep_solve`、`deep_question`、`deep_research`、`visualize`、`math_animator`、`mastery_path` 等任意已注册 capability。选项 SHALL 与基线一致：`--session`（续接既有 session id）、`-t/--tool`（可重复，启用工具）、`--kb`（可重复）、`--notebook-ref`（`<notebook_id>[:<record_id>,...]` 格式）、`--history-ref`、`-l/--language`（默认 en）、`--config key=value`（可重复）、`--config-json`（JSON 对象，与 `--config` 合并、`--config` 优先）、`-f/--format rich|json`。config 值解析 SHALL 先按 JSON 解析、失败时识别 `true`/`false`/`null` 再落回字符串。

#### Scenario: 携带工具与 KB 运行 deep_solve
- **WHEN** 执行 `deeptutor run deep_solve "Solve x^2=4" -t rag --kb my-kb`
- **THEN** 一个 turn 以 deep_solve capability、rag 工具与 my-kb 执行，终端渲染各阶段进度与最终结果，进程退出

#### Scenario: 非法 config JSON 报参数错误
- **WHEN** `--config-json '{bad'` 传入
- **THEN** 以参数错误退出并给出解析失败信息，不执行 turn

### Requirement: chat REPL
系统 SHALL 实现 `deeptutor chat` 交互 REPL：启动选项与 `run` 同构（`--session`、`-t/--tool`、`-c/--capability`、`--kb`、`--notebook-ref`、`--history-ref`、`-l/--language`、`--config`、`--config-json`）；`--session` 指向不存在的会话时报错退出，存在时 SHALL 从会话 preferences 恢复 capability/tools/KB/language/引用。REPL SHALL 支持：普通输入发起 turn（Ctrl-C 中断进行中的 turn 而不退出 REPL）；行尾反斜杠续行的多行输入；斜杠命令——`/quit`、`/session`、`/status`、`/new`（别名 `/clear`，重置 session id 开新上下文）、`/regenerate`（别名 `/retry`，重跑最后一条用户消息，无活动会话时提示先发消息）、`/tool on|off <name>`、`/cap <name>`、`/kb <name>|none`、`/history add <id>|clear`、`/notebook add <ref>|clear`、`/show last|<n>`（展开捕获的 tool result / thinking）、`/refs`、`/config show|set|clear`。每个 turn 完成后 SHALL 把返回的 session id 写回状态使后续消息续接同一会话。

#### Scenario: /regenerate 重跑上一条消息
- **WHEN** REPL 内已完成一个 turn，用户输入 `/retry`
- **THEN** 系统在同一 session 上重新生成最后一条用户消息的回答并渲染，session id 保持连续

#### Scenario: Ctrl-C 只中断当前 turn
- **WHEN** turn 执行中用户按 Ctrl-C
- **THEN** 当前 turn 被取消，REPL 回到输入提示符而非退出进程

### Requirement: kb 子命令
系统 SHALL 实现 `deeptutor kb` 子命令组：`list`（列出 KB 及默认标记）、`info <name>`（元数据与文档清单）、`set-default <name>`、`create <name> [--doc <file>...]`（创建并可选立即入库文档，经 Sidecar 解析/索引）、`add <name> --doc <file>...`（向既有 KB 追加文档）、`delete <name>`（须确认或 `--yes`）、`search <name> "<query>"`（检索并渲染命中片段）。文档入库 SHALL 复用与 API 相同的解析/索引管线（经 Python Sidecar），CLI SHALL 展示入库进度并在 Sidecar 不可用时给出可诊断错误。

#### Scenario: 创建 KB 并入库 PDF
- **WHEN** 执行 `deeptutor kb create my-kb --doc textbook.pdf`
- **THEN** KB 目录建立、文档经 Sidecar 解析并索引，命令结束时 `kb list` 可见 my-kb

### Requirement: memory 子命令
系统 SHALL 实现 `deeptutor memory` 子命令组：`show`（默认显示 L3 全局 memory，支持选项查看某 L2 surface 汇总）、`clear`（支持全部清空、仅清 trace、或指定单个 L1 surface），行为与基线分层 memory 模型对齐。清空操作 SHALL 需要确认或显式 `--yes`。

#### Scenario: 查看全局 memory
- **WHEN** 执行 `deeptutor memory show`
- **THEN** 渲染 L3 全局 memory 内容；不存在时提示为空而非报错

### Requirement: serve 命令
系统 SHALL 实现 `deeptutor serve`：仅启动 Go API 服务（不带前端），选项 `--host`（默认 `0.0.0.0`）、`--port`（缺省时读 runtime settings 的 backend_port）、开发用 `--reload` 可标后置。服务 SHALL 按配置的 chat 附件总量上限设置 WebSocket 单帧上限（基线经 uvicorn `ws_max_size`；base64 附件在一个 JSON 帧内，默认 16MB 帧上限会切断合法上传）。访问日志 SHALL 只记录非 200 响应，避免例行轮询刷屏。

#### Scenario: 端口缺省取自 settings
- **WHEN** 执行 `deeptutor serve` 不带 `--port`
- **THEN** 服务绑定 runtime settings 中的 backend_port

### Requirement: start 一键启动与进程编排（有意差异）
系统 SHALL 实现 `deeptutor start [--home <dir>]`：解析 runtime home（`--home` > `DEEPTUTOR_HOME` 环境变量 > 默认），把它写入子进程环境并初始化数据目录与 settings 文件，然后按序拉起三个受管进程——Go 主服务（基线为 uvicorn backend）、Python Sidecar（RAG/文档解析/Manim/沙箱；基线无独立 Sidecar，此为有意差异）、前端（Node）。每个进程 SHALL：注入统一环境（`DEEPTUTOR_HOME`、`BACKEND_PORT`、`FRONTEND_PORT`、`PORT`、`NEXT_PUBLIC_API_BASE`、`NEXT_PUBLIC_AUTH_ENABLED`、`DEEPTUTOR_API_BASE_URL`、`DEEPTUTOR_AUTH_ENABLED` 等）；在自己的进程组/会话中启动（POSIX `start_new_session`，Windows `CREATE_NEW_PROCESS_GROUP`）以便整树终止；stdout/stderr 以 `backend`/`sidecar`/`frontend` 前缀实时转印到 launcher 终端。就绪 SHALL 以 HTTP 健康检查判定：主服务 60s 超时、前端 120s 超时，等待期间子进程提前退出立即报错。全部就绪后打印 banner（backend/frontend URL、workspace、frontend runtime 类型）并进入监督循环：任一受管进程意外退出即触发整体关闭且以退出码 1 结束。Sidecar 不可用 SHALL 降级为警告并继续（依赖 Sidecar 的功能运行期报错），SHALL NOT 阻塞主服务与前端启动。

#### Scenario: 一键启动三进程
- **WHEN** 执行 `deeptutor start`
- **THEN** Go 主服务、Python Sidecar、前端依次启动并通过健康检查，终端打印各自 URL 与带前缀的日志流

#### Scenario: 任一进程崩溃触发整体退出
- **WHEN** 运行中前端进程意外退出
- **THEN** launcher 打印退出信息、按序终止其余受管进程并以退出码 1 结束

### Requirement: 端口发现与占用处理
系统 SHALL 从 runtime settings（system.json）读取 backend_port 与 frontend_port；启动前逐个探测占用（本地 TCP connect），冲突时 SHALL 尽力列出占用进程（POSIX 经 `lsof`，Windows 经 `netstat`+`tasklist`，含 pid 与命令行）。stdin 为 TTY 时 SHALL 交互处理：选项 [1] 换端口（默认建议自冲突端口 +1 起向上找第一个空闲口，两口互斥），新端口 SHALL 持久化回 system.json；选项 [2] 终止占用进程（SIGTERM，5s 内未释放则 KILL，再等 3s，仍占用则报告失败）后重试。非 TTY（Docker/CI）SHALL 直接以「端口被占用」错误退出。`serve --port` 显式指定时不做交互处理。

#### Scenario: 交互换端口并持久化
- **WHEN** backend_port 被占用且用户在提示中选择换端口并接受建议值
- **THEN** 启动使用新端口，system.json 中的 backend_port 更新，下次启动直接使用新值

#### Scenario: 非 TTY 遇冲突直接失败
- **WHEN** 在 Docker 中启动且端口被占用
- **THEN** 进程打印占用端口清单后以错误退出，不挂起等待输入

### Requirement: 前端运行时解析与复用
系统 SHALL 按优先级解析前端运行时：(1) 打包产物（Next.js standalone，需 Node.js 20+，缺 Node 即报错退出）——SHALL 先把打包目录复制到 workspace 内可写缓存（`data/user/runtime/web/`），把构建期占位符 `__NEXT_PUBLIC_API_BASE_PLACEHOLDER__`/`__NEXT_PUBLIC_AUTH_ENABLED_PLACEHOLDER__` 替换为实际值，并以 marker 文件（源路径、源 mtime、api_base、auth_enabled）判定缓存有效性避免重复复制；(2) 源码 checkout（`web/package.json` 存在）——以 `npm run dev -- --port <port>` 启动，缺 npm 报错。源码模式 SHALL 检测本 checkout 已在运行的 Next dev server（读 `.next/dev/lock` 的 pid/port）：健康则直接复用（跳过前端端口冲突检查），不健康且 lock 确指向 Next 进程则终止后重启，两者皆不满足时报错。两种运行时都不可得时 SHALL 报错提示安装完整包或使用含 `web/` 的源码。

#### Scenario: 打包前端占位符替换一次生效
- **WHEN** 首次以打包安装启动，随后不改配置再次启动
- **THEN** 首次复制并替换占位符，二次启动命中 marker 直接复用缓存，不重复复制

#### Scenario: 复用健康的源码 dev server
- **WHEN** 源码模式下 `.next` lock 指向的 dev server 健康存活
- **THEN** launcher 不再拉起新前端，改用已运行实例的 URL 并等待其健康检查通过

### Requirement: 退出清理与信号处理
系统 SHALL 注册 `SIGINT`/`SIGTERM`/`SIGHUP`（及 Windows `SIGBREAK`）处理器：收到信号置关闭标志、打印信号名并进入清理。清理 SHALL 幂等（多信号只执行一次）、按 前端 → Sidecar → 主服务 逆序终止：先向进程树发 SIGTERM（POSIX 按 pgid killpg，Windows `taskkill /T`），等待 8s 未退出再 KILL；并以 atexit 兜底保证 launcher 任何退出路径都不遗留孤儿进程。Ctrl+C SHALL 等价于 SIGINT 路径。

#### Scenario: Ctrl+C 全链路清理
- **WHEN** 三进程运行中用户按 Ctrl+C
- **THEN** launcher 打印停止信息、逆序 SIGTERM 各进程树并在超时后强杀，退出后无任何子进程残留

### Requirement: 打包与分发形态（有意差异）
系统 SHALL 支持两种单机分发：(1) 本机安装——Go 主服务为单一静态二进制（含 CLI），Python Sidecar 以独立 venv 分发（安装脚本创建 venv 并装 sidecar 包），前端 standalone 产物随包分发，`deeptutor start` 自动发现三者；Sidecar venv 缺失时 CLI 与主服务可运行，依赖 Sidecar 的功能给出安装指引。(2) Docker 单容器——一个镜像内含 Go 二进制、Sidecar venv 与前端产物，由容器入口进程做与 `start` 等价的编排，数据经单一 `data/` 卷挂载。基线的 pip 三包形态（`deeptutor`/`deeptutor-cli`/`deeptutor-web`）不复刻（有意差异：分发单位从 Python wheel 变为二进制 + venv + 镜像，命令面与数据布局不变）。

#### Scenario: 无 Sidecar 时核心功能可用
- **WHEN** 仅安装 Go 二进制与前端、未装 Sidecar venv，用户执行 `deeptutor start` 并使用纯 chat
- **THEN** 主服务与前端正常运行，chat 可用；触发 RAG 入库等 Sidecar 功能时返回带安装指引的明确错误

### Requirement: init 向导（后置）
`deeptutor init` 交互式初始化向导（引导选择 workspace、配置模型 API key、端口等）SHALL 标记为后置 requirement，首版由 `start` 的自动初始化（建目录 + 生成默认 settings 文件）覆盖其必要子集。

#### Scenario: 首版无 init 也能起步
- **WHEN** 全新机器上直接执行 `deeptutor start`
- **THEN** 数据目录与默认 settings 自动创建，应用可启动（模型未配置的功能在 UI 中提示配置）
