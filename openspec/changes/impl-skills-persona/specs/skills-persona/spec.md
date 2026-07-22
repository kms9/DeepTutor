# skills-persona Delta Specification

## ADDED Requirements

### Requirement: SKILL.md 包结构与 frontmatter 解析
一个技能 SHALL 是独立目录 `<root>/<name>/SKILL.md`（可附 `references/` 等支持文件）。系统 SHALL 解析 YAML frontmatter：`name`、`description`（manifest 单行摘要）、`tags`（用户组织用）、`always`（true 时全文 eager 注入每轮 system prompt）、`requires`（可用性门：`bins` 检查 PATH 二进制、`env` 检查环境变量、`sandbox` 检查沙箱能力，任一缺失则技能标记 unavailable 并列出缺失项）。技能名 SHALL 匹配 `^[a-z0-9][a-z0-9-]{0,63}$`；frontmatter 解析失败按空 meta 处理不抛错。

#### Scenario: requires 缺失时标记不可用
- **WHEN** 某技能声明 `requires: {bins: [git]}` 而服务器 PATH 无 `git`
- **THEN** manifest 中该技能标记 unavailable 且缺失项为 `CLI: git`
- **AND** 技能仍出现在列表中（供用户了解如何启用）

### Requirement: builtin/user 双层存储与遮蔽
系统 SHALL 维护两层技能：builtin 层随产品打包只读（内置 office 技能包目录：`docx`、`pdf`、`pptx`、`xlsx`、`skill-creator`），user 层位于 `data/user/workspace/skills/` 可读写；同名时 user 层遮蔽 builtin 层。写操作（update/delete）SHALL 拒绝作用于 builtin 技能（read-only 错误）；user 目录下的非法名目录 SHALL 被列表忽略。

#### Scenario: 用户遮蔽内置技能
- **WHEN** 用户创建名为 `pdf` 的自定义技能
- **THEN** 列表与 `read_skill` 解析均命中 user 层版本
- **AND** 删除该 user 技能后 builtin 版本重新可见

### Requirement: manifest 渲染与 always 注入
系统 SHALL 从可见技能集合生成 manifest 行（`- **<name>** — <description>`，unavailable 追加缺失说明），供 system prompt 的 Skills 区注入；`always: true` 且 available 的技能 SHALL 改为全文 eager 注入（Active Skills 块），并从 manifest 行中排除。重名条目保留首个（层级顺序 user > admin > builtin）。

#### Scenario: always 技能全文注入
- **WHEN** 某可用技能 frontmatter 含 `always: true`
- **THEN** 其正文（去 frontmatter）进入每轮 system prompt 的 Active Skills 块
- **AND** Skills manifest 不再列出该技能

### Requirement: read_skill 文件读取（服务侧）
skill service SHALL 提供包内文件读取：`rel_path` 严格限定在技能目录内（拒绝绝对路径、`..` 段与 resolve 后越界，报非法路径错误）；目标非文件时报 not found；内容超过 100000 字符时截断并追加显式 `[... truncated ...]` 标记。解析顺序为 user 层优先、builtin 兜底；非 admin 用户对 admin 分配技能（grant 名单内）SHALL 可经 admin workspace 根回退读取。

#### Scenario: 路径穿越被拒
- **WHEN** `read_skill` 请求 `file="../../secrets.txt"`
- **THEN** 服务返回非法路径错误，不发生任何目录外读取

### Requirement: skills REST 面
系统 SHALL 提供 `/api/v1/skills` 路由组：`GET /list`（自有技能 + 非 admin 用户合并 admin 分配技能，条目含 `name/description/tags/source/read_only`）、`GET /{name}`（detail 含全文；user 层未命中时 admin 分配范围回退，无 grant 返回 403）、`POST /create`（校验名称与 tags，frontmatter 归一化写入；重名 409）、`PUT /{name}`（支持 description/content/tags 局部更新与 `rename_to` 改名；builtin 403）、`DELETE /{name}`（builtin 403）。tag 路由 SHALL 先于 `/{name}` 声明以免被遮蔽。

