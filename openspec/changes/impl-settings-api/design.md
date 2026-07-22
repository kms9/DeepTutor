# Design: impl-settings-api

## Context

基线 REST 面分散在 `deeptutor/api/routers/settings.py`（约 40 条）、`system.py`、`tools.py`、`agent_config.py`、`mcp_settings.py`、`capabilities_settings.py`，存储底座是 `services/config/runtime_settings.py`（settings JSON + process-env override）。Go 版技术约束：gin route group 组织 `/api/v1/*`；settings JSON 经 spf13/viper 读写与热更新（foundation-config 提供 per-file viper 实例与 self-echo 抑制）；catalog 应用需重置 llm-provider 的进程级客户端工厂缓存。字段级对等以 OpenAPI golden spec 为准。

## Goals / Non-Goals

**Goals:**

- settings/system/tools/agent_config/mcp_settings/capabilities_settings/tour 全部路由组与基线逐字段对等。
- 「保存即生效」语义：写入 → viper 一致性 → 相关缓存失效（catalog → LLM/embedding 客户端重建）。
- admin 门禁与非 admin 脱敏（catalog 不可见、status 剥离 model/provider、grant 过滤）。

**Non-Goals:**

- runtime settings 服务本体（viper 实例管理、env override、WatchConfig、self-echo 抑制——foundation-config）；LLM 客户端工厂本体（llm-provider，本模块只调用其失效接口）；工具目录数据（tools-builtin 提供 `CatalogEntries()`）；多用户 grant 管理面（multi-user-auth）。

## Decisions

### D1. Go 包结构

```
internal/api/settingsapi/
├── router.go        # /api/v1/settings 组装 + admin 门禁中间件
├── ui.go            # UI 偏好组（theme/language/voice/timeout/ui/sidebar/themes/reset）
├── catalog.go       # GET/PUT /catalog、POST /apply、GET /llm-options、POST /fetch-models
├── network.go       # GET/PUT /network、GET/PUT /chat-attachments
├── parsing.go       # /document-parsing 组 + legacy /mineru 组（共享 MinerU 切片）
├── tests.go         # 即时型 /system/test/* + 流式型 /settings/tests/*（SSE run manager）
├── tools.go         # PUT /settings/enabled-tools + GET /api/v1/tools
├── mcp.go           # /api/v1/settings/mcp（GET/PUT/POST /test）
├── tour.go          # /settings/tour 组
├── system.go        # GET /system/runtime-topology、GET /system/status
├── agentconfig.go   # /agent-config 静态注册表
└── capabilities.go  # GET/PUT /capabilities/settings
```

### D2. viper 读写一致性（关键落位）

foundation-config 为每个 settings 文件（`interface.json`/`system.json`/`llm_configs.json`/`integrations.json`…）维护一个 viper 实例（`WatchConfig` 热更新 + 写回时 self-echo 抑制）。本模块统一经其门面读写：

```go
// internal/config/runtime.go（foundation-config 交付，此处为消费契约）
type RuntimeSettings interface {
    Get(file, key string) any                       // defaults 合并 + env override 后的 effective 值
    GetStored(file, key string) any                 // 仅存盘值（stored 视图）
    Mutate(ctx context.Context, file string, fn func(doc map[string]any) error) error
    // Mutate 语义：排他锁 → 读盘 → fn 修改 → 原子写盘 → 同步刷新对应 viper 实例（写后读一致，
    // 不依赖 WatchConfig 的异步回调）→ 标记 self-echo 抑制该次 fsnotify 事件
    Subscribe(file string, cb func()) (cancel func()) // 外部编辑热更新通知
}
```

写路径**同步刷新 viper** 而非等待 fsnotify，保证「PUT 后紧接 GET 必见新值」的读写一致性；外部（人工/回滚期 Python）编辑仍经 WatchConfig 进入。`stored` 与 `effective` 双视图（network/chat-attachments 响应要求）分别来自 `GetStored` 与 `Get`。

### D3. catalog apply 的缓存重置语义（catalog.go）

```go
// llm-provider 暴露的进程级工厂（消费契约）
type ClientFactory interface {
    Invalidate(reason string) // 重置 LLM + embedding 客户端缓存；下一次取用按新 settings 重建
}
```

