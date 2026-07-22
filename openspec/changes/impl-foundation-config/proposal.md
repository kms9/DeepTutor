# Proposal: impl-foundation-config

## Why

deeptutor-go 需要一个与 Python 基线（`deeptutor/services/config/runtime_settings.py`、`deeptutor/services/path_service.py`）行为对等的运行时配置层与路径服务，才能免迁移复用现有 `data/` 目录与 `data/user/settings/*.json`。本模块是 ROADMAP 中 Wave 0（基础契约）/ M0 里程碑的成员，行为规格见 `docs/golang-req/openspec/specs/foundation-config/spec.md`，是 runtime-registry 及所有落盘模块的公共前置依赖。

## What Changes

- 实现 settings 文件的 load-or-create 语义：按名解析 `data/user/settings/<name>.json`、默认值合并、归一化后原子写回、损坏 JSON 容错回落默认值。
- 实现 `system.json` 字段级归一化与钳制：端口范围校验、truthy/falsy 布尔解析、附件限额钳制与 `chat_attachment_max_total_mb` 抬升、`cors_origins` 归一化。
- 实现进程环境变量 override（仅内存生效、不落盘）与 self-echo 抑制（`export_environment` 自导出的值不构成 override），并有意忽略项目根 `.env` 文件；支持 `DEEPTUTOR_IGNORE_PROCESS_ENV_OVERRIDES` 全局禁用。
- 实现 `document_parsing.json` v2 多引擎结构与 legacy 迁移（`mineru.json` 重命名、v1 扁平结构迁入 `engines.mineru`）。
- 实现 `render_environment` 等价能力：把 system/auth/integrations 渲染为子进程环境变量表（含 `DEEPTUTOR_API_BASE_URL` 解析顺序）。
- 实现 `PathService`：基线 `data/` 目录布局逐一对等、`ensure_all_directories` 幂等创建、legacy memory 单向复制、多用户 `data/users/<uid>/` 实例化。
- 实现 `is_public_output_path` 公共产物路径判定（私有后缀黑名单 + 相对子树白名单）。
- 实现 `get_settings_file` / `get_runtime_config_file` 路径 API（扩展名补全）。

## Capabilities

### New Capabilities

- `foundation-config`: deeptutor-go 的运行时配置层与路径服务——settings JSON 读写归一化、process-env override（含 self-echo 抑制）、`data/` 目录布局与公共产物路径判定。

### Modified Capabilities

（无）

## Impact

- 新建代码：`deeptutor-go/internal/config/`（settings 加载/归一化/env override/原子写、`PathService`、公共产物路径判定）。
- 技术选型：`spf13/viper` 读取 `data/user/settings/*.json`（每文件一个实例），env override 与 self-echo 抑制按 `foundation-config` spec 实现。
- 依赖的其他 change：无（按 ROADMAP 依赖图，Wave 0 模块相互无依赖）。
- 被依赖：`impl-runtime-registry`（cfg → reg）以及所有需要 settings/路径的下游模块（session-store、sidecar-contract、llm-provider 等在运行期使用本模块产物）。
- 不修改任何现有 Python 基线代码与 `web/` 前端；不改变 `data/` 磁盘格式。
