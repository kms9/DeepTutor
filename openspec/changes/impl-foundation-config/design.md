# Design: impl-foundation-config

## Context

Python 基线用 `deeptutor/services/config/runtime_settings.py`（约 2k 行的 settings 读写/归一化/env override）+ `deeptutor/services/path_service.py`（`data/` 布局）+ `deeptutor/services/file_io.py`（`atomic_write_json`）支撑全部运行时配置。Go 版必须逐行为对等地复刻这三者，且能直接打开 Python v1.5.2 生成的 `data/` 目录（免迁移，acceptance.md §3.2 B 类）。约束：配置事实源是 `data/user/settings/*.json`，项目根 `.env` 有意忽略；技术选型固定为 `spf13/viper`（project.md「Tech Stack」）。

## Goals / Non-Goals

**Goals:**

- settings load-or-create、归一化、原子写、损坏容错，与基线逐字段对等。
- env override 语义完整：仅内存生效、self-echo 抑制、`DEEPTUTOR_IGNORE_PROCESS_ENV_OVERRIDES` 全局开关。
- `PathService` 覆盖基线全部路径 API、幂等目录创建、legacy memory 迁移、多用户 root 实例化。
- `is_public_output_path` 判定逐规则对等（供 `/api/outputs/` 使用）。

**Non-Goals:**

- `llm_configs.json`、`embedding_configs.json`、`model_catalog.json` 的语义（归 llm-provider / settings-api 模块）。
- Settings REST API（归 settings-api 模块）；本模块只提供进程内 API。
- 配置热更新的消费方行为（本模块提供 WatchConfig 通知，消费策略由各下游模块决定）。

## Decisions

### D1. Go 包结构（单包 `internal/config`）

```
deeptutor-go/internal/config/
├── paths.go        # PathService（目录布局、ensure、多用户实例化）
├── publicpath.go   # is_public_output_path 判定
├── settings.go     # SettingsStore：load-or-create、原子写、损坏容错
├── system.go       # SystemSettings 默认值 + 归一化/钳制
├── auth.go         # AuthSettings
├── integrations.go # IntegrationsSettings（MinerU/PageIndex 等 slice）
├── parsing.go      # document_parsing v2 结构与 legacy 迁移
├── envoverride.go  # env override 表 + self-echo 抑制 + export/render_environment
├── atomicio.go     # 原子写（temp + rename），对应基线 file_io.atomic_write_json
└── *_test.go
```

理由：配置层是叶子依赖、无内部分层必要；与 project.md 目录约定 `internal/config/ # runtime settings` 一致。备选（拆 `internal/paths` 独立包）被否：`is_public_output_path`、settings 路径解析与 PathService 强耦合，拆包只增加导入噪音。

### D2. 关键接口签名

```go
package config

// PathService —— 对应基线 services/path_service.py
type PathService struct{ root string } // root 默认 "data"
func NewPathService(root string) *PathService
func (p *PathService) ForUser(uid string) *PathService     // data/users/<uid>/ 实例化
func (p *PathService) UserDataDir() string                 // data/user/
func (p *PathService) SettingsFile(name string) string     // 无扩展名补 .json
func (p *PathService) RuntimeConfigFile(name string) string// 非 .yaml 结尾补 .yaml
func (p *PathService) ChatHistoryDB() string               // data/user/chat_history.db
func (p *PathService) KnowledgeBasesDir() string
func (p *PathService) ParseCacheDir() string
func (p *PathService) MemoryDir() string                   // data/memory/（含 legacy 单向复制）
func (p *PathService) WorkspaceFeatureDir(feature string) string
func (p *PathService) EnsureAllDirectories() error         // 幂等
func (p *PathService) IsPublicOutputPath(path string) bool

// SettingsStore —— 对应基线 runtime_settings.py
type SettingsStore struct { paths *PathService; /* viper 实例表、exported self-echo 表 */ }
func NewSettingsStore(paths *PathService) *SettingsStore
func (s *SettingsStore) LoadSystem() SystemSettings        // load-or-create + 归一化 + env override
func (s *SettingsStore) SaveSystem(v SystemSettings) error // 归一化后原子写
func (s *SettingsStore) LoadAuth() AuthSettings
func (s *SettingsStore) LoadIntegrations() IntegrationsSettings
func (s *SettingsStore) LoadDocumentParsing() DocumentParsingSettings // 含 v1→v2 迁移
func (s *SettingsStore) ExportEnvironment()                // 导出并登记 self-echo 表
func (s *SettingsStore) RenderEnvironment() map[string]string
```

