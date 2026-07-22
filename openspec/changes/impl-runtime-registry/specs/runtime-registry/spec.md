# runtime-registry Delta Specification

> 事实源：`docs/golang-req/openspec/specs/runtime-registry/spec.md`（基线 v1.5.2 对等）。

## ADDED Requirements

### Requirement: 工具注册与注销
系统 SHALL 提供进程级单例 ToolRegistry，按 `ToolDefinition.name` 注册工具实例；首次获取全局 registry 时 SHALL 自动加载全部内置工具（`BUILTIN_TOOL_TYPES` 对应集合）。单个内置工具实例化失败 SHALL 仅记录警告并跳过，SHALL NOT 中断其余工具的加载；已注册同名工具 SHALL 跳过重复注册（先注册者保留）。`unregister(name)` SHALL 移除指定工具，对不存在的名字 SHALL 为 no-op（供 MCP 重载使用）。

#### Scenario: 内置工具加载时单个工具构造失败
- **WHEN** registry 执行内置工具加载，其中一个工具类型构造抛出异常
- **THEN** 该工具被跳过并记录警告
- **AND** 其余内置工具全部正常注册

#### Scenario: 注销不存在的工具
- **WHEN** 调用 `unregister("nonexistent")`
- **THEN** 调用正常返回，registry 状态不变

### Requirement: 工具别名解析
系统 SHALL 支持静态别名表 `TOOL_ALIASES`（别名 → (目标工具名, 默认参数)），基线内容为：`rag_hybrid` → (`rag`, `{mode: "hybrid"}`)、`rag_naive` → (`rag`, `{mode: "naive"}`)、`rag_search` → (`rag`, `{}`)、`code_execute` → (`code_execution`, `{}`)、`run_code` → (`code_execution`, `{}`)。解析规则：若请求名已是注册工具名则直接命中，SHALL NOT 再走别名表；否则查别名表并将别名默认参数与调用方参数合并，调用方参数 SHALL 覆盖同名默认参数。`get(name)` 与 `execute(name, ...)` SHALL 使用同一解析逻辑。

#### Scenario: 别名执行时参数合并
- **WHEN** 调用 `execute("rag_hybrid", query="x", mode="naive")`
- **THEN** 实际执行 `rag` 工具
- **AND** 最终参数为 `{query: "x", mode: "naive"}`（调用方的 `mode` 覆盖别名默认值）

#### Scenario: 注册名优先于别名
- **WHEN** 存在与某别名同名的已注册工具
- **THEN** 请求按注册工具直接命中，不应用别名默认参数

### Requirement: OpenAI function schema 生成
系统 SHALL 能将 `ToolDefinition` 转为 OpenAI function-calling 格式：`{type: "function", function: {name, description, parameters}}`。当定义携带 `raw_parameters`（完整 JSON Schema，MCP 适配工具使用）时 SHALL 原样输出并补默认 `type: "object"` 与空 `properties`，且优先于结构化 `parameters` 列表。结构化参数转换 SHALL 生成 `properties` 与 `required` 列表（`required=true` 的参数进入 required）；`enum` 存在时输出；`type="array"` 且未提供 `items` 时 SHALL 回退为 `{"type": "string"}`（严格 provider 如 Gemini/Anthropic 要求 items 必填）。`build_openai_schemas(names)` 传 `names=None` 时 SHALL 返回全部注册工具的 schema。

#### Scenario: array 参数缺省 items
- **WHEN** 某工具参数声明 `type="array"` 且未提供 `items`
- **THEN** 生成的 JSON Schema 中该属性包含 `"items": {"type": "string"}`

#### Scenario: raw_parameters 优先
- **WHEN** 工具定义同时携带 `raw_parameters` 与 `parameters` 列表
- **THEN** 输出 schema 的 `parameters` 来自 `raw_parameters` 原文（补齐默认 `type`/`properties`），忽略结构化列表

### Requirement: 工具列表查询与去重
`get_enabled(names)` SHALL 按输入顺序返回工具实例：未知名字（含别名解析失败）SHALL 跳过；经别名解析到同一目标工具的重复项 SHALL 只保留首次出现。`list_tools()` SHALL 返回全部已注册工具名。`deferred_tools()` SHALL 返回全部标记 `deferred=true` 的工具（渐进式披露池，MCP 工具均属此类）。

#### Scenario: 含未知与重复名的启用列表
- **WHEN** 调用 `get_enabled(["rag", "unknown_tool", "rag_hybrid"])`
- **THEN** 返回仅含一个 `rag` 实例（`rag_hybrid` 解析后与 `rag` 重复被去除，`unknown_tool` 被跳过）

### Requirement: Prompt hint 组装
系统 SHALL 为每个工具提供 `ToolPromptHints`（short_description / when_to_use / input_format / guideline / note / phase / aliases），默认实现从 `ToolDefinition.description` 派生 short_description。`build_prompt_text(names, format, language)` SHALL 支持 `list`、`list_with_usage`、`table`、`aliases`、`phased` 五种格式，未知格式 SHALL 报错。hint 文案 SHALL 按 `language`（`en`/`zh`）本地化。

#### Scenario: 不支持的 prompt 格式
- **WHEN** 调用 `build_prompt_text(names, format="unknown")`
- **THEN** 返回错误（基线抛 `ValueError: Unsupported prompt format`）

### Requirement: 工具执行分发
`execute(name, kwargs...)` SHALL 先做别名解析（含默认参数合并），再调用目标工具的 `execute` 并返回其 `ToolResult`；目标工具不存在 SHALL 返回「Unknown tool」错误。工具名参数 SHALL 与工具自身参数命名空间隔离——工具 schema 中若存在名为 `name` 的参数（如 `read_skill`、部分 MCP 工具），SHALL NOT 与被调工具名冲突（基线以 positional-only 参数实现）。

