# DeepTutor Go 重写验收文档

> 适用范围：`deeptutor-go`（Go 主服务）+ `deeptutor-sidecar`（Python Sidecar：RAG / 文档解析 / Manim / code_execution）整体替换现有 Python 后端。
> 验收基线：DeepTutor `main` 分支 v1.5.2；前端 `web/`（Next.js 16）保持零改动复用。
> 配套文档：`openspec/ROADMAP.md`（模块划分与实施顺序）、`openspec/specs/<module>/spec.md`（各模块行为规格，Requirement/Scenario 即功能验收用例来源）。

---

## 1. 验收总则

1. **最高优先级判定标准**：现有 `web/` 前端在不修改任何代码的前提下，指向 Go 后端（`:8001`）后全部页面功能可用（Chat、Co-Writer、Book、Knowledge Center、Learning Space、Memory、Settings、多用户）。
2. **验收单位**：以 `openspec/specs/` 下的模块 spec 为验收单位；每个 spec 内的 `Requirement + Scenario` 是该模块的功能验收用例清单，实现完成的定义是「该 spec 全部 Scenario 通过 + 对应协议对等测试通过」。
3. **验收分五类**：
   - A. 协议对等（REST / WebSocket 逐字段对齐）
   - B. 数据兼容（SQLite schema / `data/` 目录布局，老数据免迁移）
   - C. 功能行为（spec Scenario 驱动）
   - D. 非功能（性能 / 稳定性 / 启动 / 资源）
   - E. 切换与回滚演练
4. **不在验收范围**：`geogebra_analysis`（注意：基线 v1.5.2 中实际已注册且用户可切换，与 AGENTS.md 描述不符；Go 版决定不迁移其执行管线，但 `/api/v1/tools` 仍须返回该条目并标记 `coming_soon=true` 供设置页占位——已作为有意差异登记）、matrix-e2e channel、PocketBase session store（Go 版首期只支持 SQLite）、`weixin/mochat` 等重定制 channel（后置，见 ROADMAP Wave 4 说明）。范围变化需在本文档更新后生效。

---

## 2. 验收基线与环境

| 项 | 约定 |
| --- | --- |
| 行为基线 | 现有 Python 后端（v1.5.2）在同一份 `data/` 上的实际行为 |
| 协议基线 | 从现有 FastAPI 导出的 OpenAPI JSON（golden spec）+ 录制的 WS 事件流 fixtures |
| 数据基线 | 一份预置的 `data/` 快照：含 sessions（含消息分支）、memory L1/L2/L3、KB、partners workspace、settings |
| 测试环境 | 单机 macOS/Linux；`deeptutor-go start` 一键拉起 Go 主服务 + Sidecar + 前端 |
| LLM 确定性 | 契约级测试一律使用 **mock/replay LLM provider**（录制的 completion/tool_call chunks 回放）；真实 LLM 只用于连通性 e2e |

---

## 3. 验收方法体系

### 3.1 A 类：协议对等

**REST：**

1. 从现有后端导出 OpenAPI JSON 作为 golden spec（约 284 条路由）。
2. Go 侧对每条已迁移路由做 contract test：同请求 → 响应 status / JSON 结构 / 字段名 / 字段类型逐项 diff。
3. 允许忽略的差异（白名单，需显式登记）：响应头顺序、时间戳精度、错误 message 文案（错误 code/status 必须一致）。

**WebSocket：**

1. 在现有后端上运行脚本化场景，录制 `/api/v1/ws` 全消息类型的事件流为 JSONL fixtures（含 seq、type、stage、metadata）。录制场景至少覆盖：
   - 普通 chat turn（含流式 content、thinking、tool_call/tool_result、sources、result、done）
   - `subscribe_turn(after_seq)` 断线重连回放
   - `cancel_turn` 中止
   - `regenerate`（消息分支：`parent_message_id`）
   - `ask_user` 暂停 → `submit_user_reply` 恢复
   - `check_active_turn` / `subscribe_session` / `ping-pong`
2. Go 侧以 mock LLM 回放同一场景，逐事件对比。忽略字段白名单：`timestamp`；`seq` 必须连续且单调递增；事件顺序必须一致。
3. legacy WS（`/api/v1/chat`、`/api/v1/book/ws`、`question/mimic`、`question/generate`、`question/judge`、`partners/{id}/ws`、`knowledge/{kb}/progress/ws`）按前端实际消费的消息子集做同样对比。

### 3.2 B 类：数据兼容

1. Go 服务直接打开基线 `data/` 快照，不执行任何 migration，验证：
   - `sessions.db`（或同名 SQLite 文件）可读写：`sessions / messages / turns / turn_events / notebook_entries / notebook_categories` 全表；
   - 历史会话在前端可完整展示（含消息分支树、turn 事件回放）；
   - `data/user/settings/*.json` 直接生效（`system.json`、`llm_configs.json`、`integrations.json` 等）；
   - `data/memory/`（trace jsonl + L2/L3 md）可读可续写，引用链不破坏；
   - `data/knowledge_bases/` 经 Sidecar 可检索；
   - `data/partners/<id>/workspace/` 可加载。
