# Design: impl-skills-persona

## Context

基线实现位于 `deeptutor/services/skill/`（`service.py`/`hub.py`/`taxonomy.py`）、`deeptutor/services/persona/service.py`、内置技能包 `deeptutor/skills/builtin/`、REST 在 `api/routers/skills.py` 与 `personas.py`。全部数据为文件（SKILL.md / PERSONA.md / `.tags.json` / `.hub-lock.json`）。Go 侧无新增存储依赖，核心是文件解析、路径安全与 REST 面对等；manifest/persona 注入以纯函数接口供 agent-loop 消费。

## Goals / Non-Goals

**Goals:**

- `internal/skills/` 与 `internal/persona/`：解析、双层存储遮蔽、manifest/always 渲染、read_skill 服务侧、tag 词表、hub 导入安全门。
- `/api/v1/skills` 与 `/api/v1/personas` REST 与基线逐字段对等。
- 内置 office 技能包（`docx`/`pdf`/`pptx`/`xlsx`/`skill-creator`）随 Go 产物打包。

**Non-Goals:**

- `read_skill`/`load_tools` 的 LLM schema 与工具面（tools-builtin）；system prompt 组装时机（agent-loop）；partner `SOUL.md`（partners 模块，仅要求注入接口可复用）；admin grant 的管理面（multi-user-auth，本模块只消费 grant 名单查询接口）。

## Decisions

### D1. Go 包结构

```
internal/skills/
├── model.go        # Skill / Meta / Requires / Availability
├── parser.go       # SKILL.md frontmatter 解析（YAML，容错为空 meta）
├── service.go      # 双层解析、list/get/create/update/delete、ReadSkillFile（对应 service.py）
├── manifest.go     # manifest 行渲染 + Active Skills（always）块
├── taxonomy.go     # .tags.json 词表 CRUD + 级联改写（对应 taxonomy.py）
└── hub.go          # hub catalog/detail/install + 安全门（对应 hub.py）

internal/persona/
└── service.go      # PERSONA.md 解析、注入渲染、遗留技能迁出（对应 persona/service.py）

internal/api/skillsapi/router.go    # gin group /api/v1/skills（tag 路由先注册）
internal/api/personasapi/router.go  # gin group /api/v1/personas

assets/skills/builtin/   # 打包内置技能（go:embed 或安装目录，见 D4）
```

### D2. 关键接口签名

```go
// internal/skills/service.go
type Service struct{ ... } // 依赖 config.PathResolver（user/admin 根）+ grant 查询接口

type Skill struct {
    Name, Description string
    Tags              []string
    Always            bool
    Requires          Requires        // bins/env/sandbox
    Source            string          // "user" | "admin" | "builtin"
    ReadOnly          bool
    Unavailable       []string        // 缺失项，如 "CLI: git"
}

func (s *Service) List(ctx context.Context, u auth.UserContext) ([]Skill, error)
func (s *Service) Get(ctx context.Context, u auth.UserContext, name string) (*SkillDetail, error) // 无 grant → ErrForbidden
func (s *Service) Create(ctx context.Context, u auth.UserContext, req CreateReq) error            // 重名 ErrConflict
func (s *Service) Update(ctx context.Context, u auth.UserContext, name string, req UpdateReq) error // builtin ErrReadOnly
func (s *Service) Delete(ctx context.Context, u auth.UserContext, name string) error

// read_skill 服务侧（tools-builtin 消费）
func (s *Service) ReadSkillFile(ctx context.Context, u auth.UserContext, name, relPath string) (string, error)
// 路径安全：filepath.IsAbs 拒绝 → filepath.Clean 后含 ".." 拒绝 → EvalSymlinks 后 HasPrefix(skillDir) 校验

// agent-loop 消费的注入接口
func (s *Service) RenderManifest(ctx context.Context, u auth.UserContext) (manifest string, activeSkills string, err error)

// internal/persona/service.go
func (p *Service) RenderInjection(ctx context.Context, u auth.UserContext, name string) string // 缺失/空 → ""，永不 error
```

### D3. hub 导入安全门（hub.go）

安装流水线（对应基线 `hub.py`）：下载包 → 解包到 `os.MkdirTemp` staging → 安全门逐项检查 → 通过后 `os.Rename` 原子换入 user 层 → 写 `.hub-lock.json`。安全门顺序：

1. verdict 为 `suspicious` → 中止；
2. 遍历时发现符号链接（`d.Type()&fs.ModeSymlink`）→ 整体中止；
3. 后缀白名单（文本/脚本/配置类）过滤，单文件 >1MB 跳过，总量 >20MB 或 >500 文件整体失败；
4. 拷贝以 `0644` 写入（不保留执行位）；
5. frontmatter 适配：删除 `always` 键（防提示注入），平铺 `bins`/`env` 折叠进 `requires.*`。

### D4. builtin 技能打包

内置技能包用 `go:embed assets/skills/builtin` 嵌入二进制，启动时物化到只读缓存目录供统一的文件读取路径使用。备选「安装目录随包分发」——被否：单二进制分发（D-006 的 Go 二进制形态）下 embed 免除路径发现问题。

### D5. 与 Python 基线文件映射

| Python 基线 | Go 落位 |
| --- | --- |
| `services/skill/service.py` | `internal/skills/{service,parser,manifest,model}.go` |
| `services/skill/taxonomy.py` | `internal/skills/taxonomy.go` |
| `services/skill/hub.py` | `internal/skills/hub.go` |
| `services/persona/service.py` | `internal/persona/service.go` |
| `deeptutor/skills/builtin/`（docx/pdf/pptx/xlsx/skill-creator） | `assets/skills/builtin/`（go:embed） |
| `api/routers/skills.py` / `personas.py` | `internal/api/skillsapi/router.go` / `personasapi/router.go` |

### D6. gin 路由声明顺序

gin 对 `/tags/list` 与 `/:name` 存在通配冲突风险：与基线一致，tag 路由组先注册（spec 明确要求「tag 路由先于 `/{name}` 声明」）；gin 的 route tree 对静态段优先，但为避免 `:name` 与 `tags` 段冲突（gin panic），采用 `/tags/...` 静态前缀组 + `/:name` 通配注册顺序控制，contract test 覆盖。

## Risks / Trade-offs

- [YAML frontmatter 解析差异（gopkg.in/yaml.v3 与 PyYAML 边缘行为）] → 解析失败按空 meta 容错（spec 要求）；用基线技能包做解析 golden 对比。
- [gin 通配路由与静态路由冲突导致注册 panic] → 路由注册顺序 + 集成测试覆盖全部路径组合。
- [hub 远端不可用] → catalog/detail 代理错误映射为明确的上游错误响应，不影响本地技能功能。
- [路径穿越绕过（symlink 指向目录外）] → `EvalSymlinks` 后再做前缀校验，而非只 Clean 字符串。