`PUT /catalog` 与 `POST /apply`（及 `POST /tour/complete`）流程：校验 → `RuntimeSettings.Mutate` 落盘 → `ClientFactory.Invalidate("catalog applied")` → **WARNING 日志**（在飞轮次可能切换后端客户端，spec 明确要求记录）。embedding 指纹变化连带触发 knowledge 模块 reconcile（经其加载路径，不在本模块内联）。

### D4. 流式测试 run（tests.go）

`POST /settings/tests/{service}/start` 接受可选**草稿 catalog**：构造一次性 eino ChatModel/embedding 客户端（不进工厂缓存、不落盘）→ 后台 goroutine 执行测试 → 事件入 per-run buffer；`GET .../{run_id}/events` 以 SSE 推送（周期 heartbeat；`completed`/`failed` 终止）；`POST .../cancel` 协作式取消。run 管理复用与 memory run manager 相同的模式（seq buffer + SSE），实现独立避免跨模块耦合。

### D5. tools REST 与开关求交（tools.go）

- 数据源：`tools.Registry.CatalogEntries()`（tools-builtin），含 definition、`description_i18n`、`{en,zh}` prompt hints、aliases、`toggleable`、`coming_soon`（`geogebra_analysis` 恒 `coming_soon=true, enabled=false`，D-004）、`capability` 归属。
- `PUT /settings/enabled-tools`：输入 ∩ 当前合法 toggleable 名单 → 去重 → 持久化（退役名称静默丢弃）。
- 读取路径（`GET /tools` 与轮次 enabled 集合共用一个函数）：存量偏好 ∩ admin grant 白名单——被吊销工具既从响应隐藏也不进轮次工具集（防"存量配置复活"）。

### D6. MinerU token 三态与脱敏（parsing.go）

`PUT` 请求体 `api_token` 三态：字段缺省或 `null` → 保留存量；`""` → 清除；非空 → 替换。`GET` 恒回 `api_token_set: bool`，绝不回显原文。legacy `/mineru` 组与 `/document-parsing` 读写同一配置切片（同一 `Mutate(file="system.json", key="document_parsing.mineru")` 路径）。`install` 经 Sidecar 执行 pip；模型下载为后台 job（status/cancel 端点轮询）。

### D7. 与 Python 基线文件映射

| Python 基线 | Go 落位 |
| --- | --- |
| `api/routers/settings.py`（约 40 路由） | `internal/api/settingsapi/{router,ui,catalog,network,parsing,tests,tools,tour}.go` |
| `api/routers/system.py` | `internal/api/settingsapi/system.go` |
| `api/routers/tools.py` | `internal/api/settingsapi/tools.go`（数据源 `internal/tools/registry.go`） |
| `api/routers/agent_config.py` | `internal/api/settingsapi/agentconfig.go` |
| `api/routers/mcp_settings.py` | `internal/api/settingsapi/mcp.go` |
| `api/routers/capabilities_settings.py` | `internal/api/settingsapi/capabilities.go` |
| `services/config/runtime_settings.py` | foundation-config 的 `internal/config/`（本模块消费） |

### D8. 备选方案取舍

- **写后一致性**：备选「写盘后等 WatchConfig 回调刷新」——被否，fsnotify 异步导致 PUT 后 GET 读到旧值（contract test 必挂）；采用 Mutate 内同步刷新 + self-echo 抑制。
- **admin 门禁**：备选逐 handler 判断——被否，易漏；catalog/网络/解析/测试/mcp 等组统一挂 admin 中间件（gin group middleware），UI 偏好组不挂。
- **agent_config**：保持静态注册表硬编码（与基线一致），不做成配置文件。

## Risks / Trade-offs

- [路由数量大（约 40+），字段漂移风险] → OpenAPI golden spec contract test 逐条锁定；响应模型定义为显式 struct（避免 map 序列化的字段缺失/多余）。
- [catalog apply 与在飞轮次竞态] → 与基线一致仅 WARNING 记录，不做轮次粘滞（spec 明确该权衡）；客户端引用以取用时快照方式传递，避免半更新状态。
- [MCP manager 热重载阻塞 PUT 响应] → 重载带超时，超时返回已保存 + 连接状态为 pending 的响应结构（与基线实时状态语义一致，以 golden spec 为准）。
- [非 admin 脱敏遗漏（status/model/provider、catalog key）] → 脱敏在响应序列化层集中实现（admin/非 admin 两个响应 struct），单测断言非 admin 响应不含敏感键。
