## 1. CLI 骨架与渲染

- [ ] 1.1 搭建 `cmd/deeptutor/` cobra 命令树：root（无参数输出帮助、非错误退出）、`RegisterCommand` 后置子命令注册机制（skill/plugin/notebook/provider/book 预留）
- [ ] 1.2 实现 `render.go`：rich 风格终端渲染（lipgloss + glamour：面板/表格/流式文本）与 `--format json` 机器可读输出
- [ ] 1.3 实现 `config` 与 `session` 子命令组（对应基线 `config_cmd.py`、`session_cmd.py`）

## 2. run 与 chat REPL

- [ ] 2.1 实现进程内 SDK 装配路径（orchestrator + turn runtime，与 API 服务共用装配函数，无需起服务）
- [ ] 2.2 实现 `run.go`：capability 位置参数 + 全部选项（`--session`/`-t`/`--kb`/`--notebook-ref`/`--history-ref`/`-l`/`--config`/`--config-json`/`-f`）、config 值三级解析（JSON → true/false/null → 字符串）、非法 `--config-json` 参数错误退出
- [ ] 2.3 实现 `chat.go` REPL 骨架：readline 循环、行尾反斜杠续行、`--session` 不存在报错退出 / 存在时恢复 preferences（capability/tools/KB/language/引用）、turn 完成写回 session id
- [ ] 2.4 实现 Ctrl-C 语义：turn 在独立 goroutine + context cancel，中断当前 turn 不退出 REPL
- [ ] 2.5 实现全套斜杠命令：`/quit`、`/session`、`/status`、`/new`（`/clear`）、`/regenerate`（`/retry`，无活动会话提示）、`/tool on|off`、`/cap`、`/kb`、`/history`、`/notebook`、`/show last|<n>`、`/refs`、`/config show|set|clear`

## 3. kb / memory / partner 子命令

- [ ] 3.1 实现 `kb.go`：list（默认标记）/info/set-default/create（`--doc` 经 Sidecar 解析索引 + 进度展示）/add/delete（确认或 `--yes`）/search（命中片段渲染）；Sidecar 不可用给出可诊断错误
- [ ] 3.2 实现 `memory.go`：show（默认 L3、选项看 L2 surface）/clear（全部/仅 trace/单 L1 surface，需确认或 `--yes`）
- [ ] 3.3 实现 `partner.go` 子命令组（list 等，对应基线 `deeptutor_cli/partner.py`）

## 4. serve 与 launcher 核心

- [ ] 4.1 实现 `serve.go`：`--host`（默认 0.0.0.0）/`--port`（缺省读 settings backend_port）、WS 单帧上限按 chat 附件总量上限配置、访问日志仅记非 200
- [ ] 4.2 实现 `launcher/home.go`：runtime home 解析（`--home` > `DEEPTUTOR_HOME` > 默认）+ 自动初始化（建数据目录 + 默认 settings，D-007 必要子集）
- [ ] 4.3 实现 `launcher/process.go` `ManagedProcess`：进程组启动（POSIX Setsid / Windows CREATE_NEW_PROCESS_GROUP 接口）、`backend`/`sidecar`/`frontend` 前缀日志泵、`Terminate`（killpg TERM → 8s → KILL）、`Exited()`
- [ ] 4.4 实现 `launcher/ports.go`：TCP 探测、占用进程列举（lsof，含 pid 与命令行）、TTY 交互（[1] 换端口 +1 向上找空闲且两口互斥、持久化回 system.json；[2] 终止占用 TERM→5s→KILL→3s 再验）、非 TTY 直接失败、`serve --port` 跳过交互
- [ ] 4.5 实现 `launcher/frontend.go`：打包产物复制到 `data/user/runtime/web/` + 占位符替换（`__NEXT_PUBLIC_API_BASE_PLACEHOLDER__`/`__NEXT_PUBLIC_AUTH_ENABLED_PLACEHOLDER__`）+ marker 缓存判定 + Node 20+ 检测；源码模式 `npm run dev` + `.next/dev/lock` 健康 dev server 复用/终止重启/报错三分支；两者皆不可得的报错提示
- [ ] 4.6 实现 `launcher/sidecar.go`：venv 发现顺序（env 显式 → 安装布局 → 源码 venv）、拉起与健康检查、缺失降级警告不阻塞（D-006）
- [ ] 4.7 实现 `launcher/launcher.go` + `health.go` + `signals.go`：统一环境注入、按序拉起三进程、健康检查（backend 60s / frontend 120s、等待期早退立即报错）、banner、监督循环（意外退出 → 逆序清理 → exit 1）、信号幂等清理（SIGINT/SIGTERM/SIGHUP、前端→Sidecar→主服务、main 顶层 defer 兜底）
- [ ] 4.8 实现 `start.go`：`--home` 接线、Go 主服务以 re-exec `deeptutor serve` 作为受管子进程

## 5. 分发形态（D-006）

- [ ] 5.1 实现单机分发构建：`CGO_ENABLED=0` 静态二进制、Sidecar venv 安装脚本、前端 standalone 产物打包、`start` 自动发现三者；Sidecar 缺失时依赖功能返回安装指引
- [ ] 5.2 实现 Docker 单容器：单镜像含三组件、入口进程执行 `start` 等价编排（非 TTY 端口语义）、`data/` 单卷挂载

## 6. 测试与验收（spec Scenario 落测试 + M5 矩阵）

- [ ] 6.1 CLI Scenario 测试：无参数帮助退出码 0、`run deep_solve -t rag --kb my-kb`（mock LLM）、非法 `--config-json` 参数错误、`serve` 端口缺省取 settings
- [ ] 6.2 REPL Scenario 测试（PTY）：`/retry` 同 session 重跑、Ctrl-C 只中断 turn 不退 REPL
- [ ] 6.3 kb/memory Scenario 测试：`kb create --doc` 经 Sidecar 入库后 list 可见（Sidecar mock + 真实各一）、`memory show` 空时提示不报错
- [ ] 6.4 launcher e2e：一键三进程启动（健康检查 + banner + 前缀日志）、前端进程 kill 后整体退出码 1、Ctrl+C 全链路清理无子进程残留（pgrep 断言）
- [ ] 6.5 端口 Scenario 测试：TTY 交互换端口并持久化 system.json、非 TTY（管道 stdin）冲突直接失败
- [ ] 6.6 前端运行时 Scenario 测试：打包占位符替换一次生效（二次启动命中 marker 不复制）、复用健康源码 dev server
- [ ] 6.7 分发验收：单机安装（二进制 + venv + 前端）与 Docker 单容器均可启动；无 Sidecar 时纯 chat 可用、RAG 入库返回安装指引（D-006 Scenario）
- [ ] 6.8 全新机器起步验收：无 init 直接 `start`，目录与默认 settings 自动创建（D-007 Scenario）
- [ ] 6.9 M5 验收矩阵（acceptance.md 第 4 节）：CLI 命令级对比（`run/chat/kb/memory/serve/start`，REPL 含 `/regenerate`）、launcher e2e、打包安装验证、全量回归（A/B/C/D 类：全部 fixtures + Playwright）
- [ ] 6.10 切换演练（acceptance.md 3.5）：在含真实使用痕迹的 `data/` 上完成「停 Python 起 Go（同端口 8001）→ 使用 → 回滚 Python → 使用」全程无损，出具验收报告
