## Why

deeptutor-go 需要把 Partner（IM 伴侣）子系统迁移到 Go：每个 partner 是由 chat agent loop 驱动的常驻实例，拥有独立 workspace（SOUL.md、memory、skills、KB、notebook）与工具策略，来自 IM channel 或 Web 的每条消息都作为一个 chat turn 进入 `ChatOrchestrator` 的同一执行链。行为规格见 `docs/golang-req/openspec/specs/partners/spec.md`；按 `docs/golang-req/openspec/ROADMAP.md`，本模块属于 Wave 4（外围与交付）、里程碑 M4，是 M4 三模块中体量最大的一个（基线约 `deeptutor/services/partners/` + `deeptutor/partners/channels/` + `deeptutor/api/routers/partners.py`）。chat 主链路（M1–M3）已就绪后，partners 是前端 Partners 页面与 IM 场景可用的必要条件，也是 cli-launcher（M5）的前置依赖。

## What Changes

- 新增 partner manager：`data/partners/<id>/` 配置持久化（`config.yaml` 原子写、`auto_start` 生命周期意图语义、slug/CJK 稳定 id）、start/stop/destroy/auto-start/reload_channels 生命周期。
- 新增 partner runtime：inbound 消息 → `ChatOrchestrator` chat turn（合成用户 scope、SOUL persona 注入、三元工具策略、per-session 串行锁、backup LLM 重试）。
- 新增 StreamEvent → IM 投递映射：最终回复 / terminator 文本 / narration progress / tool hint / `_stream_delta` 流式与 `_streamed` 去重，及会话持久化（user + assistant 含事件 trace）。
- 新增 channel adapter 抽象与 outbound 分发管线：MessageBus 解耦、delta coalescing、指纹抑制、指数退避重试、channel 注册表（内置优先、缺依赖记录原因）。
- 新增首批四个 channel：feishu（开放平台 WS 长连接 + CardKit streaming card）、telegram（长轮询 + edit 就地流式）、slack（Socket Mode + 线程级 session）、discord（Gateway + REST v10 + PATCH 流式）；其余 channel 后置，`matrix-e2e` 明确不迁移（有意差异）。
- 新增 partner 斜杠命令集（`/help`、`/new`、`/sessions`、`/tool` 等，turn 前拦截）。
- 新增 `/api/v1/partners` REST 面（约 31 条，整组 admin 门控，秘密字段深度掩码）与 Web 聊天入口（HTTP / SSE / WebSocket），WS turn 以 LiveTurn 与 socket 解耦，支持断线 attach 重放续流。
- 新增 soul 模板库（`data/partners/_souls.yaml` seed 与旧 seed 原地升级）与资产供给（KB / skill / notebook 复制进 partner workspace）。

## Capabilities

### New Capabilities

- `partners`: Partner 子系统全部行为——配置持久化与数据布局兼容、生命周期、ChatOrchestrator 同一执行链、事件到 IM 映射、channel adapter 抽象与 outbound 管线、首批四 channel、后置 channel 与 matrix-e2e 不迁移、斜杠命令、REST 面、Web 聊天与 LiveTurn、soul 模板库与资产供给。

### Modified Capabilities

（无——本 change 不修改既有 spec 的 Requirement。）

## Impact

- 依赖的其他 change（按 ROADMAP 依赖图，合入前需已验收）：
  - `impl-capability-chat-solve`：partner turn 复用 chat capability 的同一 agent loop（chat → part 边）。
  - `impl-turn-runtime`：turn 生命周期、`UnifiedContext` 组装、事件持久化与 session store。
  - 间接/关联：`impl-agent-loop`（经 chat capability 间接依赖）、`impl-multi-user-auth`（partner 合成 scope 与 `/api/v1/partners` 的 admin 门控守卫）、`impl-cron-events`（destroy 时清理 `partner:<id>` 名下 cron job）。
- 被依赖：`impl-cli-launcher`（`deeptutor partner` 子命令与 launcher 启动时 auto-start partners）、`impl-subagents`（partner backend 以 partner runtime 为 consult 目标）。
- 新增 Go 包：`internal/partners/`（manager、runtime、workspace、channels、bus）、`internal/api/` 下 partners router 与 WS endpoint。
- 新增外部依赖：feishu/telegram/slack/discord 四个 channel SDK（选型与理由见 design.md）。
- 数据面：`data/partners/` 目录布局与基线逐一对等，Go 版可直接打开 Python v1.5.2 创建的 partner 数据（免迁移）。
- 前端影响：无——`/api/v1/partners` REST/SSE/WS 协议逐字段对等，`web/` 零改动。
