## Context

基线实现位于 `deeptutor/multi_user/`（context/identity/models/paths/grants/router/model_access/tool_access/knowledge_access/partner_access/skill_access/audit 等 13 个文件）、`deeptutor/api/routers/auth.py` 与 `deeptutor/services/auth.py`。三块核心：认证（bcrypt + HS256 JWT + Cookie/Bearer/WS token）、多用户 workspace 隔离（`data/user/` vs `data/users/<uid>/` vs `data/system/`）、grants 资源授权与访问过滤。

关键历史包袱：基线以 Python `ContextVar` 隐式传播 CurrentUser，一旦 auth 依赖与 handler 不在同一事件循环上下文（如后台任务、二次派发），上下文静默丢失并回落 admin workspace（issue #481 根因）。Go 版按 **D-005**（acceptance.md 第 6 节）改为显式 `context.Context` 传递：行为对等、实现方式不同，从类型上根除该缺陷类。

本模块是横切前提：全部受保护 REST/WS 路由挂本模块守卫；`impl-partners`、`impl-subagents`、`impl-knowledge` 消费其 scope 解析与访问过滤。

## Goals / Non-Goals

**Goals:**

- `auth.json` 开关语义（单用户 local admin 模式 / JWT 模式）与 14 条 `/api/v1/auth` 端点；
- 显式 `context.Context` 用户上下文（D-005）与 `require_auth`/`require_admin` 守卫（HTTP/WS 对称）；
- workspace 隔离布局 + pre-v1.5 旧树幂等迁移 + 旧平面用户格式迁移；
- grants v2（含 `mcp_tools=null` deny-by-default、秘密字段递归拒绝）与运行期访问过滤、`/api/v1/multi-user` 管理面、审计日志。

**Non-Goals:**

- PocketBase 代理模式（`integrations.pocketbase_url`）——后置 requirement（有意差异，首版仅内置 JWT + bcrypt）；
- KB/skill/partner 本体管理（各自 change 交付；本模块只提供 grant 过滤钩子与 scope 路径解析）；
- 沙箱运行器本体（本模块只约定 `data/system/` 不入沙箱、`exec_enabled` 三态语义）；
- 前端登录/用户管理页面（零改动复用）。

## Decisions

### D1. Go 包结构

```
internal/multiuser/
├── userctx/
│   └── userctx.go     # CurrentUser 类型 + context 注入/提取（D-005 核心，独立小包防循环依赖）
├── identity.go        # auth_users.json 读写（持锁原子写、旧平面格式迁移、bootstrap 用户合并）
├── token.go           # HS256 JWT 签发/校验、auth_secret 生成持久化（data/system/auth/）
├── guards.go          # RequireAuth / RequireAdmin gin 中间件 + WS 前置认证助手
├── paths.go           # scope 路径解析（admin/user/partner scope root）、惰性建树、pre-v1.5 迁移
├── grants.go          # grant v2 读写/规范化/秘密字段递归校验
├── access.go          # 运行期过滤：model/tool/kb/partner 可见性（对应基线 *_access.py）
├── avatar.go          # 头像上传（magic bytes 嗅探）/读取/删除
└── audit.go           # data/system/audit/ 审计追加
internal/api/routers/auth.go       # /api/v1/auth（14 条）
internal/api/routers/multiuser.go  # /api/v1/multi-user 管理面
```

### D2. 显式用户上下文（D-005）

```go
// package userctx —— 唯一合法的 CurrentUser 载体是 context.Context。
type Scope struct {
    Kind string // "admin" | "user" | "partner"
    Root string // scope root 绝对路径
}

type CurrentUser struct {
    ID       string // "local-admin" / "env-admin" / "u_<uuid hex>"
    Username string
    Role     string // "admin" | "user"
    Scope    Scope
}

func With(ctx context.Context, u CurrentUser) context.Context

// From 提取 CurrentUser；不存在时返回 ok=false——调用方必须显式处理，
// 禁止提供「缺失即返回 admin」的兜底函数（D-005：不静默回落 admin）。
func From(ctx context.Context) (CurrentUser, bool)

// MustFrom 供按用户寻址的服务层使用：缺失即 error，错误信息指明调用点。
func MustFrom(ctx context.Context) (CurrentUser, error)
```

- 中间件在认证成功后 `c.Request = c.Request.WithContext(userctx.With(...))`；auth 关闭时注入合成 local admin（这是唯一的 local-admin 分支，显式而非兜底）。
- 服务层规约：所有按用户寻址的函数签名第一参数为 `ctx context.Context`，内部用 `MustFrom`；代码评审 + lint 规则（禁止在服务层调用 `context.Background()`）守住不变量。
- 行为对等：HTTP 请求结束 ctx 随之释放；WS 连接在 upgrade 前认证一次，CurrentUser 绑定连接生命周期（存入连接结构体，逐消息派生 ctx）。

### D3. 守卫与 JWT 落位

