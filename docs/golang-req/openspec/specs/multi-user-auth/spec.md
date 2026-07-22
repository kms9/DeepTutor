# Multi-User Auth Specification

## Purpose
本模块定义 deeptutor-go 的认证与多用户子系统：JWT 登录/令牌校验（`auth.json` 的 `enabled` 开关，关闭时为单用户本地模式）、用户身份存储与角色（admin/user）、按用户隔离的 workspace（admin 用 `data/user/`，非 admin 用 `data/users/<uid>/`）、管理员对非 admin 用户的逻辑资源授权（模型/KB/skill/partner/工具白名单），以及请求级用户上下文的建立与传递。基线用 Python `ContextVar` 隐式携带当前用户；Go 版 SHALL 以显式 `context.Context` 值传递（有意差异：实现方式不同、行为对等——每个已认证入口在 handler 运行前装配同一 CurrentUser 语义）。

- 参考实现（基线）：`deeptutor/multi_user/`（`context.py`、`identity.py`、`models.py`、`paths.py`、`grants.py`、`router.py`、`model_access.py`、`tool_access.py`、`knowledge_access.py`、`partner_access.py`、`audit.py` 等 13 个文件）、`deeptutor/api/routers/auth.py`、`deeptutor/services/auth.py`
- 依赖 spec / 里程碑：依赖：foundation-config（runtime settings 读取）、knowledge-kb-manager（KB 权限解析）；里程碑：M2（认证与 scope 需先于 REST 面全面落地）

## Requirements

### Requirement: auth.enabled 开关与单用户模式
系统 SHALL 从 runtime settings 的 `auth.json` 读取 `enabled`（默认 false）、`username`、`password_hash`、`token_expire_hours`、`cookie_secure`。`enabled=false` 时系统 SHALL 处于单用户本地模式：所有请求视作 local admin（`user_id="local-admin"`、`username="local"`、`role="admin"`，admin scope），`/login` 直接返回「无需登录」，`/register` 与 profile 端点返回 400，`require_admin` 无条件放行并返回合成 local-admin 身份。`enabled=true` 时全部受保护路由 SHALL 要求有效 JWT。

#### Scenario: 关闭认证时全部视作 local admin
- **WHEN** `auth.json` 中 `enabled=false`，任意客户端不带凭据访问受保护 API
- **THEN** 请求以 local admin 身份放行，`GET /api/v1/auth/status` 返回 `{enabled:false, authenticated:true, user_id:"local-admin", role:"admin", is_admin:true}`

### Requirement: 登录、JWT 令牌与 Cookie
系统 SHALL 实现 bcrypt 口令校验 + HS256 JWT：payload 含 `sub`（username）、`role`、`uid`（user id）、`exp`（`token_expire_hours` 小时后）、`iat`；签名密钥为部署级随机 `auth_secret`，首次启用时自动生成并持久化（`data/system/auth/`）。登录成功 SHALL 以 httponly Cookie `dt_token` 下发令牌（`max_age = token_expire_hours*3600`；`cookie_secure=true` 时 `SameSite=None; Secure`，否则 `SameSite=Lax`），响应体含 `{ok, user_id, username, role, is_admin}`。令牌提取 SHALL 同时支持 `Authorization: Bearer <token>` 头与 `dt_token` Cookie；WebSocket SHALL 支持 `?token=` 查询参数与 Cookie，认证 SHALL 在 accept 前完成，失败以 close code 4001 拒绝。logout 的删除 Cookie SHALL 携带与创建时相同的属性集（否则浏览器静默保留旧 Cookie）。基线的 PocketBase 代理模式（`integrations.pocketbase_url`）SHALL 标记为后置 requirement（有意差异：首版仅内置 JWT + bcrypt 模式）。

#### Scenario: 凭据错误返回 401
- **WHEN** `POST /api/v1/auth/login` 提交错误口令
- **THEN** 返回 401 `Incorrect username or password`，不设 Cookie

#### Scenario: WS 在 accept 前认证
- **WHEN** 客户端向任一 WS 端点发起连接且令牌无效
- **THEN** 服务端不 accept，直接以 close code 4001 关闭

