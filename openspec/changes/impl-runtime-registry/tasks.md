## 1. 协议类型（internal/core）

- [ ] 1.1 定义 `ToolDefinition` / `ToolParameter` / `ToolResult` / `ToolPromptHints` 与 `Tool` 接口（`tool_protocol.go`），字段与基线 `deeptutor/core/tool_protocol.py` 对齐
- [ ] 1.2 定义 `Capability` 接口与 `CapabilityManifest`（`capability_protocol.go`），含 `request_schema`/`config_defaults` 字段
- [ ] 1.3 定义 `UnifiedContext` 骨架（`session_id`/`active_capability`/`metadata` 等本模块消费的最小字段集）
- [ ] 1.4 定义 `EventPublisher` 接口 + no-op 实现与 `CAPABILITY_COMPLETE` 事件常量（`events.go`）

## 2. ToolRegistry

- [ ] 2.1 实现 `ToolRegistry`：`Register`（同名跳过）/ `Unregister`（no-op 容忍）/ `ListTools` / `DeferredTools`，并发安全
- [ ] 2.2 实现 `TOOL_ALIASES` 静态表与解析逻辑（注册名优先、别名默认参数合并、调用方参数覆盖），`Get` 与 `Execute` 共用
- [ ] 2.3 实现 `BuildOpenAISchemas`：结构化参数转换（`required`/`enum`/array 缺省 `items` 回退 `{"type":"string"}`）、`raw_parameters` 优先并补默认 `type`/`properties`、`names==nil` 返回全部
- [ ] 2.4 实现 `GetEnabled`：按输入顺序、别名解析后去重、未知名跳过
- [ ] 2.5 实现 `ToolPromptHints` 与 `BuildPromptText` 五种格式（`list`/`list_with_usage`/`table`/`aliases`/`phased`）+ en/zh 本地化，未知格式返回错误
- [ ] 2.6 实现 `Execute` 分发：别名解析 → 目标工具 `Execute` → 透传 `ToolResult`（含 `pause_for_user`/`terminate_turn`）；未知工具返回 `Unknown tool: <name>` 错误
- [ ] 2.7 实现 `Global()` 单例 + `bootstrap` 内置工具 factory 表惰性加载（单个 factory 失败仅告警跳过）

## 3. CapabilityRegistry

- [ ] 3.1 实现 `CapabilityRegistry`：内置 7 项 factory 清单加载（失败跳过、同名内置优先）、`Get` 未命中返回 `nil,false`、`ListCapabilities`
- [ ] 3.2 实现 `GetManifests`：全量 manifest 字典列表 + `description_i18n` 本地化附加

## 4. ChatOrchestrator

- [ ] 4.1 实现 `Handle`：session_id UUID 补齐、capability 选择（默认 `chat`）、未知 capability → `error` 事件 + 结束流（不抛异常）
- [ ] 4.2 实现事件转发管线：首帧 `session` 事件（携带 `session_id`/`turn_id`）→ 独立 StreamBus 创建与 turn_id 全局注册 → `capability.Run` goroutine + bus 事件按序转发 → panic/error 转 `error` 事件 → 末尾必发 `DONE` → close + bus 注销（`defer` 链保证全路径收尾）
- [ ] 4.3 实现 `CAPABILITY_COMPLETE` 发布（EventBus 缺席/失败静默容忍）与 `ctx.Done()` 取消传播（goleak 验证无泄漏）

## 5. Scenario 测试与对等验收

- [ ] 5.1 将 spec 全部 Scenario 落为 Go 测试：内置加载单项失败、别名参数合并/注册名优先、schema 边界（array items / raw_parameters）、`get_enabled` 去重、未知 prompt 格式、`execute` 未知工具与 `name` 参数隔离、`pause_for_user` 透传、capability 加载失败/manifest 查询、orchestrator 默认路由/未知 capability/正常完成/异常收尾/EventBus 缺席
- [ ] 5.2 协议对等验收：用基线导出的 golden OpenAI schema JSON 对 `BuildOpenAISchemas` 全集做逐字段 diff；orchestrator 事件序（`session`→…→`DONE`）与 acceptance.md M1「Agent Loop / 统一 WS」矩阵所需的事件形状用 WS golden fixtures 中 registry 可见子集回放比对
