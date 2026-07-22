## Why

deeptutor-go 需要交付命令行界面与一键启动器：`deeptutor` CLI 覆盖单轮 capability 执行（`run`）、chat REPL、KB/memory 管理、`serve` 与 `start`；launcher 编排 Go 主服务 + Python Sidecar + 前端三个进程（端口发现与占用处理、健康检查、生命周期与退出清理），并约定单机分发形态（Go 静态二进制 + Sidecar venv、Docker 单容器）。行为规格见 `docs/golang-req/openspec/specs/cli-launcher/spec.md`；按 `docs/golang-req/openspec/ROADMAP.md`，本模块属于 Wave 4、里程碑 M5（交付面收口），是全部里程碑中最后合入的 change，其验收即 `docs/golang-req/acceptance.md` 第 4 节 M5 矩阵（CLI 命令级对比、launcher e2e、打包安装验证、全量回归、切换演练）。

## What Changes

- 新增 `deeptutor` 单一入口（spf13/cobra）：首版命令集 `run`、`chat`、`kb`、`memory`、`serve`、`start`、`partner`、`config`、`session`；`skill`/`plugin`/`notebook`/`provider`/`book` 随对应模块里程碑后置，框架预留子命令注册机制。
- 新增 `run` 单轮执行：进程内 SDK 等价路径执行 turn，选项与基线一致（`--session`/`-t`/`--kb`/`--notebook-ref`/`--history-ref`/`-l`/`--config`/`--config-json`/`-f rich|json`），config 值 JSON → bool/null → 字符串三级解析。
- 新增 chat REPL：启动选项与 `run` 同构、会话 preferences 恢复、Ctrl-C 只中断当前 turn、续行输入、全套斜杠命令（含 `/regenerate`/`/retry`）。
- 新增 `kb`（list/info/set-default/create/add/delete/search，入库经 Sidecar）与 `memory`（show/clear）子命令组。
- 新增 `serve`：仅启动 Go API 服务，端口缺省读 settings，WS 单帧上限随 chat 附件策略，访问日志仅记非 200。
- 新增 `start` 三进程编排（**D-006**）：runtime home 解析、统一环境注入、进程组隔离、前缀日志转印、HTTP 健康检查（主服务 60s / 前端 120s）、监督循环（任一进程意外退出整体关闭 exit 1）、Sidecar 缺失降级警告不阻塞。
- 新增端口发现与占用处理：TTY 交互（换端口持久化回 system.json / 终止占用进程）、非 TTY 直接失败。
- 新增前端运行时解析：打包 standalone 产物复制到可写缓存并替换 `__NEXT_PUBLIC_API_BASE_PLACEHOLDER__`/`__NEXT_PUBLIC_AUTH_ENABLED_PLACEHOLDER__`（marker 判缓存）、源码模式 `npm run dev` 与 `.next/dev/lock` 健康 dev server 复用。
- 新增退出清理与信号处理：SIGINT/SIGTERM/SIGHUP 幂等清理、前端 → Sidecar → 主服务逆序整树终止（8s 宽限后 KILL）、atexit 兜底。
- 新增分发形态（**D-006**）：Go 静态二进制 + Sidecar venv + 前端产物；Docker 单容器等价编排；基线 pip 三包形态不复刻。`deeptutor init` 向导后置（**D-007**），首版由 `start` 自动初始化覆盖必要子集。

## Capabilities

### New Capabilities

- `cli-launcher`: CLI 与启动器全部行为——命令面总览、run 单轮执行、chat REPL、kb/memory 子命令、serve、start 三进程编排（D-006）、端口发现与占用处理、前端运行时解析与复用、退出清理与信号处理、打包与分发形态（D-006）、init 向导后置（D-007）。

### Modified Capabilities

（无——本 change 不修改既有 spec 的 Requirement。）

## Impact

- 依赖的其他 change（按 ROADMAP 依赖图与 spec 声明，合入前需已验收）：
  - `impl-turn-runtime`：`run`/`chat` 进程内执行 turn 的运行时（turn → cli 边）。
  - `impl-partners`：`partner` 子命令与服务启动时 auto-start partners（part → cli 边）。
  - `impl-sidecar-contract`：`kb` 入库经 Sidecar、launcher 拉起并健康检查 Sidecar 进程。
  - 间接/关联：`impl-foundation-config`（runtime settings/home 解析、端口持久化）、`impl-capability-chat-solve`（`run chat` 等 capability 执行）、`impl-knowledge`（kb 子命令消费 KB 管理器）、`impl-memory`（memory 子命令）。
- 新增 Go 包：`cmd/deeptutor/`（cobra 命令树）、`internal/runtime/launcher/`（进程编排、端口、前端运行时、信号清理）。
- 新增外部依赖：`spf13/cobra`（CLI）；终端渲染选用轻量库（design.md 决策）；进程管理仅标准库 `os/exec` + `os/signal`。
- 交付面：单机分发（二进制 + Sidecar venv + 前端 standalone）与 Docker 单容器；有意差异 **D-006**（三进程编排与分发形态变更）、**D-007**（`init` 后置）。
- 前端影响：无——前端零改动，由 launcher 注入占位符替换与环境变量。