```go
// guards.go
func RequireAuth(deps GuardDeps) gin.HandlerFunc   // enabled=false → 注入 local admin；否则校验 token（401 + WWW-Authenticate: Bearer）
func RequireAdmin(deps GuardDeps) gin.HandlerFunc  // RequireAuth 之上要求 role=admin（403 "Admin access required"）

// WS 前置认证：在 gorilla/websocket Upgrade 之前调用；失败由调用方以 close code 4001 拒绝。
func AuthenticateWS(r *http.Request, deps GuardDeps) (userctx.CurrentUser, error) // ?token= / Cookie 双提取
```

- JWT：`golang-jwt/jwt/v5`，HS256，payload `{sub, role, uid, exp, iat}`；`auth_secret` 首次启用时 `crypto/rand` 生成并持久化 `data/system/auth/`。
- 口令：`golang.org/x/crypto/bcrypt`（与基线 passlib bcrypt hash 互认，同一 `$2b$` 格式——老用户表免迁移）。
- Cookie：`dt_token` httponly；`cookie_secure=true` → `SameSite=None; Secure`，否则 `SameSite=Lax`；logout 删除时属性集与创建一致（gin 需手写 `Set-Cookie` 保证 SameSite 对齐）。

### D4. workspace 布局与迁移

- `paths.go` 提供 `ScopeFor(u CurrentUser) Scope` 与 `EnsureWorkspace(ctx)`（非 admin 惰性建 `knowledge_bases/`、`memory/` 等全树）；admin 恒为主 workspace，不建 `data/users/` 目录。
- pre-v1.5 迁移 `MigrateLegacyLayout(dataRoot string)`：`multi-user/_system` → `data/system`、`multi-user/<uid>` → `data/users/<uid>`；目标存在即跳过 + 告警；幂等，可在读路径无条件调用（与基线一致）；实现为 `os.Rename` 优先、跨设备回落复制。
- `data/system/` 由 `paths.go` 单点给出，沙箱 mount 组装处（`impl-tools-builtin` 的 exec 工具）以 deny-list 排除。

### D5. grants v2 与运行期过滤

- `grants.go`：读取时规范化任意历史 payload 为 v2（丢弃 `models.embedding`/`models.search`/`spaces`；缺省字段=不受限，但 `mcp_tools` 缺省=deny）；保存时拒绝 admin 用户 grant、递归扫描违规键（`api_key`/`secret`/`password`/`token`/`path`/`base_url`/`*_key`），报错含字段路径。
- `access.go` 暴露纯过滤函数供各域调用：`FilterModels(ctx, all)`、`FilterKBs(ctx, all)`、`AllowedTools(ctx, pool)`（enabled_tools 三元 + mcp_tools deny-by-default）、`FilterPartners(ctx, all)`、`ExecAllowed(ctx, deployPolicy)`（三态）。admin 直通，非 admin 按 grant。
- 审计：`audit.go` 以 JSONL 追加 `{actor, action, target, at}` 到 `data/system/audit/`，grants 写路径强制调用。

### D6. 与 Python 基线文件映射

| Python 基线 | Go 落位 |
| --- | --- |
| `deeptutor/multi_user/context.py` | `internal/multiuser/userctx/userctx.go`（ContextVar → 显式 context，D-005） |
| `deeptutor/multi_user/identity.py`、`models.py` | `internal/multiuser/identity.go` + `userctx` 类型 |
| `deeptutor/services/auth.py` | `internal/multiuser/token.go` + `guards.go` |
| `deeptutor/multi_user/router.py` | `internal/api/routers/multiuser.go` |
| `deeptutor/multi_user/paths.py` | `internal/multiuser/paths.go` |
| `deeptutor/multi_user/grants.py` | `internal/multiuser/grants.go` |
| `deeptutor/multi_user/{model,tool,knowledge,partner,skill}_access.py` | `internal/multiuser/access.go` |
| `deeptutor/multi_user/audit.py` | `internal/multiuser/audit.go` |
| `deeptutor/api/routers/auth.py` | `internal/api/routers/auth.go` + `avatar.go` |

## Risks / Trade-offs

- [bcrypt hash 变体不互认（passlib `$2b$` vs Go bcrypt）会锁死老用户] → `x/crypto/bcrypt` 原生支持 `$2a$/$2b$` 校验；对基线生成的真实 hash 做兼容单测（数据兼容验收项）。
- [显式 context 规约靠纪律，新代码可能漏传] → `userctx` 不提供任何「缺省 admin」API；`MustFrom` 错误显式化；lint/评审规则纳入 tasks；这正是 D-005 要买的保险，成本可接受。
- [gin 对 SameSite=None Cookie 的删除语义与浏览器差异] → logout 手写 `Set-Cookie` 头保证属性集与创建一致（spec 明确要求），加浏览器行为集成测试。
- [magic bytes 嗅探误判/绕过] → 只认 PNG/JPEG/WebP 前缀白名单、拒绝 SVG（存储型 XSS）；响应恒带 `nosniff` + `inline`，双层防御。
- [旧树迁移与并发启动竞态] → 迁移入口持文件锁 + 幂等（目标存在即跳过），可在读路径无条件调用。
- [grants 秘密字段黑名单可能漏键] → 递归校验 + `*_key` 通配兜底；grant 只存逻辑 id 的原则写入 API 文档与测试。
