# Delta Spec: foundation-config

> 事实源：`docs/golang-req/openspec/specs/foundation-config/spec.md`（Requirement/Scenario 原样搬运，未增删语义）。

## ADDED Requirements

### Requirement: settings 文件按名 load-or-create 并归一化
系统 SHALL 以 `data/user/settings/<name>.json` 为每类设置的持久化文件（`name` 不含扩展名时自动补 `.json`）。加载时 SHALL 执行 load-or-create 语义：文件不存在或为空时以默认值创建；文件存在时以 `{**defaults, **loaded}` 合并后做归一化（normalize），若归一化结果与磁盘内容不同 SHALL 原子写回。文件内容损坏（非 JSON 或非 object）时 SHALL 按空文件对待、回落默认值，SHALL NOT 抛错中断启动。所有写入 SHALL 使用原子写（临时文件 + rename）。

基线已知设置文件族至少包括：`system.json`、`auth.json`、`integrations.json`、`document_parsing.json`、`pageindex.json`、`llamaindex.json`、`graphrag.json`、`lightrag.json`（另有 `llm_configs.json`、`embedding_configs.json`、`model_catalog.json` 等归其他模块管理）。

#### Scenario: 首次启动创建默认 system.json
- **WHEN** `data/user/settings/system.json` 不存在时调用 `load_system`
- **THEN** 返回默认结构（`version=1`、`backend_port=8001`、`frontend_port=3782`、`sandbox_allow_subprocess=true` 等）
- **AND** 该默认结构被原子写入磁盘

#### Scenario: 损坏的 JSON 不中断加载
- **WHEN** `system.json` 内容为非法 JSON
- **THEN** 加载返回归一化后的默认值并写回，进程不崩溃

### Requirement: system.json 字段归一化与钳制
系统 SHALL 对 `system.json` 做字段级归一化：`backend_port`/`frontend_port` 必须在 1–65535 之间否则回落默认；布尔字段接受 truthy 集合 `{"1","true","yes","on"}` 与 falsy 集合 `{"0","false","no","off"}`（大小写不敏感），无法识别时用默认值；`chat_attachment_max_file_mb` 钳制到 [1,1024]、`chat_attachment_max_total_mb` 钳制到 [1,2048] 且 SHALL 抬升到不小于 `chat_attachment_max_file_mb`；`chat_attachment_max_chars_per_doc`/`chat_attachment_max_chars_total` 钳制到 [10000,5000000]。`cors_origins` SHALL 归一化为字符串数组。

#### Scenario: 附件总量小于单文件上限时被抬升
- **WHEN** 保存 `chat_attachment_max_file_mb=100`、`chat_attachment_max_total_mb=25`
- **THEN** 归一化后 `chat_attachment_max_total_mb` 为 100

#### Scenario: 非法端口回落默认
- **WHEN** `backend_port` 为 `0` 或 `"abc"`
- **THEN** 归一化后 `backend_port` 为 8001

### Requirement: 进程环境变量 override 且忽略 .env
系统 SHALL 在加载设置时按需叠加进程环境变量 override（仅内存生效，SHALL NOT 持久化回 JSON），并 SHALL NOT 读取项目根目录的 `.env` 文件。基线 override 键至少包括：system 类的 `BACKEND_PORT`、`FRONTEND_PORT`、`NEXT_PUBLIC_API_BASE_EXTERNAL`、`PUBLIC_API_BASE`、`NEXT_PUBLIC_API_BASE`、`CORS_ORIGIN`、`CORS_ORIGINS`、`DISABLE_SSL_VERIFY`、`CHAT_ATTACHMENT_DIR`、`DEEPTUTOR_SANDBOX_ALLOW_SUBPROCESS`、`CHAT_ATTACHMENT_MAX_*` 四项；auth 类的 `AUTH_ENABLED`（或 `NEXT_PUBLIC_AUTH_ENABLED`）、`AUTH_USERNAME`、`AUTH_PASSWORD_HASH`、`AUTH_TOKEN_EXPIRE_HOURS`、`AUTH_COOKIE_SECURE`；MinerU 类的 `MINERU_MODE`、`MINERU_API_BASE_URL`、`MINERU_API_TOKEN` 等；PageIndex 类的 `PAGEINDEX_API_KEY`、`PAGEINDEX_API_BASE_URL`。设置环境变量 `DEEPTUTOR_IGNORE_PROCESS_ENV_OVERRIDES` 为 truthy 时 SHALL 全局禁用 override。

系统自身通过 `export_environment` 导出到进程环境的值 SHALL NOT 在后续加载中被误当作外部 override 回读（self-echo 抑制）：仅外部预先存在、或与内部导出值不同的环境变量才生效。

#### Scenario: 环境变量覆盖端口但不写盘
- **WHEN** 进程环境有 `BACKEND_PORT=9001` 且 `system.json` 中为 8001
- **THEN** `load_system` 返回 `backend_port=9001`
- **AND** 磁盘上 `system.json` 仍为 8001

#### Scenario: 自导出的环境变量不构成 override
- **WHEN** 系统先调用 `export_environment` 导出 `BACKEND_PORT=8001`，随后用户把 `system.json` 改为 8002
- **THEN** 再次 `load_system` 返回 8002，而非被自己导出的 8001 覆盖

