# Change: impl-settings-api

## Why

deeptutor-go 需要与基线对等的配置与系统状态 REST 面（settings 约 40 条路由 + system + tools + agent_config + mcp_settings + capabilities_settings），否则前端 Settings 全页面不可用。本 change 依据模块行为规格 `docs/golang-req/openspec/specs/settings-api/spec.md` 立项，对应 `docs/golang-req/openspec/ROADMAP.md` Wave 2「领域服务与工具」、里程碑 M2（acceptance.md M2 矩阵「Settings/System/Personas/Tools API」项：contract test 通过、前端 Settings 全页面可用）。核心语义：全部运行时配置写入 `data/user/settings/*.json`（经 foundation-config 的 runtime settings 服务），保存后即时生效，仅少数项标注 `restart_required`。

## What Changes

- 配置写入即时生效：settings 持久化落 `data/user/settings/*.json`（defaults 合并、process-env override）；模型 catalog 保存/应用后立即失效并重建进程级 LLM 与 embedding 客户端缓存（WARNING 日志）；网络端口/WS 帧上限标注 `restart_required`。
- UI 偏好组：theme/language/voice-autoplay/chat-response-timeout（30–1800 越界 422）/ui 批量/sidebar/themes/reset，读取 defaults 合并、损坏回退。
- 模型 catalog 组与权限分层：`GET /settings`（非 admin 仅 `ui`）、`GET/PUT /catalog`、`POST /apply`、`GET /llm-options`（grant 过滤）、`POST /fetch-models`（provider 错误映射 502）；catalog/网络/解析/测试类路由 admin 门禁。
- 网络与 chat 附件策略组：stored/effective 双视图、附件 bounds 校验、WS 帧上限重启标注。
- 文档解析引擎组：多引擎控制面（MinerU token 三态与脱敏）、test/install（经 Sidecar）/models/download 后台任务、legacy `/mineru` 组共享同一配置切片。
- 服务连通性测试组：即时型 `POST /system/test/*` 与流式型 tests run（SSE + heartbeat + cancel，支持草稿 catalog 不落盘）。
- 工具开关与 tools REST：`PUT /settings/enabled-tools`（合法名单过滤去重）、读取与 admin grant 求交；`GET /api/v1/tools` 全条目（definition/i18n/aliases/toggleable/enabled/coming_soon/capability）。
- system REST：`GET /system/runtime-topology` 与 `GET /system/status`（非 admin 剥离 `model`/`provider` 防指纹泄露）。
- agent_config REST（静态注册表）、mcp_settings REST（admin 门禁、保存热重载、探活）、capabilities_settings REST（defaults 合并 schema + 局部合并保存即时生效）、引导 tour 组。

## Capabilities

### New Capabilities

- `settings-api`: 配置与系统状态 REST 面的 Go 侧行为契约——即时生效语义、UI 偏好、catalog 权限分层、网络/附件/解析引擎/连通性测试、工具开关与 tools 目录、system/agent_config/mcp/capabilities/tour 各组。

### Modified Capabilities

（无）

## Impact

- **依赖的其他 change**：
  - `impl-foundation-config`：settings JSON 读写（viper 实例、WatchConfig 热更新、self-echo 抑制）与 process-env override 语义是本模块全部路由的存储底座。
  - 间接依赖：`impl-llm-provider`（catalog 应用与连通性测试的客户端构造）；admin 门禁在 multi-user-auth 交付前按单用户 admin 语义实现。
- **被依赖/协作**：`GET /api/v1/tools` 消费 `impl-tools-builtin` 的工具目录（含 D-004 的 `geogebra_analysis` coming_soon 条目）；catalog apply 重置的 LLM 客户端缓存属 `impl-llm-provider` 的进程级工厂。
- **受影响代码**：新增 `internal/api/settingsapi/`（settings/system/tools/agent_config/mcp/capabilities/tour 路由组）与 `internal/config/` 的 catalog 应用/缓存失效钩子。
- **前端影响**：Settings 全页面（外观/模型/网络/附件/解析/测试/工具/MCP/能力参数/引导）逐字段对等（OpenAPI golden spec 验收）。