### Requirement: 用户身份存储与首用户引导
系统 SHALL 将用户存于 `data/user/auth_users.json`（经 `data/system/auth` 体系管理），记录形如 `{username: {id, hash, role, created_at, disabled, avatar}}`；user id 生成为 `u_<uuid hex>`（另有 `local-admin`/`env-admin` 哨兵值），API 层 SHALL 以 `^[A-Za-z0-9_-]{1,64}$` 校验后才允许触达文件系统。SHALL 兼容读取旧平面格式（`{username: "<hash>"}`）并迁移为记录格式；`auth.json` 中的 `username`+`password_hash` 作为单用户 bootstrap 用户合并进用户表。写入 SHALL 持锁原子进行（防注册竞态）。首用户规则：用户表为空时 `POST /register` 开放且注册者自动成为 admin；一旦存在用户，`/register` 关闭（403），后续账号只能由 admin 经 `POST /users` 创建（恒为 role=user，之后可提升）。username 校验 SHALL 接受标准 email 或 `[A-Za-z0-9_\-.]{3,64}` 的普通用户名，口令至少 8 字符。

#### Scenario: 第一个注册者成为 admin
- **WHEN** 用户表为空时 `POST /api/v1/auth/register` 成功
- **THEN** 返回 201 且 `{role:"admin", is_first_user:true, is_admin:true}`；随后第二次注册返回 403

### Requirement: /api/v1/auth REST 面
系统 SHALL 暴露以下端点（14 条）：公开——`GET /status`、`POST /login`、`POST /logout`、`POST /register`、`GET /is_first_user`；自助（须认证）——`GET /profile`、`PUT /profile`（仅接受空串或 `icon:<name>:<color>` 形式的 avatar marker；`img:` marker 由上传端点独占）、`PUT /profile/avatar`（上传图片：≤1MB，按 magic bytes 嗅探仅收 PNG/JPEG/WebP——SVG 明确拒绝以防存储型 XSS；marker 版本号递增 `img:<version>` 供缓存失效）、`DELETE /profile/avatar`、`GET /avatar/{user_id}`（任何已认证用户可读，响应带 `X-Content-Type-Options: nosniff` 与 `Content-Disposition: inline`）；admin——`GET /users`（不含 hash）、`POST /users`、`DELETE /users/{username}`（禁止删除自己，400；连带删除头像文件）、`PUT /users/{username}/role`（`admin`/`user`，禁止改自己角色，400）。

#### Scenario: 上传非法头像格式被拒
- **WHEN** `PUT /profile/avatar` 上传 SVG 文件
- **THEN** 返回 415 `Avatar must be a PNG, JPEG or WebP image.`

#### Scenario: admin 不能删除自己
- **WHEN** admin 调 `DELETE /users/<自己的 username>`
- **THEN** 返回 400 `You cannot delete your own account`

### Requirement: 请求级用户上下文（有意差异：显式 context 传递）
系统 SHALL 保证每个已认证入口（HTTP 与 WebSocket）在业务逻辑执行前解析出 `CurrentUser{id, username, role, scope}` 并沿调用链传递到所有按用户寻址的服务（路径解析、KB 管理、grants 查询等）。基线用 `ContextVar` 隐式传播并依赖「auth 依赖必须在与 handler 相同的事件循环上下文中 set」的不变量（违反即静默路由到 admin workspace 的事故根因）；Go 版 SHALL 改为显式 `context.Context` 携带 CurrentUser（有意差异），中间件在认证成功后注入，服务层 API 强制接收 context——从类型上消除「忘记装配用户上下文而静默落回 admin workspace」这一类缺陷。行为对等要求：payload 为空（auth 关闭）解析为 local admin；HTTP 请求结束即释放；WS 连接期间上下文持续有效直至连接关闭。

#### Scenario: 缺失用户上下文不得静默回落 admin
- **WHEN** 某个新增服务函数在没有 CurrentUser 的 context 下被调用且需要按用户寻址
- **THEN** 该调用编译期即要求传入 context，运行期无用户时显式失败或走明确定义的 local-admin 分支，SHALL NOT 静默使用 admin workspace 处理非 admin 用户请求

### Requirement: 多用户 workspace 隔离与目录布局
系统 SHALL 采用与基线一致的单树布局：`data/user/` 为 admin workspace（admin scope root 为 `data/`）；`data/users/<uid>/` 每个非 admin 用户一个 workspace（scope kind=`user`）；`data/partners/<id>/workspace/` 为 partner 合成 scope；`data/system/` 存部署级状态（`auth/`、`grants/`、`audit/`、`indexes/`），SHALL NOT 被挂载进沙箱运行器。非 admin scope 首次使用时 SHALL 惰性创建完整 workspace 树（含 `knowledge_bases/`、`memory/`）。admin 用户 SHALL 始终使用主 workspace（admin scope），SHALL NOT 拥有独立的 `data/users/` 目录。SHALL 实现 pre-v1.5 旧布局的一次性就地迁移：`multi-user/_system` → `data/system`、`multi-user/<uid>` → `data/users/<uid>`；目标已存在的条目跳过不覆盖并告警留待人工处理，迁移幂等且可在读路径上无条件调用。

