# Change: impl-knowledge

## Why

deeptutor-go 需要在 Go 侧承载知识库（KB）子系统的接入面——KB 生命周期、注册表、embedding reconcile、进度追踪与 progress WS、约 46 条 `/api/v1/knowledge` 路由——而把索引/检索 pipeline 本体留在 Python Sidecar（facade 模式）。本 change 依据模块行为规格 `docs/golang-req/openspec/specs/knowledge/spec.md` 立项，对应 `docs/golang-req/openspec/ROADMAP.md` Wave 2「领域服务与工具」、里程碑 M2（acceptance.md M2 矩阵「Knowledge 接入面」项：KB CRUD 在 Go、索引/检索经 Sidecar、progress WS 推送与基线一致）。数据布局 `data/knowledge_bases/<name>/` 必须与基线兼容，老 KB 免迁移可用。

## What Changes

- KB 目录布局与注册表兼容：`raw/` + `metadata.json` + 版本化索引目录（默认 provider `version-N/`），注册表 `kb_config.json` 文件锁 + 原子写，磁盘存在未注册 KB 自动注册。
- KB 创建/上传/删除全流程：`initializing` 状态登记、文档落 `raw/`、经 Sidecar 发起索引、task id + SSE 日志流。
- 索引/重建经 Sidecar 与版本化存储：Go 侧不内嵌任何 embedding/解析实现；按 embedding signature 选择/创建 `version-N/`，历史版本保留、切回复用。
- embedding 兼容性 reconcile：加载注册表时对照磁盘版本与当前指纹，维护 `needs_reindex`/`embedding_mismatch` 标记；连接型 KB 跳过。
- 进度追踪与 `WS /{kb_name}/progress/ws`（gorilla/websocket）：持久化进度帧、broadcaster 实时推送、无活动任务快速完成路径（发送终态后立即关闭）。
- 连接型 KB（Obsidian / 本地文件夹 / LightRAG server）与 linked folders 增量同步。
- RAG provider 配置面（rag-providers / rag-pipelines config / preflight / model-options / active-model）。
- KB 配置与默认 KB（含 `default`/`current`/`selected`/`默认` 别名归一）、文件与目录管理（文件树/建目录/移动/下载/预览截断/防穿越）。
- 多用户可见性与写保护：读开放可见 KB，写校验可写归属，`rag` 工具复用同一解析逻辑。

## Capabilities

### New Capabilities

- `knowledge`: KB 子系统 Go 侧行为契约——目录/注册表兼容、生命周期、Sidecar facade 索引与版本化、embedding reconcile、进度 WS、连接型 KB 与 linked folders、provider 配置面、多用户可见性。

### Modified Capabilities

（无）

## Impact

- **依赖的其他 change**（按 ROADMAP 依赖图 `turn → kb`、`side → kb`）：
  - `impl-turn-runtime`：KB 附加到轮次（UnifiedContext 的 KB 上下文）与多用户上下文传递。
  - `impl-sidecar-contract`：索引/检索/解析请求经 typed Sidecar 契约（`RAGIndex`/`RAGSearch`/`ParseDocument`）转发。
  - 间接依赖：`impl-foundation-config`（路径服务与 settings）。multi-user-auth 的 KB 归属在 M2 阶段可先以单用户路径实现（spec 依赖说明）。
- **被依赖**：`impl-tools-builtin` 的 `rag` 工具复用本模块 KB 归属解析。
- **受影响代码**：新增 `internal/knowledge/`（manager/registry/reconcile/progress/connected/linked）与 `internal/api/knowledgeapi/`（REST + progress WS）。
- **前端影响**：Knowledge Center 全页面（列表/创建/上传/进度/重建 CTA/连接型 KB/provider 设置）；Sidecar 不可用时 KB 请求返回明确错误（acceptance.md 3.4 Sidecar 降级）。