### D3. viper 用法与 env override 的 self-echo 抑制

- 每个 settings 文件一个 viper 实例：`v := viper.New(); v.SetConfigFile(paths.SettingsFile(name)); v.SetConfigType("json")`；`WatchConfig` + `OnConfigChange` 提供热更新通知。
- **不使用** viper 的 `AutomaticEnv`/`BindEnv` 做 override。理由：基线的 override 是「显式键表 + self-echo 抑制 + 全局禁用开关」，viper 的自动绑定无法表达 self-echo 语义。改为显式三层合并：`defaults → 磁盘 JSON（viper 读出）→ normalize → applyEnvOverrides()`，override 只改内存返回值，从不写回 viper 也不写盘。
- self-echo 抑制实现：`ExportEnvironment` 在 `os.Setenv` 的同时把 `{键: 导出值}` 记入进程内 `exportedEnv map[string]string`；`applyEnvOverrides` 对每个 override 键判定——环境值存在，且（键不在 `exportedEnv`，或环境值 ≠ 登记值）时才生效。与基线 `runtime_settings.py` 的抑制语义逐条对应（spec「进程环境变量 override 且忽略 .env」）。
- 忽略 `.env`：不引入 godotenv 类依赖，启动路径上不存在任何 `.env` 读取代码，天然满足。

### D4. 原子写与损坏容错

`atomicio.go`：`os.CreateTemp(dir, name+".*.tmp")` → 写入 + `Sync` → `os.Rename`（同目录内 rename 在 POSIX 上原子）。加载时 `json.Unmarshal` 失败或顶层非 object 一律按空文件对待：回落默认值、归一化、写回，不返回错误（对应基线「损坏 JSON 不中断启动」）。

### D5. 归一化引擎

归一化以 `map[string]any` 为中间形态（与基线 dict 处理一致），每类设置提供 `normalizeXxx(raw map[string]any) map[string]any`；归一化结果与磁盘 JSON 做规范化比较（`json.Marshal` 后字节比较，键排序），不同才写回，避免每次启动都触发磁盘写。钳制规则（端口 1–65535、附件限额区间与抬升、truthy/falsy 集合）逐条照抄 spec。

### D6. document_parsing v1→v2 迁移

加载顺序：`document_parsing.json` 不存在且 `mineru.json` 存在 → `os.Rename` 原地重命名；读出后若顶层无 `"version": 2` 且含 MinerU 键（v1 扁平结构）→ 包装为 `{"version":2,"engine":"mineru","engines":{"mineru":<原内容>}}` 写回。枚举校验（engine/mode/model_version/model_download_source）越界回落默认。

### D7. 与 Python 基线的映射表

| Python 基线 | Go 落位 |
| --- | --- |
| `deeptutor/services/config/runtime_settings.py` `load_system`/`save_system` 等 | `internal/config/settings.go` + `system.go`/`auth.go`/`integrations.go` |
| `runtime_settings.py` env override 键表与 self-echo 抑制 | `internal/config/envoverride.go` |
| `runtime_settings.py` `export_environment`/`render_environment` | `envoverride.go` `ExportEnvironment`/`RenderEnvironment` |
| `runtime_settings.py` document_parsing v2 + mineru 迁移 | `internal/config/parsing.go` |
| `deeptutor/services/path_service.py` | `internal/config/paths.go` |
| `path_service.py` `is_public_output_path` | `internal/config/publicpath.go` |
| `deeptutor/services/file_io.py` `atomic_write_json` | `internal/config/atomicio.go` |

## Risks / Trade-offs

- [基线归一化细节遗漏（字段多、分支密）] → 实现时对照基线源码逐字段核对；每个 spec Scenario 落为表驱动测试；用 Python v1.5.2 生成的真实 `data/user/settings/` 快照做数据兼容测试（acceptance.md §3.2）。
- [viper 的 JSON 写回可能改动键序/缩进，造成与 Python 写出文件的无谓 diff] → 写回不经 viper，统一走 `atomicio.go` 自己 `json.MarshalIndent`；viper 仅用于读取与 WatchConfig。
- [self-echo 表是进程内状态，多实例 SettingsStore 会失配] → SettingsStore 在进程内以单例使用（由组装根注入），文档化该约束。
- [WatchConfig 在部分文件系统上事件抖动] → 通知仅作缓存失效信号，加载路径始终可全量重读，抖动无正确性影响。

本模块无登记的有意差异（acceptance.md §6 D-001~D-008 均不涉及 foundation-config），行为以与基线完全对齐为准。