#### Scenario: 非 admin 用户读写落入自己的 workspace
- **WHEN** role=user 的用户上传文档建 KB 并写 memory
- **THEN** 全部文件落在 `data/users/<其 uid>/` 之下，`data/user/`（admin workspace）无任何写入

#### Scenario: 旧 multi-user 树自动迁移
- **WHEN** 启动时存在 pre-v1.5 的 `multi-user/` 目录且 `data/users/` 中无同名目标
- **THEN** `_system` 与各用户目录被移动到 `data/system` 与 `data/users/<uid>`，空的旧根目录被删除

### Requirement: 管理员资源授权（grants）
系统 SHALL 为每个非 admin 用户维护一份 grant（`data/system/grants/<user_id>.json`，v2 shape）：`models.llm`（可用 LLM 白名单条目）、`knowledge_bases`、`skills`、`partners`（可见/可 consult 的 partner）、`enabled_tools`（三元语义：`null`=全池默认、`[]`=无、列表=白名单）、`mcp_tools`（非 admin 对 MCP 工具 SHALL 默认拒绝：`null` 即 deny，须 admin 显式授名）、`exec_enabled`（三态覆盖：`null` 随部署策略、`false` 恒拒、`true` 仅在沙箱具备用户级隔离时生效）。读取 SHALL 把任意历史 payload 规范化为 v2 shape（v1 中从未被消费的 `models.embedding`/`models.search`/`spaces` 丢弃，缺省字段视为不受限）。保存 SHALL 拒绝为 admin 用户建 grant（admin 直接用主 workspace），并 SHALL 递归校验 grant 内不得出现秘密/路径类字段（`api_key`、`secret`、`password`、`token`、`path`、`base_url` 及任何 `*_key`）——grant 只携带逻辑 id，运行期由服务端解析。授权 SHALL 在运行期生效：模型列表、KB 可见性、可挂载工具、partner 可见性均按当前用户 role+grant 过滤。

#### Scenario: grant 中夹带秘密被拒
- **WHEN** admin 提交的 grant payload 中含 `{"api_key": "sk-..."}` 字段
- **THEN** 保存被拒绝并报错指明违规字段路径，磁盘上的 grant 不变

#### Scenario: 非 admin 的 MCP 工具默认拒绝
- **WHEN** role=user 的用户 grant 中 `mcp_tools` 为 `null`
- **THEN** 该用户的 turn 不挂载任何 MCP 工具；admin 授予 `["tool_a"]` 后仅 `tool_a` 可用

### Requirement: /api/v1/multi-user 管理面与审计
系统 SHALL 在 `/api/v1/multi-user`（admin 门控）下暴露管理端点：`GET /resources`（列出全部可指派资源：激活的 LLM 模型、admin 的 KB、skill 目录、partner 卡片、内置工具池与 MCP 工具名）、`GET /users/{user_id}/grants`、`PUT /users/{user_id}/grants`、skill 安装到 admin 目录的端点。admin 的授权变更等敏感操作 SHALL 记入审计日志（`data/system/audit/`，含操作者、动作、目标、时间戳）。

#### Scenario: 授权变更留痕
- **WHEN** admin 通过 `PUT /users/{uid}/grants` 修改某用户可用模型
- **THEN** grants 文件更新且审计日志追加一条含 admin 身份与目标 user id 的记录

### Requirement: require_auth / require_admin 守卫语义
系统 SHALL 提供两级可复用守卫：`require_auth` 在 `enabled=true` 时校验令牌（缺失/无效 → 401，`WWW-Authenticate: Bearer`），成功后装配 CurrentUser 上下文；`require_admin` 在其上要求 `role=admin`（否则 403 `Admin access required`）。守卫 SHALL 对 HTTP 与 WebSocket 语义对称（同一 token 解析逻辑、同一 CurrentUser 装配），admin 门控整组应用于 partners CRUD、multi-user 管理面、subagent 设置写入等。

#### Scenario: 普通用户访问 admin 面被拒
- **WHEN** role=user 的已认证用户调 `GET /api/v1/partners`
- **THEN** 返回 403 `Admin access required`
