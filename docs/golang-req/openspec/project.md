# Project Context

## Purpose

`deeptutor-go`：用 Go 重写 DeepTutor 除 RAG 等 Python 生态绑定域外的全部后端，配套 `deeptutor-sidecar`（Python）承载 RAG / 文档解析 / Manim 渲染 / code_execution 沙箱。目标：

- 现有 `web/` 前端（Next.js 16）零改动复用（REST/WS 协议逐字段对等）；
- 沿用现有 SQLite schema 与 `data/` 目录布局，老数据免迁移；
- 保持 local-first：单机一条命令启动（Go 主服务 + Sidecar + 前端）。

行为基线：现有 Python 实现（`deeptutor/`，v1.5.2）。各 spec 中引用的 `deeptutor/...` 路径均指该基线仓库中的参考实现。

## Tech Stack

已确认的技术选型约束（实施 change 的 design.md 必须遵循）：

- Go ≥ 1.23，单 module。
- **HTTP/REST**：`gin-gonic/gin`（route group 组织 `/api/v1/*`）；**WebSocket**：`gorilla/websocket` 在 gin handler 内 upgrade。
- **LLM/Agent 框架**：`cloudwego/eino`（核心抽象：ChatModel / ToolsNode / StreamReader）+ `eino-ext`（openai / claude 等 model 组件）。LLM provider 适配层实现为 eino ChatModel；Agent Loop 基于 eino 的 tool-calling 循环（ReAct agent 或自定义 graph）；mock/replay provider 实现为自定义 ChatModel。eino 流式输出映射到 StreamEvent 协议。
- **SQLite**：`modernc.org/sqlite`（免 CGO）+ `database/sql`，沿用基线 schema。
- **配置**：`spf13/viper` 读取 `data/user/settings/*.json`（每文件一个 viper 实例、WatchConfig 热更新），env override 优先级与基线一致（含 self-echo 抑制语义，见 `foundation-config` spec）。
- **CLI**：`spf13/cobra`。
- Python Sidecar：FastAPI 薄壳，复用现有 `services/rag`、`services/parsing`、`services/sandbox`、`agents/math_animator/renderer` 代码，暴露 typed API（OpenAPI schema 冻结于 `contracts/sidecar/`）。
- 前端不在本项目范围；`web/proxy.ts` 默认转发到 `http://localhost:8001`，Go 服务监听同端口。

## Project Conventions

### 目录结构

```
deeptutor-go/
├── cmd/deeptutor/          # CLI 入口
├── internal/
│   ├── api/                # REST routers + WS endpoints
│   ├── runtime/            # orchestrator、turn runtime、registry、launcher
│   ├── core/               # StreamEvent、StreamBus、UnifiedContext、protocol 定义
│   ├── capabilities/       # chat/solve/question/research/visualize/mastery/math_animator
│   ├── tools/              # builtin tools
│   ├── llm/                # provider 适配 + agentic client
│   ├── memory/             # L1/L2/L3 + consolidator
│   ├── partners/           # channel adapters + partner runtime
│   ├── session/            # SQLite store
│   ├── sidecar/            # Sidecar 客户端（typed contract）
│   └── config/             # runtime settings
├── prompts/                # 各 capability 的 {en,zh} prompts（从基线仓库拷贝）
└── contracts/              # openapi.json、WS fixtures、sidecar contract
```

### Spec 约定

- 每个模块一份 `specs/<module>/spec.md`；正文中文，代码标识/协议字段/专有名词保留英文；Requirement 使用 SHALL 措辞；每个 Requirement 至少一个 `#### Scenario:`。
- Spec 描述**行为**而非实现；字段级对等以 `contracts/` 下的 golden spec 与 fixtures 为准，spec 不重复枚举全部字段。
- 实施某模块时，从对应 spec 建立 openspec change（`changes/<change-id>/`：`proposal.md` + `tasks.md` + delta specs），完成验收后 archive。

### 测试约定

- 契约级测试使用 mock/replay LLM provider（回放录制的 chunk 序列），保证确定性；真实 LLM 仅做连通性 e2e。
- 验收标准见 `docs/golang-req/acceptance.md`；模块完成 = spec 全部 Scenario 通过 + 协议对等测试通过。

## Domain Context

DeepTutor 是 agent-native 学习助手：两级插件模型（单发 Tool + 接管整轮的 Capability），统一 Turn Runtime（`session_id + turn_id + seq`，支持断线 resume、cancel、regenerate、ask_user 暂停/恢复），统一 StreamEvent 协议，文件型 L1/L2/L3 可审计记忆，IM Partners，多用户工作区隔离。

## Important Constraints

- 禁止修改 `web/` 前端与现有 Python 仓库（基线只读）。
- 禁止引入 PostgreSQL/消息队列等分布式基础设施；单机 SQLite + 文件系统。
- `data/user/settings/*.json` 是运行时配置事实源；忽略项目根 `.env`。
- 与基线的任何有意行为差异必须登记（acceptance.md 第 6 节）。
