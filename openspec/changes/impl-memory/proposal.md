# Change: impl-memory

## Why

deeptutor-go 需要在 Go 侧完整复刻三层文件型记忆子系统（L1 trace / L2 摘要 / L3 综合 + LLM consolidator + memory workbench REST + `read_memory`/`write_memory` 存储语义），且与基线 v1.5.2 的 `memory/` 目录逐字节兼容（老数据免迁移续写）。本 change 依据模块行为规格 `docs/golang-req/openspec/specs/memory/spec.md` 立项，对应 `docs/golang-req/openspec/ROADMAP.md` Wave 2「领域服务与工具」、里程碑 M2（acceptance.md M2 矩阵「Memory」项：基线 memory 数据续写；consolidator 四操作产出结构合法且带来源引用）。

## What Changes

- 实现与基线一致的三层目录布局：`trace/<surface>/<YYYY-MM-DD>.jsonl`（L1，UTC 日切）、`L2/<surface>.md` + `.meta.json`、`L3/{recent,profile,scope,preferences}.md` + `.meta.json`、`backup/<timestamp>/`（v1 迁移归档，幂等）。
- L1 append-only trace：per-surface 串行化写、永不向调用方抛错、坏行跳过遍历、按时间/按 id 集合回查。
- L2/L3 文档模型：`m_<ULID>` entry id、footnote 来源引用、原子写（临时文件 + rename）、ops（Add/Edit）原子应用与逐条接受/拒绝报告。
- LLM 驱动 consolidator 四操作（`update`/`audit`/`dedup`/`merge`），经 eino ChatModel 调用；`L3/preferences.md` 拒绝 `update`/`audit` 自动 consolidate。
- consolidator run 生命周期：run manager、单文档单活动 run（409）、断线后继续执行、`seq` 事件缓冲 + SSE 重放、cancel/undo、legacy 端点薄包装。
- memory workbench REST（`/api/v1/memory` 约 27 条路由）：overview/doc/resolve/run/settings/trace/snapshot 各组。
- snapshot 工作区镜像：实时派生 + `pending_changes` 差异 + `changes.jsonl` 提交。
- `read_memory`/`write_memory` 的存储侧语义（工具面契约在 tools-builtin spec）。
- 并发写保护：per-path 互斥锁串行化全部文档写路径。

## Capabilities

### New Capabilities

- `memory`: 三层文件型记忆子系统的 Go 侧行为契约——目录布局与数据兼容、L1 append-only、L2/L3 文档模型、consolidator 四操作与 run 生命周期、memory REST 工作台、snapshot、read_memory/write_memory 存储语义、并发写保护。

### Modified Capabilities

（无）

## Impact

- **依赖的其他 change**（按 ROADMAP 依赖图 `turn → mem`）：
  - `impl-turn-runtime`：memory trace 挂钩由各 capability 轮次在 turn 生命周期内触发；per-turn 用户上下文（多用户路径解析、partner scope 覆盖）来自 turn runtime 的 UnifiedContext。
  - 间接依赖（经上游 change 已交付）：`impl-foundation-config`（路径服务）、`impl-llm-provider`（consolidator 的 eino ChatModel 调用）。
- **被依赖**：`impl-tools-builtin` 的 `read_memory`/`write_memory` 消费本模块存储接口；各 capability 的 trace 挂钩。
- **受影响代码**：新增 `internal/memory/`（store/trace/document/ops/consolidator/snapshot）与 `internal/api/`下的 memory 路由组。
- **数据兼容**：`data/.../memory/` 免迁移续写（acceptance.md 3.2 B 类数据兼容），Go 写入后回切 Python 仍可打开。
- **前端影响**：Memory 工作台页面全功能（overview、文档编辑、run SSE、snapshot、trace 管理）。
