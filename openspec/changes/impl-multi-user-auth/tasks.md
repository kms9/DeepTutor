## 1. 用户上下文与身份基础（横切前提，先行）

- [ ] 1.1 实现 `internal/multiuser/userctx/`：`CurrentUser{id, username, role, scope}`、`With`/`From`/`MustFrom`（不提供缺省 admin 兜底，D-005）
- [ ] 1.2 实现 `identity.go`：`auth_users.json` 记录格式读写（持锁原子写）、`u_<uuid hex>` id 生成、`^[A-Za-z0-9_-]{1,64}$` user id 校验、旧平面格式迁移、`auth.json` bootstrap 用户合并
- [ ] 1.3 实现 `token.go`：HS256 JWT 签发/校验（payload `sub/role/uid/exp/iat`）、`auth_secret` 首启生成并持久化 `data/system/auth/`、bcrypt 校验（兼容基线 `$2b$` hash）
- [ ] 1.4 实现 `paths.go`：admin/user/partner scope 解析、非 admin 惰性建全树（含 `knowledge_bases/`、`memory/`）、admin 不建 `data/users/`、`data/system/` 单点定义

## 2. 守卫与单用户模式

- [ ] 2.1 实现 `guards.go` `RequireAuth`：`enabled=false` 注入合成 local admin；`enabled=true` Bearer 头 + `dt_token` Cookie 双提取、401 + `WWW-Authenticate: Bearer`、成功注入 CurrentUser context
- [ ] 2.2 实现 `RequireAdmin`（403 `Admin access required`）与 `AuthenticateWS`（`?token=`/Cookie、upgrade 前认证、失败 4001），保证 HTTP/WS 语义对称
- [ ] 2.3 实现单用户模式端点语义：`/login` 返回「无需登录」、`/register` 与 profile 端点 400、`/status` 返回 local-admin 身份

## 3. auth REST 面

- [ ] 3.1 实现公开端点：`GET /status`、`POST /login`（Cookie 属性随 `cookie_secure`、响应体 `{ok, user_id, username, role, is_admin}`）、`POST /logout`（删 Cookie 属性集与创建一致）、`POST /register`（首用户即 admin、之后 403）、`GET /is_first_user`
- [ ] 3.2 实现 username/口令校验规则（email 或 `[A-Za-z0-9_\-.]{3,64}`、口令 ≥8 字符）
- [ ] 3.3 实现自助端点：`GET /profile`、`PUT /profile`（仅空串或 `icon:<name>:<color>` marker，`img:` 由上传独占）
- [ ] 3.4 实现 `avatar.go` 与头像端点：`PUT /profile/avatar`（≤1MB、magic bytes 仅 PNG/JPEG/WebP、SVG 415、`img:<version>` 递增）、`DELETE /profile/avatar`、`GET /avatar/{user_id}`（`nosniff` + `inline`）
- [ ] 3.5 实现 admin 端点：`GET /users`（不含 hash）、`POST /users`（恒 role=user）、`DELETE /users/{username}`（禁删自己 400、连删头像）、`PUT /users/{username}/role`（禁改自己 400）

## 4. workspace 隔离与迁移

- [ ] 4.1 实现非 admin 读写落入 `data/users/<uid>/` 的路径解析接线（KB/memory/notebook 等服务经 `userctx` + `paths.go` 寻址）
- [ ] 4.2 实现 `MigrateLegacyLayout`：`multi-user/_system`→`data/system`、`multi-user/<uid>`→`data/users/<uid>`、目标存在跳过并告警、幂等、空旧根删除、文件锁防并发
- [ ] 4.3 确认 `data/system/` 不入沙箱 mount（与 exec 工具的 mount 组装处对接并加断言测试）

## 5. grants 与管理面

- [ ] 5.1 实现 `grants.go`：v2 shape 读写、历史 payload 规范化（丢弃 `models.embedding`/`models.search`/`spaces`）、拒绝 admin grant、秘密/路径字段递归校验（含 `*_key`，报错带字段路径）
- [ ] 5.2 实现 `access.go` 运行期过滤：`FilterModels`/`FilterKBs`/`AllowedTools`（enabled_tools 三元、`mcp_tools=null` deny-by-default）/`FilterPartners`/`ExecAllowed`（三态）
- [ ] 5.3 实现 `internal/api/routers/multiuser.go`（admin 门控）：`GET /resources`、`GET/PUT /users/{user_id}/grants`、skill 安装端点
- [ ] 5.4 实现 `audit.go` 审计日志（`data/system/audit/` JSONL：actor/action/target/timestamp），grants 写路径强制留痕

## 6. 测试与验收（spec Scenario 落测试）

- [ ] 6.1 开关与登录 Scenario 测试：enabled=false 全 local admin（/status 字段断言）、错误口令 401 不设 Cookie、WS 无效 token 不 accept 直接 4001
- [ ] 6.2 身份存储 Scenario 测试：首注册者 admin + 二次注册 403、旧平面格式迁移、基线 bcrypt hash 互认、并发注册持锁
- [ ] 6.3 REST 面 Scenario 测试：SVG 头像 415、admin 禁删自己 400、profile marker 规则、logout Cookie 属性对齐
- [ ] 6.4 上下文 Scenario 测试（D-005）：服务层无 CurrentUser 调用显式失败不落 admin；WS 连接期上下文持续有效；加 lint 规则禁止服务层 `context.Background()`
- [ ] 6.5 workspace Scenario 测试：非 admin 读写全落自己 workspace（admin workspace 零写入断言）、pre-v1.5 树迁移幂等
- [ ] 6.6 grants Scenario 测试：夹带 `api_key` 拒绝且磁盘不变、`mcp_tools=null` deny / 授名后仅名单可用、普通用户访问 `/api/v1/partners` 403、授权变更审计留痕
- [ ] 6.7 协议对等验收：`/api/v1/auth`（14 条）与 `/api/v1/multi-user` 全路由 golden spec contract test；acceptance.md M4 矩阵「登录、workspace 隔离、管理员资源分配与基线一致」逐项通过
