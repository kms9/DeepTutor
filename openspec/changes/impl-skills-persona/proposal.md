# Change: impl-skills-persona

## Why

deeptutor-go 需要在 Go 侧复刻两套用户可编辑资产：技能（SKILL.md 能力包，模型按需取用）与 persona（PERSONA.md 语气预设，eager 注入 system prompt），否则 chat 的 Skills manifest 注入、`read_skill`/`load_tools` 工具与前端 Skills/Personas 设置页均无后端。本 change 依据模块行为规格 `docs/golang-req/openspec/specs/skills-persona/spec.md` 立项，对应 `docs/golang-req/openspec/ROADMAP.md` Wave 2「领域服务与工具」、里程碑 M2（acceptance.md M2 矩阵「Skills」项：SKILL.md 解析、`read_skill/load_tools` 行为一致）。

## What Changes

- SKILL.md 包结构与 YAML frontmatter 解析（`name`/`description`/`tags`/`always`/`requires` 可用性门），技能名正则校验，解析失败按空 meta 容错。
- builtin/user 双层存储与遮蔽：builtin 只读（`docx`/`pdf`/`pptx`/`xlsx`/`skill-creator`），user 层 `data/user/workspace/skills/` 可读写，同名 user 遮蔽 builtin。
- manifest 渲染与 `always: true` 全文 eager 注入（Active Skills 块），供 agent-loop 的 system prompt 组装消费。
- `read_skill` 服务侧文件读取：路径穿越拒绝、100000 字符截断、user 优先 builtin 兜底、非 admin 经 admin 分配范围回退。
- skills REST（`/api/v1/skills`）：list/detail/create/update（含 rename）/delete、builtin 403、tag 路由先于 `/{name}` 声明。
- tag 词表管理：`.tags.json` seed 初始化与回填、CRUD、改名级联改写技能 frontmatter。
- hub 浏览与导入安全门：catalog/detail/install、`suspicious` verdict 中止、后缀白名单与体量上限、符号链接中止、执行位剥离、`always` 剥离（防提示注入）、staging 原子换入、`.hub-lock.json` 溯源。
- persona 模型与注入：`PERSONA.md`、每轮最多一个、缺失不破坏轮次、遗留 persona 型技能自动迁出（幂等）。
- personas REST（`/api/v1/personas`）：list/get/create/update/delete，admin 预设对普通用户只读可见。

## Capabilities

### New Capabilities

- `skills-persona`: 技能与 persona 两套用户可编辑资产的 Go 侧行为契约——SKILL.md 解析、双层存储、manifest/always 注入、read_skill 服务侧、skills/personas REST、tag 词表、hub 导入安全门、persona 注入语义。

### Modified Capabilities

（无）

## Impact

- **依赖的其他 change**：
  - `impl-foundation-config`：workspace 路径服务（user/admin skills 与 personas 根目录解析）。
- **被依赖**：`impl-agent-loop`（manifest / Active Skills / persona 注入接口）与 `impl-tools-builtin`（`read_skill`/`load_tools` 工具面）。partner 的 `SOUL.md` 在 partners 模块，仅要求本模块 persona 注入接口可复用。
- **受影响代码**：新增 `internal/skills/`（parser/service/hub/taxonomy）、`internal/persona/`、`internal/api/` 下 skills 与 personas 路由组；打包内置 office 技能包目录。
- **前端影响**：Skills 页（列表/编辑/tag/hub 安装）、Personas 页全功能（acceptance.md M2 矩阵「Settings/System/Personas/Tools API」与「Skills」项）。
