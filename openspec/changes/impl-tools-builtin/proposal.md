# Change: impl-tools-builtin

## Why

deeptutor-go 需要与基线 v1.5.2 行为对等的全部内置工具（Level 1 Tool），否则 chat agent loop 没有可调用的能力面。本 change 依据模块行为规格 `docs/golang-req/openspec/specs/tools-builtin/spec.md` 立项，对应 `docs/golang-req/openspec/ROADMAP.md` 的 Wave 2「领域服务与工具」；实施节奏为三批：第一批 LLM/HTTP 类随 M1 chat 主链路交付，第二批本地状态类与第三批 Sidecar 类随 M2 各后端模块交付（见 ROADMAP 第 4 节里程碑映射与 acceptance.md M1/M2 验收矩阵）。

## What Changes

- 定义 Go 侧统一工具协议：OpenAI function-calling 兼容 schema（`type=array` 参数补 `items`）、统一 tool result 结构（`content` / `sources` / `metadata` / `success` / `pause_for_user`），`success=false` 不终止轮次。
- 服务端注入参数（`_sandbox_*`、`_workspace_dir`、`_cron_owner`、`_tool_loader`、`source_index`、`conversation_history` 等）由 pipeline 注入，不出现在 LLM schema 中、不可被 LLM 覆盖。
- 注册与基线一致的工具目录：用户可切换集合（`brainstorm`、`web_search`、`paper_search`、`reason`、`geogebra_analysis`、`imagegen`、`videogen`）与自动挂载集合（`rag`、`code_execution`、`read_source`、`read_memory`、`write_memory`、`read_skill`、`list_notebook`、`write_note`、`web_fetch`、`github`、`exec`、`load_tools`、`cron`、`ask_user`）；支持 tool alias 表（`rag_hybrid`/`rag_naive`/`rag_search` → `rag`，`code_execute`/`run_code` → `code_execution`）。
- `geogebra_analysis` 列入 `COMING_SOON`：不注册运行时、`/api/v1/tools` 返回条目并标 `coming_soon=true`（有意差异 D-004，见 acceptance.md 第 6 节）。
- 实现三批工具：第一批 `brainstorm`/`reason`/`web_search`/`paper_search`/`web_fetch`/`github`/`imagegen`/`videogen`；第二批 `read_source`/`read_skill`/`load_tools`/`read_memory`/`write_memory`/`list_notebook`/`write_note`/`cron`/`ask_user`；第三批 `rag`/`exec`/`code_execution`（Sidecar 转发）。
- 工具实现落位为 eino tool 组件（`tool.InvokableTool`，schema 由 `ToolInfo` 描述），供 agent-loop 的 eino tool-calling 循环消费。

## Capabilities

### New Capabilities

- `tools-builtin`: deeptutor-go 全部内置工具的行为契约——工具 schema、输入参数语义、tool result 归一（含 `sources` 进 `stream.sources`）、用户可切换/自动挂载两类集合划分、alias、三批工具的逐工具行为。

### Modified Capabilities

（无——本 change 只新增 `tools-builtin` capability，不修改既有 spec 的 Requirement。）

## Impact

- **依赖的其他 change**（按 ROADMAP 依赖图 `side → tools`、`loop → tools`）：
  - `impl-agent-loop`：工具挂载时机、context gate、pause/resume（`ask_user`）与 tool result 进入下一轮上下文的循环语义由 agent-loop 承载。
  - `impl-sidecar-contract`：第三批 `rag`/`exec`/`code_execution` 经 typed Sidecar 契约转发。
  - 第二批工具对接 `impl-memory`（read_memory/write_memory 存储语义）、`impl-skills-persona`（read_skill/load_tools 服务侧）、`impl-notebook`（list_notebook/write_note 存储侧）、`impl-cron-events`（cron 持久化与执行）——工具面本 change 交付，存储/服务侧语义由对应 change 交付，可先以接口 stub 并行。
- **受影响代码**：新增 `internal/tools/`（工具实现 + 注册表接入）；`/api/v1/tools` 列表条目行为与 `impl-settings-api` 的 tools REST 共享目录数据。
- **前端影响**：`/settings/tools` 页开关集合与 coming_soon 占位卡片（D-004 需手工验证）；chat 页 `stream.sources` 引用卡片。
