# settings-api Specification

## Purpose

本模块定义配置与系统状态 REST 面在 Go 侧的目标行为：settings REST（`/api/v1/settings`，约 40 条路由：UI 偏好、模型 catalog、网络、chat 附件策略、文档解析引擎、服务测试、工具开关、tour）、system REST（`/api/v1/system`：拓扑/状态/自检）、tools REST（`/api/v1/tools`：内置工具目录）、agent_config、mcp_settings（`/api/v1/settings/mcp`）与 capabilities_settings。核心语义：全部运行时配置写入 `data/user/settings/*.json`（经 foundation-config 的 runtime settings 服务），保存后对后续请求即时生效——仅少数显式标注 `restart_required` 的项（网络端口、WS 帧上限）需要重启。字段级对等由 OpenAPI golden spec 保证，本 spec 按功能组约束行为。

- 参考实现（基线）：`deeptutor/api/routers/settings.py`、`system.py`、`tools.py`、`agent_config.py`、`mcp_settings.py`、`capabilities_settings.py`、`deeptutor/services/config/runtime_settings.py`
- 依赖 spec / 里程碑：依赖 foundation-config（settings JSON 读写与 process-env override）、llm-provider（catalog 应用与连通性测试）、multi-user-auth（admin 门禁）；里程碑：M2

## Requirements

### Requirement: 配置写入即时生效
所有 settings 路由的持久化 SHALL 落到 `data/user/settings/*.json`（interface/system/catalog 等分文件），读取时 defaults 合并、process-env override 生效（语义见 foundation-config spec）。模型 catalog 保存/应用后 SHALL 立即失效并重建进程级 LLM 与 embedding 客户端缓存，使后续轮次用新配置——正在飞行中的轮次可能因此切换后端客户端，系统 SHALL 以 WARNING 日志记录该操作。网络端口与 WS 帧上限等进程启动期固定的项 SHALL 在响应中显式标注 `restart_required`。

#### Scenario: catalog 应用即时生效
- **WHEN** 管理员 `POST /settings/apply` 应用了新的 LLM profile
- **THEN** 进程级 LLM/embedding 客户端缓存被重置并记录 WARNING
- **AND** 下一个 chat 轮次即使用新 profile，无需重启

#### Scenario: 网络设置标注需重启
- **WHEN** 管理员 `PUT /settings/network` 修改 backend_port
- **THEN** 设置写入 system settings 文件且响应含 `restart_required: true`

### Requirement: UI 偏好组
系统 SHALL 提供个人 UI 偏好路由（任何已认证用户可写）：`PUT /theme`（`light`/`dark`/`glass`/`snow`）、`PUT /language`（`zh`/`en`）、`PUT /voice-autoplay`、`PUT /chat-response-timeout`（30–1800 秒，越界 422）、`PUT /ui`（批量）、`PUT /sidebar/description`、`PUT /sidebar/nav-order`、`GET /sidebar`、`GET /themes`、`POST /reset`（恢复默认）。读取 SHALL 将存盘值与默认值合并，损坏文件回退默认。

#### Scenario: 偏好合并默认值
- **WHEN** interface settings 文件缺少 `chat_response_timeout` 字段
- **THEN** `GET /settings` 返回默认值 180

### Requirement: 模型 catalog 组与权限分层
系统 SHALL 提供：`GET /settings`（admin 返回 `ui` + `catalog` + `providers` 下拉选项；非 admin 仅返回 `ui`，SHALL NOT 暴露 provider URL/key）、`GET /catalog`、`PUT /catalog`（保存并失效缓存）、`POST /apply`（把 catalog 应用到 runtime settings）、`GET /llm-options`（admin 返回全量选项，非 admin 返回 grant 过滤后的选项）、`POST /fetch-models`（按 `binding`+`base_url`+`api_key` 向 OpenAI 兼容 provider 拉取模型 id 列表，provider 错误映射 502）。catalog/网络/解析/测试类路由 SHALL 一律 admin 门禁（403）。

#### Scenario: 非 admin 不见 catalog
- **WHEN** 非 admin 用户请求 `GET /settings`
- **THEN** 响应仅含 `ui` 键
- **AND** 其模型选择经 `GET /llm-options` 得到 grant 过滤后的列表

### Requirement: 网络与 chat 附件策略组
`GET/PUT /network` SHALL 读写端口、`public_api_base` 与 CORS origins（归一化），响应同时给出 stored 与 effective 两套视图及 auth/cookie 派生状态。`GET/PUT /chat-attachments` SHALL 读写附件尺寸与抽取字符预算（PUT admin 门禁、GET 全员可读供 composer 客户端限流），响应含 stored/effective/bounds 三段；尺寸与字符上限 SHALL 每消息即时生效，WS 帧上限 SHALL 标注需重启才能放大。

#### Scenario: 附件上限越界被拒
- **WHEN** 管理员提交超出 bounds 的 `max_file_mb`
- **THEN** 请求以校验错误被拒，存量配置不变

### Requirement: 文档解析引擎组
系统 SHALL 提供多引擎文档解析控制面：`GET /document-parsing`（激活引擎、各引擎配置切片（MinerU `api_token` 脱敏为布尔）、可用引擎、就绪度报告、可安装/可下载模型的引擎清单）、`PUT /document-parsing`（切换激活引擎 + 按引擎局部合并配置；MinerU token 三态：缺省/`null` 保留、空串清除、非空替换）、`POST /document-parsing/test`（指定或激活引擎的就绪测试）、`POST /document-parsing/install`（一键 pip 安装可选引擎，经 Sidecar 执行）、`POST /document-parsing/models/download` 与 `GET /document-parsing/job/status`、`POST /document-parsing/job/cancel`（模型下载后台任务）。legacy `/mineru` 组（GET/PUT/test/models/download/status/cancel）SHALL 保留并与新组读写同一份 MinerU 配置切片。

