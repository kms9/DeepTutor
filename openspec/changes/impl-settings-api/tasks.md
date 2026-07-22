# Tasks: impl-settings-api

## 1. 存储与生效语义基座

- [ ] 1.1 对接 foundation-config 的 `RuntimeSettings` 门面：`Mutate` 同步刷新 viper（写后读一致）+ self-echo 抑制、stored/effective 双视图读取
- [ ] 1.2 catalog 缓存重置钩子：`PUT /catalog`、`POST /apply` 落盘后调用 llm-provider `ClientFactory.Invalidate` 并记 WARNING
- [ ] 1.3 `restart_required` 标注机制：网络端口、WS 帧上限等启动期固定项在响应中显式标注
- [ ] 1.4 admin 门禁中间件（catalog/网络/解析/测试/mcp/tour 组挂载；UI 偏好组全员）

## 2. UI 偏好与 catalog 组

- [ ] 2.1 UI 偏好组：`PUT /theme`（四主题枚举）、`PUT /language`、`PUT /voice-autoplay`、`PUT /chat-response-timeout`（30–1800 越界 422）、`PUT /ui`、`PUT /sidebar/description`、`PUT /sidebar/nav-order`、`GET /sidebar`、`GET /themes`、`POST /reset`；defaults 合并与损坏回退
- [ ] 2.2 catalog 组：`GET /settings`（admin 含 `ui`+`catalog`+`providers`；非 admin 仅 `ui` 不暴露 URL/key）、`GET/PUT /catalog`、`POST /apply`、`GET /llm-options`（admin 全量 / 非 admin grant 过滤）、`POST /fetch-models`（OpenAI 兼容拉取、provider 错误映射 502）

## 3. 网络、附件与解析引擎组

- [ ] 3.1 `GET/PUT /network`：端口/`public_api_base`/CORS origins 归一化、stored+effective 双视图、auth/cookie 派生状态
- [ ] 3.2 `GET/PUT /chat-attachments`：stored/effective/bounds 三段、bounds 越界校验拒绝、GET 全员可读、WS 帧上限重启标注
- [ ] 3.3 `/document-parsing` 组：GET（引擎切片 + MinerU token 脱敏为布尔 + 就绪度/可安装清单）、PUT（激活切换 + 局部合并 + token 三态）、`POST /test`、`POST /install`（经 Sidecar）、`POST /models/download` + `GET /job/status` + `POST /job/cancel`
- [ ] 3.4 legacy `/mineru` 组（GET/PUT/test/models/download/status/cancel）共享同一 MinerU 配置切片

## 4. 测试组、tools、system 与其余路由组

- [ ] 4.1 即时型：`POST /system/test/llm`、`/system/test/embeddings`、`/system/test/search`（success/耗时/错误）
- [ ] 4.2 流式型：`POST /settings/tests/{service}/start`（草稿 catalog 一次性客户端不落盘）、`GET .../{run_id}/events`（SSE + heartbeat、completed/failed 终止）、`POST .../cancel`
- [ ] 4.3 `PUT /settings/enabled-tools`（合法 toggleable 过滤 + 去重 + 退役名称静默丢弃）；读取路径与 admin grant 求交（吊销不可复活），轮次 enabled 集合共用同一函数
- [ ] 4.4 `GET /api/v1/tools`：全条目（definition/`description_i18n`/`{en,zh}` hints/aliases/toggleable/enabled/coming_soon/capability），grant 外 toggleable 隐藏，`geogebra_analysis` coming_soon 条目（D-004）
- [ ] 4.5 system：`GET /system/runtime-topology`、`GET /system/status`（非 admin 剥离 `model`/`provider`）
- [ ] 4.6 `GET /agent-config/agents`（四条静态 UI 元数据）与 `GET /agent-config/agents/{agent_type}`（未知类型 error 负载）
- [ ] 4.7 mcp_settings：`GET`（注册表 + manager 实时状态）、`PUT`（transport 归一 + URL 校验 + 保存热重载）、`POST /test`（探活）；admin 门禁
- [ ] 4.8 capabilities_settings：`GET`（defaults 合并完整 schema）、`PUT`（局部合并保存即时生效）
- [ ] 4.9 tour 组：`GET /settings/tour/status`、`POST /settings/tour/complete`（应用 catalog + 失效缓存 + launch_at/redirect_at）、`POST /settings/tour/reopen`

## 5. 测试与验收（spec Scenario 落测试 + 协议对等）

- [ ] 5.1 将本 spec 全部 13 个 Scenario 落为 Go 测试：catalog 应用即时生效（WARNING + 新 profile 生效）、网络 restart_required、偏好合并默认、非 admin 不见 catalog、附件越界拒绝、token 三态、草稿测试不污染、吊销工具不复活、非 admin status 脱敏、agent 注册表、MCP 保存热重载、能力参数即时生效、tour complete
- [ ] 5.2 settings/system/tools/agent-config/mcp/capabilities/tour 全路由 contract test 对 OpenAPI golden spec 逐字段 diff（acceptance.md 3.1 A 类 + M2 矩阵「Settings/System/Personas/Tools API」项：前端 Settings 全页面可用）
- [ ] 5.3 数据兼容：指向基线 `data/user/settings/*.json` 直接生效、Go 写入后回切 Python 可读（acceptance.md 3.2 B 类）
- [ ] 5.4 与 impl-tools-builtin 联测：`GET /tools` 条目与工具注册表一致、`enabled-tools` 开关影响轮次工具集（M2 端到端）