#### Scenario: 非 admin 读取分配技能
- **WHEN** 非 admin 用户 GET 一个不在自己 workspace、但在 admin 分配名单内的技能
- **THEN** 返回 admin 层的技能 detail
- **AND** 名单外的技能返回 403

### Requirement: tag 词表管理
系统 SHALL 在 user 技能根维护 `.tags.json` 规范词表（默认 seed `style`、`tool`，首次访问自动初始化并回填技能 frontmatter 中出现的 tags）。`GET /tags/list`、`POST /tags/create`（重名 409）、`PUT /tags/{tag}`（改名并级联改写全部使用该 tag 的 user 技能 frontmatter）、`DELETE /tags/{tag}`（删除并从技能 frontmatter 中移除）。tag 名 SHALL 匹配 `^[a-z0-9][a-z0-9- _]{0,31}$`。

#### Scenario: tag 改名级联
- **WHEN** 将 tag `style` 改名为 `voice` 且三个 user 技能带有 `style`
- **THEN** 词表更新且三个技能的 frontmatter tags 均改写为 `voice`

### Requirement: hub 浏览与导入安全门
系统 SHALL 支持技能 hub（默认 `eduhub`）：`GET /hub/catalog`（代理目录检索，返回 `web_url` 供跳出链接）、`GET /hub/detail`（单技能元数据 + SKILL.md 正文）、`POST /install`（按 `<hub>:<slug>[@version]` ref 安装到调用者 user 层，返回安装结果 + 安全 verdict + 版本）。导入 SHALL 执行安全门：`suspicious` verdict 中止；支持文件按后缀白名单（文本/脚本/配置类）+ 单文件 1MB / 总量 20MB / 500 文件上限过滤，越限跳过或整体失败；符号链接直接中止导入；拷贝不保留执行位；frontmatter 适配时 `always` 字段 SHALL 被剥离（防提示注入），平铺 `bins`/`env` 折叠进 `requires.*`；staging 目录原子换入避免半成品；来源记入 `.hub-lock.json` 供 UI 溯源。

#### Scenario: 导入包剥离 always
- **WHEN** 从 hub 安装的技能包 frontmatter 含 `always: true`
- **THEN** 安装后的 SKILL.md frontmatter 不含 `always`
- **AND** 该技能只能经 manifest + `read_skill` 被使用

### Requirement: persona 模型与注入
一个 persona SHALL 是 `data/user/workspace/personas/<name>/PERSONA.md`（frontmatter：`name`、`description`）。persona 是语气/行为预设而非能力包：被选中的 persona SHALL 将正文 eager 注入 system prompt（带"整轮扮演该 persona"的引导语），每轮最多激活一个；persona 不存在或正文为空时注入空串，SHALL NOT 使轮次失败。基线遗留的 persona 型技能 SHALL 在首次访问 persona service 时自动从 skills 目录迁出（幂等）。

#### Scenario: 缺失 persona 不破坏轮次
- **WHEN** 会话引用的 persona 已被删除
- **THEN** 该轮 system prompt 不含 persona 块且轮次正常执行

### Requirement: personas REST 面
系统 SHALL 提供 `/api/v1/personas` 路由组：`GET /list`（自有 persona + 非 admin 用户合并 admin 预设，预设标记 `source="admin"`、`read_only=true`，同名时 user 层优先）、`GET /{name}`（user 未命中且非 admin 时回退 admin 预设）、`POST /create`（重名 409）、`PUT /{name}`（支持改名）、`DELETE /{name}`。admin persona 对所有用户可见且无 grant 机制（persona 不携带特权工作流）。

#### Scenario: admin 预设对普通用户只读
- **WHEN** 非 admin 用户尝试 `PUT` 一个仅存在于 admin 层的 persona
- **THEN** 请求以 not found 语义失败（用户层无此 persona 可写）
- **AND** `GET /list` 中该 persona 标记 `read_only=true`