#### Scenario: token 三态语义
- **WHEN** 管理员在未编辑 token 字段的情况下保存 MinerU 设置（payload `api_token=null`）
- **THEN** 存量 token 保留且 GET 响应仅回 `api_token_set=true`，绝不回显原文

### Requirement: 服务连通性测试组
系统 SHALL 提供两套自检：即时型 `POST /system/test/llm`、`/system/test/embeddings`、`/system/test/search`（真实调用一次并返回 success/耗时/错误）；流式型 `POST /settings/tests/{service}/start`（可携带草稿 catalog 启动测试 run，返回 run_id）+ `GET /settings/tests/{service}/{run_id}/events`（SSE 逐事件推送，含 heartbeat，`completed`/`failed` 终止）+ `POST .../cancel`。流式测试 SHALL 支持在不落盘的草稿配置上运行。

#### Scenario: 草稿配置测试不污染存量
- **WHEN** 管理员对未保存的 catalog 草稿启动 llm 测试
- **THEN** 测试使用草稿配置执行且存盘 catalog 不变

### Requirement: 工具开关与 tools REST
`PUT /settings/enabled-tools` SHALL 持久化用户可切换工具集合，输入 SHALL 过滤为当前合法 toggleable 名单并去重（退役名称静默丢弃）；读取路径 SHALL 再与 admin grant 白名单求交，使被吊销的工具无法经存量配置复活。`GET /api/v1/tools` SHALL 返回全部内置工具条目：definition（name/description/parameters）、`description_i18n` 与 `{en,zh}` prompt hints、aliases、`toggleable`（仅用户可切换集合为 true）、`enabled`（toggleable 反映用户偏好，锁定工具恒 true，coming_soon 恒 false）、`coming_soon`、`capability`（capability 拥有的工具标注归属）；grant 之外的 toggleable 工具 SHALL 从响应中隐藏。

#### Scenario: 吊销的工具不可复活
- **WHEN** 用户存量配置启用了 `imagegen` 而 admin 已把它移出该用户 grant
- **THEN** `GET /tools` 不再返回 `imagegen`
- **AND** 轮次的 enabled 工具集也不含它

### Requirement: system REST
系统 SHALL 提供 `GET /system/runtime-topology`（描述主运行时：`/api/v1/ws` 传输、orchestrator、session store 与兼容路由清单，供运维/前端识别统一运行时）与 `GET /system/status`（backend/llm/embeddings/search 各段状态：configured/not_configured/error 及 search 的 provider 特例态；非 admin 响应 SHALL 剥离 `model` 与 `provider` 字段防部署指纹泄露）。

#### Scenario: 非 admin 状态脱敏
- **WHEN** 非 admin 用户请求 `GET /system/status`
- **THEN** 各段仅含状态标记，不含模型名与 search provider 名

### Requirement: agent_config REST
系统 SHALL 提供 `GET /agent-config/agents`（solve/question/research/co_writer 四种 agent 的 UI 元数据：icon/color/label_key，静态注册表）与 `GET /agent-config/agents/{agent_type}`（未知类型返回 error 负载）。

#### Scenario: 前端数据驱动渲染
- **WHEN** 前端请求 agents 注册表
- **THEN** 返回与基线一致的四条 UI 元数据

### Requirement: mcp_settings REST
系统 SHALL 提供 admin 门禁的 `/api/v1/settings/mcp`：`GET`（server 注册表 + manager 实时连接状态）、`PUT`（校验各 server 的 transport 归一结果与 URL 合法性后保存并热重载 manager）、`POST /test`（保存前探活单个 server 配置）。注册表为部署全局状态；stdio server 的 `command` 在宿主机执行，故 SHALL NOT 向非 admin 开放编辑。

#### Scenario: 保存即热重载
- **WHEN** 管理员 PUT 新的 MCP server 注册表
- **THEN** 配置持久化且 MCP manager 重载连接，响应返回最新连接状态

### Requirement: capabilities_settings REST
系统 SHALL 提供 `GET /capabilities/settings`（各 capability 可调参数——temperature、max_tokens、阶段预算、迭代上限——defaults 合并后的完整 schema）与 `PUT /capabilities/settings`（局部合并保存并返回新状态），保存后对后续轮次即时生效。

#### Scenario: 调整能力参数即时生效
- **WHEN** 管理员降低 deep_research 的迭代上限并保存
- **THEN** 下一次 deep_research 轮次按新上限执行

### Requirement: 引导 tour 组
系统 SHALL 提供 `GET /settings/tour/status`（读取 tour 缓存文件的 active/status/launch_at/redirect_at）、`POST /settings/tour/complete`（admin：应用 catalog + 失效缓存 + 更新缓存文件并返回重启/跳转时间戳）、`POST /settings/tour/reopen`（返回指引用户在终端运行 `deeptutor init` 的提示）。

#### Scenario: 完成引导即应用配置
- **WHEN** 管理员在首次引导中 `POST /tour/complete` 且携带测试通过的 catalog
- **THEN** catalog 被应用、运行时缓存重置，响应给出前端倒计时所需的 launch_at/redirect_at