### Requirement: document_parsing 多引擎结构与 legacy 迁移
系统 SHALL 以 `document_parsing.json` v2 结构持久化解析引擎配置：`{"version":2, "engine":<active>, "engines":{...}}`，引擎枚举为 `text_only`、`mineru`、`docling`、`markitdown`、`pymupdf4llm`，未知 engine 名 SHALL 回落默认 `text_only`。兼容迁移：存在 legacy `mineru.json` 且 `document_parsing.json` 不存在时 SHALL 原地重命名；若磁盘内容为 v1 扁平 MinerU 结构（顶层含 MinerU 键），SHALL 迁移进 `engines.mineru` 并把 active engine 钉为 `mineru`（保持既有安装行为）。MinerU slice 的 `mode` 枚举为 `local`/`cloud`，`model_version` 枚举为 `pipeline`/`vlm`，`model_download_source` 枚举为 `huggingface`/`modelscope`。

#### Scenario: v1 mineru.json 免手工迁移
- **WHEN** 目录中只有 v1 扁平结构的 `mineru.json`
- **THEN** 首次加载后磁盘上出现 v2 `document_parsing.json`，其 `engine` 为 `mineru` 且原有 MinerU 配置保留在 `engines.mineru` 中

### Requirement: 渲染子进程环境变量集合
系统 SHALL 提供 `render_environment` 等价能力：把 system/auth/integrations 设置渲染为子进程（前端 Node 进程、Docker entrypoint）所需的环境变量名值表，键集合与基线一致（`BACKEND_PORT`、`FRONTEND_PORT`、`NEXT_PUBLIC_API_BASE*`、`CORS_ORIGIN(S)`、`DISABLE_SSL_VERIFY`、`CHAT_ATTACHMENT_DIR`、`DEEPTUTOR_SANDBOX_ALLOW_SUBPROCESS`、`AUTH_*`、`NEXT_PUBLIC_AUTH_ENABLED`、`DEEPTUTOR_API_BASE_URL`、`DEEPTUTOR_AUTH_ENABLED`、`POCKETBASE_*`）。其中 `DEEPTUTOR_API_BASE_URL` 的解析顺序 SHALL 为 `next_public_api_base` → `next_public_api_base_external` → `http://localhost:<backend_port>`。

#### Scenario: 默认配置下的 API base 推导
- **WHEN** `next_public_api_base` 与 `next_public_api_base_external` 均为空、`backend_port=8001`
- **THEN** 渲染出的 `DEEPTUTOR_API_BASE_URL` 为 `http://localhost:8001`

### Requirement: PathService 目录布局兼容
系统 SHALL 复用基线 `data/` 目录布局，路径 API 的语义与磁盘位置逐一对等：

- workspace root（默认 `data/`）下：`knowledge_bases/`、`parse_cache/`、`memory/`、`user/`
- `data/user/` 下：`chat_history.db`、`logs/`、`settings/`、`workspace/`
- `data/user/workspace/` 下：`memory/`（legacy）、`notebook/`、`co-writer/`、`book/`、`chat/`
- `data/user/workspace/chat/` 下的 feature 子目录：`chat`、`deep_solve`、`deep_question`、`deep_research`、`math_animator`、`_detached_code_execution`
- 多用户模式下以 `data/users/<uid>/` 为 workspace root 实例化同一套 API，公共 API 不变

`ensure_all_directories` 等价能力 SHALL 幂等创建上述目录。memory 目录 SHALL 使用 workspace root 下的 `data/memory/`；当默认 root 下存在 legacy `data/user/workspace/memory/` 中的 `.md` 文件时，SHALL 单向复制（不覆盖已存在文件）到新位置。

#### Scenario: 打开既有 Python 版数据目录
- **WHEN** Go 版指向一个由 Python v1.5.2 创建的 `data/` 目录启动
- **THEN** 所有路径解析（settings、chat_history.db、knowledge_bases、workspace feature 目录）与 Python 版一致，无任何目录迁移或重命名发生

#### Scenario: legacy memory 文件迁移
- **WHEN** `data/user/workspace/memory/notes.md` 存在且 `data/memory/notes.md` 不存在
- **THEN** 解析 memory 目录时 `notes.md` 被复制到 `data/memory/`，源文件保留

### Requirement: 公共产物路径判定
系统 SHALL 提供 `is_public_output_path` 等价判定，决定一个文件是否可经 `/api/outputs/` 对外暴露。判定 SHALL 满足：路径必须落在 `data/user/` 之内且为存在的文件；后缀在私有集合 `{.json,.sqlite,.db,.md,.yaml,.yml,.py,.log}` 内的一律拒绝；仅以下相对子树放行——`workspace/co-writer/audio/`、`workspace/chat/deep_solve/<task>/**/artifacts/**`、`workspace/chat/math_animator/<task>/**/artifacts/**`、`workspace/chat/*/<task>/**/code_runs/**`、`workspace/chat/*/<task>/**/media/**`、`workspace/chat/chat/<task>/exec/**`、`workspace/chat/_detached_code_execution/**`。其余路径 SHALL 拒绝。

#### Scenario: 私有后缀被拒绝
- **WHEN** 判定 `workspace/chat/math_animator/t1/artifacts/meta.json`
- **THEN** 返回 false（即使位于 artifacts 白名单子树）

#### Scenario: Manim 视频产物放行
- **WHEN** 判定 `workspace/chat/math_animator/turn_x/artifacts/anim.mp4` 且文件存在
- **THEN** 返回 true

### Requirement: settings 路径 API
系统 SHALL 提供 `get_settings_file(name)` 语义：`name` 无扩展名时补 `.json`；并提供 `get_runtime_config_file(name)` 语义：非 `.yaml` 结尾时补 `.yaml`（用于 `main.yaml`、`agents.yaml` 等运行时 YAML 配置）。

#### Scenario: 补全扩展名
- **WHEN** 请求 settings 文件 `system` 与 runtime config `main`
- **THEN** 分别解析为 `data/user/settings/system.json` 与 `data/user/settings/main.yaml`
