## Why

deeptutor-go 需要迁移认证与多用户子系统：JWT 登录/令牌校验（`auth.json` 的 `enabled` 开关，关闭即单用户本地模式）、用户身份存储与角色、按用户隔离的 workspace、管理员资源授权（grants）与请求级用户上下文。行为规格见 `docs/golang-req/openspec/specs/multi-user-auth/spec.md`；按 `docs/golang-req/openspec/ROADMAP.md`，本模块列于 Wave 4、里程碑 M4，但其 spec 注明「认证与 scope 需先于 REST 面全面落地（M2）」——守卫与 CurrentUser 上下文是全部受保护路由的横切前提。基线用 Python `ContextVar` 隐式传播用户上下文，跨事件循环失配时会静默路由到 admin workspace（issue #481 根因）；Go 版改为显式 `context.Context` 传递（有意差异 D-005，行为对等、实现更安全）。

## What Changes

- 新增 `auth.enabled` 开关与单用户模式：关闭时全部请求视作 local admin（`local-admin`/`local`/`admin`），`/login` 返回「无需登录」、`/register` 与 profile 端点 400。
- 新增 bcrypt + HS256 JWT 登录：httponly Cookie `dt_token`（SameSite/Secure 属性随 `cookie_secure`）、Bearer 头与 Cookie 双提取、WS `?token=` 且 accept 前认证（失败 close 4001）、logout 删 Cookie 属性对齐；PocketBase 代理模式后置（有意差异，首版仅内置 JWT + bcrypt）。
- 新增用户身份存储：`data/user/auth_users.json` 记录格式（兼容旧平面格式迁移）、`u_<uuid hex>` id、持锁原子写、首用户注册即 admin、之后 `/register` 关闭仅 admin 建号。
- 新增 `/api/v1/auth` REST 面（14 条：status/login/logout/register/is_first_user、profile 与头像上传（magic bytes 嗅探、拒绝 SVG）、admin 用户管理）。
- 新增请求级用户上下文：显式 `context.Context` 携带 `CurrentUser{id, username, role, scope}`（**D-005**），中间件认证后注入、服务层 API 强制接收 context，缺失时显式失败不静默回落 admin。
- 新增多用户 workspace 隔离：`data/user/`（admin）、`data/users/<uid>/`（非 admin，惰性建树）、`data/system/`（部署级，不入沙箱），含 pre-v1.5 `multi-user/` 旧布局一次性幂等迁移。
- 新增管理员资源授权 grants（`data/system/grants/<uid>.json` v2 shape）：模型/KB/skill/partner/工具白名单、`mcp_tools=null` deny-by-default、`exec_enabled` 三态、秘密/路径字段递归拒绝、运行期生效。
- 新增 `/api/v1/multi-user` 管理面与审计日志（`data/system/audit/`），以及 `require_auth`/`require_admin` 两级守卫（HTTP 与 WS 语义对称）。

## Capabilities

### New Capabilities

- `multi-user-auth`: 认证与多用户全部行为——auth.enabled 开关与单用户模式、登录/JWT/Cookie、用户身份存储与首用户引导、auth REST 面、请求级用户上下文（D-005 显式 context）、workspace 隔离与旧布局迁移、grants 资源授权、multi-user 管理面与审计、require_auth/require_admin 守卫。

### Modified Capabilities

（无——本 change 不修改既有 spec 的 Requirement。）

## Impact

- 依赖的其他 change（按 ROADMAP 依赖图与 spec 声明，合入前需已验收）：
  - `impl-turn-runtime`：turn 执行链需在用户 scope 内寻址（turn → auth 边）；WS 入口的 CurrentUser 装配作用于 turn 生命周期。
  - `impl-foundation-config`：runtime settings 读取 `auth.json`（enabled/username/password_hash/token_expire_hours/cookie_secure）。
  - 间接/关联：`impl-knowledge`（`resolve_kb` 权限解析消费 grants）、`impl-partners`（partner 可见性与 admin 门控消费本模块守卫）。
- 被依赖：几乎全部 REST/WS router（守卫中间件）、`impl-partners`、`impl-subagents`（`GET /partners` 可见性、settings admin 门控）。
- 新增 Go 包：`internal/multiuser/`（identity、guards、userctx、paths、grants、audit、access）、`internal/api/` 下 auth 与 multi-user router。
- 新增外部依赖：`golang.org/x/crypto/bcrypt`（口令散列）、`golang-jwt/jwt/v5`（HS256 JWT）。
- 数据面：`data/user/auth_users.json`、`data/system/{auth,grants,audit,indexes}/`、`data/users/<uid>/` 布局与基线对等；旧平面用户格式与 pre-v1.5 `multi-user/` 树可无损迁移。
- 有意差异：**D-005**（显式 context 传递，替代 ContextVar 隐式传播；行为对等、上下文缺失必须报错）；PocketBase 代理模式后置。前端影响：无。