2. 反向兼容：Go 写入后的 `data/`，回切到 Python 后端仍可正常打开（回滚前提）。

### 3.3 C 类：功能行为

- 每个模块 spec 的 Scenario 需落为自动化测试（Go 单测/集成测试）或登记为手工验收项（明确操作步骤与预期）。
- 手工验收项占比目标 < 20%，且仅限 UI 交互类（如前端展示效果）。

### 3.4 D 类：非功能

| 指标 | 通过标准 | 测量方法 |
| --- | --- | --- |
| Go 主服务冷启动 | < 2s（不含 Sidecar/前端） | 启动到 `/api/v1/system/health` 就绪 |
| 首 token 延迟 | 不劣于 Python 基线（同 provider 同网络，取 P50/P95） | mock provider 固定延迟对比运行时开销 |
| WS 回放吞吐 | `subscribe_turn` 回放 1 万事件 < 1s | fixtures 压测 |
| 内存 | 空闲 RSS < Python 基线的 50% | 同场景对比 |
| 稳定性 | 连续 24h 循环执行录制场景无泄漏、无 goroutine 堆积 | pprof + metrics |
| Sidecar 降级 | Sidecar 不可用时，chat 主链路可用，KB/解析/动画类请求返回明确错误 | kill sidecar 后跑场景 |

### 3.5 E 类：切换与回滚

- 切换 = 停 Python 后端 → 启 Go 后端（同端口 8001）→ 前端无需任何变更。
- 回滚 = 反向操作；依赖 3.2 的反向兼容验收。
- 演练要求：在含真实使用痕迹的 `data/` 上完成一次「切换 → 使用 → 回滚 → 使用」全程无损演练。

---

## 4. 里程碑验收矩阵

> 模块与里程碑的对应关系见 `openspec/ROADMAP.md`。以下为每个里程碑的退出条件（全部满足才进入下一里程碑）。

### M0 契约冻结与骨架

| 验收项 | 方法 | 通过标准 |
| --- | --- | --- |
| OpenAPI golden spec 导出 | 脚本从现有后端导出 | 覆盖全部 32 个 router；纳入版本控制 |
| WS fixtures 录制器/回放器 | 工具 + 首批 fixtures | 3.1 列出的 7 类场景全部有 fixture |
| Go 骨架 | 单测 | config 读取现有 settings JSON；SQLite store 打开基线库 CRUD 通过 |
| Sidecar 骨架 | contract test | `RAGSearch / RAGIndex / ParseDocument / RenderManim / ExecCode` typed API 可调用，schema 校验拒绝未知字段 |
| mock LLM provider | 单测 | 可回放录制的 completion / tool_call chunk 序列 |

### M1 chat 主链路

| 验收项 | 方法 | 通过标准 |
| --- | --- | --- |
| 统一 WS 全消息类型 | fixtures 回放 | 11 种消息类型行为对等；seq 连续；resume/cancel/regenerate/ask_user 场景全过 |
| Agent Loop | mock LLM 集成测试 | tool_call 循环、轮次预算强制收敛、context-gated mounting 与基线一致 |
| LLM provider | 真实 e2e + mock 契约 | OpenAI-compat + Anthropic 流式与 function calling 可用 |
| 第一批 tools | 单测/集成 | `brainstorm/reason/web_search/paper_search/web_fetch/github/read_source/ask_user` + Sidecar 的 `rag/exec/code_execution` |
| 前端 Chat 页 | 手工 + Playwright | 发消息、流式、附件（解析走 Sidecar）、中止、重新生成、ask_user 交互全可用 |
| sessions REST | contract test | 会话列表/历史/分支与基线一致 |

### M2 Memory + Settings + Knowledge 接入面

| 验收项 | 方法 | 通过标准 |
| --- | --- | --- |
| Memory | 数据兼容 + Scenario | 基线 memory 数据续写；consolidator（update/audit/dedup/merge）产出结构合法且带来源引用 |
| Settings/System/Personas/Tools API | contract test | 前端 Settings 全页面可用 |
| Knowledge 接入面 | contract test + e2e | KB CRUD 在 Go；索引/检索经 Sidecar；progress WS 推送与基线一致 |
| Notebook / Question Notebook | contract test | 前端对应页面可用 |
| Skills | Scenario | SKILL.md 解析、`read_skill/load_tools` 行为一致 |
| Cron + EventBus | Scenario | cron 任务持久化、触发执行 turn |

### M3 其余 Capabilities

| 验收项 | 方法 | 通过标准 |
| --- | --- | --- |
| deep_solve / mastery_path / deep_question / deep_research / visualize | fixtures + 手工 | 各 capability 阶段事件（stage_start/stage_end/progress）与基线一致；前端对应页面可用 |
| math_animator | e2e | 编排在 Go，渲染经 Sidecar Manim 产出视频文件 |
| Book / Co-Writer | contract test + 手工 | book WS 事件流对等；前端 Book/Co-Writer 页面可用 |
| learning 引擎 | 单测 | grading/mastery/scheduler 对同输入产出与 Python 版一致（golden case 对比） |