#### Scenario: 执行带 name 参数的工具
- **WHEN** 通过 registry 执行 `read_skill` 且工具参数含 `name="my-skill"`
- **THEN** registry 正确定位 `read_skill` 工具并把 `name="my-skill"` 作为工具参数传入

#### Scenario: 未知工具执行
- **WHEN** 调用 `execute("no_such_tool")`
- **THEN** 返回「Unknown tool: no_such_tool」错误，不产生工具副作用

### Requirement: ToolResult 契约
工具执行结果 SHALL 携带：`content`（回写给 LLM 的 `role=tool` 消息体）、`sources`（引用行，经 stream `sources` 事件浮出）、`metadata`（自由载荷，兼作结构化 UI 提示通道）、`success`（false 表示显式失败但 content 仍可读）、`terminate_turn`（true 表示 loop 在本次分发后终止，工具输出即终答）、`pause_for_user`（非空表示 loop 暂停等待用户回复，`ask_user` 使用，载荷形如 AskUserPayload）。

#### Scenario: pause_for_user 结果传递
- **WHEN** `ask_user` 工具返回带 `pause_for_user` 载荷的结果
- **THEN** registry 将该字段原样透传给调用方（loop 侧暂停语义见 agent-loop spec）

### Requirement: CapabilityRegistry 内置清单加载
系统 SHALL 维护内置 capability 清单（名字 → 实现），基线为 7 个：`chat`、`deep_solve`、`deep_question`、`deep_research`、`math_animator`、`visualize`、`mastery_path`。首次获取全局 registry 时 SHALL 加载全部内置 capability，随后加载插件 capability；单个 capability 加载失败 SHALL 记录警告并跳过；已注册同名 SHALL 跳过（内置优先，插件不覆盖内置）。`get(name)` 未命中 SHALL 返回空值而非报错。

#### Scenario: 单个 capability 加载失败
- **WHEN** 加载内置清单时 `visualize` 初始化抛异常
- **THEN** `visualize` 缺席但其余 6 个 capability 可用
- **AND** `list_capabilities()` 不包含 `visualize`

### Requirement: Capability manifest 查询
每个 capability SHALL 声明静态 `CapabilityManifest`（`name`、`description`、`stages`、`tools_used`、`cli_aliases`、`request_schema`、`config_defaults`）。`get_manifests()` SHALL 返回全部 capability 的 manifest 字典列表，并附加 `description_i18n`（按 i18n 表本地化的描述）。

#### Scenario: 列出 manifests
- **WHEN** 调用 `get_manifests()`
- **THEN** 每个条目包含 `name`/`description`/`description_i18n`/`stages`/`tools_used`/`cli_aliases`/`request_schema`/`config_defaults` 全部字段

### Requirement: Orchestrator 路由语义
`ChatOrchestrator.handle(context)` SHALL 是单轮执行的统一入口：若 `context.session_id` 为空 SHALL 生成 UUID 补齐；capability 选择 SHALL 取 `context.active_capability`，为空时默认 `chat`；选定 capability 不存在时 SHALL 通过流发出一条 `error` 事件（内容含未知名与可用列表）并随即结束流，SHALL NOT 抛异常给调用方。

#### Scenario: 默认路由到 chat
- **WHEN** `handle` 收到 `active_capability` 为空的 context
- **THEN** 由 `chat` capability 处理该轮

#### Scenario: 未知 capability
- **WHEN** `active_capability = "no_such_cap"`
- **THEN** 流中产出一条 `error` 事件，内容包含 `Unknown capability: no_such_cap` 与可用 capability 列表
- **AND** 流正常结束

### Requirement: Orchestrator StreamBus 生命周期与事件转发
路由成功后 orchestrator SHALL：(1) 先向调用方产出一条 `session` 事件，metadata 携带 `session_id` 与 `turn_id`（取自 `context.metadata["turn_id"]`，可为空串）；(2) 为该轮创建独立 StreamBus，若 turn_id 非空 SHALL 将 bus 以 turn_id 注册到全局 bus 注册表（供 `user_input` 消息路由，见 api-unified-ws spec），轮结束后注销；(3) 并发运行 `capability.run(context, bus)` 并把 bus 上的事件按序转发给调用方；(4) capability 抛异常时 SHALL 转为一条 `error` 事件（source 为 capability 名），SHALL NOT 让异常终止流；(5) 无论成功失败 SHALL 在末尾 emit 一条 `DONE` 事件（source 为 capability 名）并关闭 bus。

#### Scenario: capability 正常完成
- **WHEN** capability 在 bus 上产出若干事件后返回
- **THEN** 调用方依次收到 `session` 事件、capability 的全部事件、最后一条 `DONE` 事件

#### Scenario: capability 中途抛异常
- **WHEN** `capability.run` 抛出异常
- **THEN** 调用方收到一条 `error` 事件（内容为异常消息，source 为该 capability 名）
- **AND** 随后仍收到 `DONE` 事件，流正常收尾，bus 从注册表注销

### Requirement: 完成事件发布
每轮结束（含失败轮）后 orchestrator SHALL 向全局 EventBus 发布 `CAPABILITY_COMPLETE` 事件，携带 `capability` 名、`session_id`、`turn_id`（task_id 取 turn_id，缺省回退 session_id）与本轮 `user_input`。EventBus 不可用或发布失败 SHALL 静默容忍（仅 debug 日志），SHALL NOT 影响该轮结果。

#### Scenario: EventBus 未运行
- **WHEN** 某轮完成时全局 EventBus 不可用
- **THEN** `handle` 正常结束，调用方感知不到任何差异
