# Tasks: impl-skills-persona

## 1. 解析与双层存储

- [ ] 1.1 `internal/skills/parser.go` + `model.go`：SKILL.md frontmatter 解析（name/description/tags/always/requires）、名称正则校验、解析失败空 meta 容错
- [ ] 1.2 `requires` 可用性门：`bins`（PATH 查找）、`env`、`sandbox` 检查与缺失项报告（`CLI: git` 格式）
- [ ] 1.3 内置技能打包：`assets/skills/builtin/`（docx/pdf/pptx/xlsx/skill-creator）go:embed + 启动物化
- [ ] 1.4 `internal/skills/service.go`：builtin/user 双层解析与遮蔽、builtin 写拒绝（read-only）、非法名目录忽略

## 2. manifest、read_skill 与 persona

- [ ] 2.1 `internal/skills/manifest.go`：manifest 行渲染（unavailable 追加缺失说明）、`always: true` Active Skills 全文块并从 manifest 排除、重名保留首个（user > admin > builtin）
- [ ] 2.2 `ReadSkillFile`：绝对路径/`..`/EvalSymlinks 越界三重拒绝、非文件 not found、100000 字符截断加 `[... truncated ...]` 标记、非 admin 经 admin 分配范围回退
- [ ] 2.3 `internal/persona/service.go`：PERSONA.md 解析、注入渲染（缺失/空正文注入空串永不失败）、遗留 persona 型技能自动迁出（幂等）

## 3. REST 面（gin route groups）

- [ ] 3.1 `/api/v1/skills`：`GET /list`（合并 admin 分配、含 source/read_only）、`GET /{name}`（grant 回退 / 403）、`POST /create`（409）、`PUT /{name}`（局部更新 + rename_to；builtin 403）、`DELETE /{name}`（builtin 403）；tag 路由先注册
- [ ] 3.2 tag 词表：`.tags.json` seed 初始化与 frontmatter 回填、`GET /tags/list`、`POST /tags/create`（409）、`PUT /tags/{tag}`（级联改写）、`DELETE /tags/{tag}`（frontmatter 移除）、tag 名正则校验
- [ ] 3.3 `/api/v1/personas`：`GET /list`（admin 预设 `source="admin"`、`read_only=true`、同名 user 优先）、`GET /{name}`（admin 回退）、`POST /create`（409）、`PUT /{name}`（改名）、`DELETE /{name}`

## 4. hub 导入

- [ ] 4.1 `internal/skills/hub.go`：`GET /hub/catalog`（代理 + `web_url`）、`GET /hub/detail`
- [ ] 4.2 `POST /install`：ref 解析（`<hub>:<slug>[@version]`）、staging 下载解包、安全门（suspicious 中止、符号链接中止、后缀白名单、1MB/20MB/500 文件上限、执行位剥离、`always` 剥离、`bins`/`env` 折叠）、原子换入、`.hub-lock.json` 溯源

## 5. 测试与验收（spec Scenario 落测试 + 协议对等）

- [ ] 5.1 将本 spec 全部 10 个 Scenario 落为 Go 测试：requires 缺失标记、user 遮蔽 builtin、always 全文注入、路径穿越拒绝、非 admin 分配技能读取/403、tag 改名级联、导入剥离 always、缺失 persona 不破坏轮次、admin 预设只读
- [ ] 5.2 `/api/v1/skills` 与 `/api/v1/personas` 全路由 contract test 对 OpenAPI golden spec 逐字段 diff（acceptance.md 3.1 A 类 + M2 矩阵「Settings/System/Personas/Tools API」项）
- [ ] 5.3 `read_skill`/`load_tools` 行为一致性验收（acceptance.md M2 矩阵「Skills」项，与 impl-tools-builtin 的工具面联测）
- [ ] 5.4 数据兼容：指向基线 `data/user/workspace/skills/` 与 `personas/` 目录列表/读取/编辑不改写既有格式（acceptance.md 3.2 B 类）