### M4 Partners + Subagents + 多用户

| 验收项 | 方法 | 通过标准 |
| --- | --- | --- |
| Partner runtime | Scenario | workspace/SOUL.md/独立 memory/独立工具策略加载正确 |
| Channels（feishu/telegram/slack/discord） | 真实账号 e2e | 收发消息、流式回复、附件 |
| consult_subagent | e2e | Claude Code / Codex CLI 子进程流式输出进入 tool result |
| 多用户 | contract test | 登录、workspace 隔离、管理员资源分配与基线一致 |

### M5 CLI / Launcher 与整体切换

| 验收项 | 方法 | 通过标准 |
| --- | --- | --- |
| CLI | 命令级对比 | `run/chat/kb/memory/serve/start` 输出与行为对齐（REPL 含 /regenerate） |
| Launcher | e2e | `start` 一键拉起 Go + Sidecar + 前端；端口发现与占用处理 |
| 打包 | 安装验证 | 单机分发（二进制 + sidecar venv）与 Docker 单容器均可启动 |
| 全量回归 | 全部 fixtures + Playwright | A/B/C/D 类全部通过 |
| 切换演练 | 3.5 流程 | 「切换→使用→回滚→使用」无损 |

---

## 5. 验收产物清单

| 产物 | 位置（建议） | 说明 |
| --- | --- | --- |
| OpenAPI golden spec | `deeptutor-go/contracts/openapi.json` | M0 产出，冻结后变更需评审 |
| WS fixtures | `deeptutor-go/contracts/fixtures/*.jsonl` | 录制场景 + 回放工具 |
| Sidecar API contract | `deeptutor-go/contracts/sidecar/` | OpenAPI 或 protobuf |
| 数据基线快照 | `deeptutor-go/testdata/data-snapshot/` | 覆盖 3.2 全部数据域 |
| 验收报告 | 每里程碑一份 | 按第 4 节矩阵逐项勾选，含失败项与豁免记录 |

## 6. 豁免与差异登记

任何与 Python 基线的**有意**行为差异（bug 修复、废弃行为不迁移等）必须登记：

| 字段 | 说明 |
| --- | --- |
| 差异 ID | 递增编号 |
| 涉及模块 spec | 如 `turn-runtime` |
| 基线行为 / 新行为 | 描述 |
| 理由 | 为什么不对齐 |
| 前端影响 | 有/无，有则说明验证方式 |

未登记的差异一律按缺陷处理。

### 已登记差异

| ID | 模块 spec | 基线行为 | 新行为 | 理由 | 前端影响 |
| --- | --- | --- | --- | --- | --- |
| D-001 | session-store | `append_turn_event` 对 `(turn_id, seq)` 冲突 `INSERT OR REPLACE` 静默覆盖 | 冲突报错；正常路径由 runtime 唯一 seq 分配保证不冲突 | 覆盖会掩盖生产者分叉 | 无 |
| D-002 | turn-runtime | `ask_user` 暂停期间 turn 状态仍为 `running` | 拆分出 `waiting_input` 状态 | 超时/告警/并发管理需要区分 | 无（WS 事件面不变；`check_active_turn` 响应中该状态映射回基线语义） |
| D-003 | turn-runtime | 事件终态批量 flush，进程崩溃丢整轮事件 | append-before-publish（或等价 WAL），已提交事件崩溃不丢 | 可靠性 | 无 |
| D-004 | tools-builtin | `geogebra_analysis` 已注册且用户可切换 | 不迁移执行管线；`/api/v1/tools` 返回该条目并标 `coming_soon=true` | 视觉重建管线强 Python 绑定、价值待定 | 设置页显示为 coming soon；需手工验证 |
| D-005 | multi-user-auth | 用户上下文经 `ContextVar` 隐式传播（跨事件循环失配时静默路由到 admin workspace，issue #481 根因） | 显式 context 传递；上下文缺失必须报错，SHALL NOT 静默回落 admin | 行为对等但实现更安全 | 无 |
| D-006 | cli-launcher | `start` 编排后端+前端双进程；pip 三包分发 | `start` 编排 Go 主服务 + Sidecar + 前端三进程（Sidecar 缺失降级警告不阻塞）；分发为 Go 二进制 + Sidecar venv + Docker 单容器 | 架构形态变更 | 无 |
| D-007 | cli-launcher | `deeptutor init` 交互式初始化向导 | 首期后置不实现 | 非核心路径，降低 M5 范围 | 无（CLI 提示不可用） |
| D-008 | capability-question | `/api/v1/question/generate` WS 走旧版 `AgentCoordinator` | 复用 QuestionPipeline，但对客户端的消息序列（`task_id → status → 进度 → batch_summary → complete`）保持不变 | 消除双实现 | 无（WS 消息面不变，以 fixtures 验证） |
